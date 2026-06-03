# Technical Spec: Subscription Plan Migration Engine(diff change)

**Feature:** Automated Plan Migration on Billing Cycle Boundaries  
**Team:** Platform Billing  
**Author:** (redacted for external use)  
**Status:** Draft — for review  
**Date:** 2026-05-29

---

<details>
  <summary>Click to expand this section</summary>
  
  This is the hidden content. You can write regular text here.
</details>

## 1. Background

```
test
```

ExampleDotCom's billing system currently supports a static plan model: tenants are assigned a plan at creation time and can only be manually upgraded or downgraded by a support agent via an internal admin tool. This creates two problems:

1. **Sales friction** — customers cannot change their plan without filing a support ticket.
2. **Proration complexity** — the current system bills a full cycle regardless of when a change occurs, leading to customer complaints and refund requests.

---

## 2. Goals

- Allow tenants to request an upgrade or downgrade to a different plan at any time via API.
- Changes take effect at the **end of the current billing cycle** (downgrade) or **immediately** (upgrade), consistent with industry norms.
- Generate an accurate proration credit or charge at the point of transition.
- Emit structured billing events so downstream systems (invoicing, analytics, notifications) can react.
- Maintain a full audit trail of all plan changes.

## 3. Non-Goals

- UI/frontend work — this spec covers the API and backend only. A separate UI spec will follow.
- Free-trial logic — existing trial-to-paid conversion flow is unchanged.
- Multi-currency support — deferred to a later phase.
- Enterprise contract overrides — custom billing agreements are out of scope.

---

## 4. Proposed Design

### 4.1 Overview

We introduce a `PlanChangeRequest` entity. When a tenant submits a change request, the system:

1. Validates the request (plan exists, tenant is eligible, no conflicting pending request).
2. Determines the **effective date**: immediately for upgrades, next billing cycle start for downgrades.
3. Calculates the **proration amount** and stores it as a pending credit/charge.
4. Schedules execution via a background job keyed to the effective date.
5. On execution, atomically updates the subscription record, settles the proration, and emits a `subscription.plan_changed` event.

### 4.2 Data Model

#### New table: `plan_change_requests`

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` | Primary key |
| `tenant_id` | `uuid` | FK → `tenants.id` |
| `from_plan_id` | `uuid` | FK → `plans.id` |
| `to_plan_id` | `uuid` | FK → `plans.id` |
| `change_type` | `enum('upgrade', 'downgrade')` | Computed at request time |
| `status` | `enum('pending', 'applied', 'cancelled')` | |
| `effective_at` | `timestamptz` | When the change should be applied |
| `proration_amount_cents` | `integer` | Positive = charge, negative = credit |
| `proration_currency` | `varchar(3)` | ISO 4217 |
| `requested_at` | `timestamptz` | |
| `applied_at` | `timestamptz` | Nullable |
| `cancelled_at` | `timestamptz` | Nullable |
| `requested_by` | `uuid` | FK → `users.id` — who triggered the request |

#### Modified table: `subscriptions`

Add column `pending_plan_change_id uuid REFERENCES plan_change_requests(id)` (nullable). Prevents more than one pending change at a time via application-level check (see §4.5).

### 4.3 Proration Calculation

For an **upgrade** taking effect immediately:

```
days_remaining = (cycle_end_date - today) / cycle_length_days
new_plan_daily_rate = new_plan_price / cycle_length_days
old_plan_daily_rate = old_plan_price / cycle_length_days
proration_charge = (new_plan_daily_rate - old_plan_daily_rate) * days_remaining
```

For a **downgrade** taking effect at the next cycle boundary, no proration charge is applied — the tenant simply pays the lower amount from the next invoice onwards.

All amounts are rounded to the nearest cent (half-up) in the tenant's billing currency.

### 4.4 API

#### `POST /v1/subscriptions/{subscriptionId}/plan-change-requests`

Request a plan change.

**Request body**
```json
{
  "toPlanId": "plan_pro_monthly"
}
```

**Response `202 Accepted`**
```json
{
  "id": "pcr_01hw...",
  "subscriptionId": "sub_01hv...",
  "fromPlanId": "plan_starter_monthly",
  "toPlanId": "plan_pro_monthly",
  "changeType": "upgrade",
  "status": "pending",
  "effectiveAt": "2026-05-29T14:32:00Z",
  "prorationAmountCents": 1450,
  "prorationCurrency": "USD"
}
```

**Error cases**

| HTTP status | Code | Reason |
|---|---|---|
| `404` | `subscription_not_found` | Subscription does not belong to tenant |
| `409` | `plan_change_already_pending` | A pending request already exists |
| `422` | `same_plan` | `toPlanId` matches current plan |
| `422` | `plan_not_available` | Target plan is inactive or not offered to this tenant's region/tier |

---

#### `GET /v1/subscriptions/{subscriptionId}/plan-change-requests`

List all plan change requests for a subscription (paginated, ordered by `requested_at` desc).

---

#### `DELETE /v1/subscriptions/{subscriptionId}/plan-change-requests/{id}`

Cancel a pending request. Only valid while `status = 'pending'`. Returns `204 No Content`.

---

### 4.5 Business Rules

1. **One pending change at a time.** A tenant may not submit a new request while one is already `pending`. They must cancel the existing request first.
2. **Upgrade = immediate.** If the new plan's monthly price is higher than the current plan, `effective_at` is set to `now()` and the proration charge is added to the next invoice.
3. **Downgrade = end of cycle.** If the new plan's price is lower, `effective_at` is set to the current `cycle_end_date` on the subscription. No proration charge is generated.
4. **Free plans.** Downgrading to a $0 plan is treated the same as any other downgrade — effective at end of cycle.
5. **Cancellation window.** A pending downgrade can be cancelled at any time before `effective_at`. A pending upgrade that has already been applied cannot be cancelled (since it took effect immediately on submission).
6. **Idempotency.** Clients should supply an `Idempotency-Key` header. The system returns the same `202` response for duplicate requests within a 24-hour window.

### 4.6 Background Job: `apply-pending-plan-changes`

- Runs every **5 minutes** via cron.
- Queries `plan_change_requests` where `status = 'pending' AND effective_at <= now()`.
- For each row, within a database transaction:
  1. Lock the row with `SELECT ... FOR UPDATE SKIP LOCKED` to prevent concurrent processing.
  2. Update `subscriptions.plan_id = to_plan_id`.
  3. Set `subscriptions.pending_plan_change_id = NULL`.
  4. Set `plan_change_requests.status = 'applied'`, `applied_at = now()`.
  5. Insert a `billing_line_items` row for the proration amount (if non-zero).
  6. Publish `subscription.plan_changed` event (see §4.7).
- On failure, the row remains `pending` and will be retried on the next cron tick. After 3 consecutive failures (tracked via `failure_count` column), the row is moved to `status = 'failed'` and an alert is fired to the on-call channel.

### 4.7 Events

#### `subscription.plan_changed`

Published to the `billing.subscription-events` topic.

```json
{
  "eventType": "subscription.plan_changed",
  "eventId": "evt_01hw...",
  "occurredAt": "2026-05-29T14:32:05Z",
  "data": {
    "subscriptionId": "sub_01hv...",
    "tenantId": "ten_01hu...",
    "fromPlanId": "plan_starter_monthly",
    "toPlanId": "plan_pro_monthly",
    "changeType": "upgrade",
    "prorationAmountCents": 1450,
    "prorationCurrency": "USD",
    "planChangeRequestId": "pcr_01hw..."
  }
}
```

Consumers:
- **Invoicing service** — picks up proration line items and includes them on the next invoice.
- **Notification service** — sends a confirmation email to the tenant's billing contact.
- **Analytics service** — records plan change events for churn/upgrade funnel reporting.

---

## 5. Migration Plan

1. Deploy the `plan_change_requests` table migration and the `pending_plan_change_id` column on `subscriptions`. Both are additive and non-breaking.
2. Deploy the updated `subscription-service` with the new endpoints (behind feature flag `plan-migration-engine`).
3. Deploy the background job as a new deployment unit (`billing-plan-change-worker`).
4. Enable the feature flag for internal test tenants, run smoke tests.
5. Roll out to 10% of tenants, monitor error rate and job lag.
6. Full rollout + remove feature flag.

---

## 6. Observability

- **Metric:** `billing.plan_change_requests.created` (counter, tagged by `change_type`)
- **Metric:** `billing.plan_change_requests.applied_lag_seconds` (histogram — time between `effective_at` and `applied_at`)
- **Metric:** `billing.plan_change_requests.failures` (counter)
- **Log:** Structured log entry at each state transition with `plan_change_request_id`, `tenant_id`, and `change_type`.
- **Alert:** PagerDuty alert if `failures` count exceeds 5 in a 10-minute window.

---

## 7. Security & Access Control

- The `POST`/`DELETE` endpoints require the `billing:write` scope.
- The `GET` endpoint requires `billing:read`.
- Tenants can only access their own subscription's change requests — enforced by filtering on `tenant_id` extracted from the authenticated JWT.
- The background job runs with a service account that has no inbound HTTP surface.

---

## 8. Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | Should we support **mid-cycle upgrades that carry a prorated credit** for the remaining old-plan days, or simply charge the new rate from day 0? Current proposal charges only the delta — confirm with Finance. | Billing PM | Open |
| 2 | What is the correct behaviour if a tenant's payment method is expired at the time an upgrade is processed? Block the upgrade, or apply the change and flag for collection? | Billing PM + Finance | Open |
| 3 | Do we need a **grace period** after a downgrade takes effect before access to premium features is revoked? Or is revocation immediate? | Product | Open |
| 4 | Should the `plan-change-worker` be a separate deployment or a cron job inside the existing `billing-service`? The separate deployment is easier to scale independently but adds ops overhead. | Engineering | Open |
| 5 | Is `Idempotency-Key` required or optional? Stripe makes it optional but strongly recommended — align on our standard. | API Platform | Open |

---

## 9. Acceptance Criteria

- [ ] A tenant can request an upgrade via API and the change takes effect within 1 minute.
- [ ] A tenant can request a downgrade and the change takes effect exactly at the billing cycle boundary (within the 5-minute cron window).
- [ ] Proration amounts match the formula in §4.3 to the cent for a set of reference test cases.
- [ ] A second pending request returns `409 plan_change_already_pending`.
- [ ] Cancelling a pending downgrade before `effective_at` leaves the subscription unchanged.
- [ ] `subscription.plan_changed` event is emitted for every successfully applied change.
- [ ] Failed jobs after 3 retries trigger a PagerDuty alert.
- [ ] All endpoints return correct HTTP status codes for error cases listed in §4.4.

---

*End of spec*
