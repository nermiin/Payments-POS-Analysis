# Worked Scenarios: Payments & POS

Each scenario walks a fictional transaction through PayRoute, so that rules can be checked against real numbers. Amounts are illustrative.

## S-01 Hotel Pre-Authorization and Partial Capture

| Step | Event | Amount (EUR) | State | Open hold |
|------|-------|--------------|-------|-----------|
| 1 | Authorize deposit at check-in | 200.00 | AUTHORIZED | 200.00 |
| 2 | Capture final bill | 183.50 | CAPTURED | 0.00 |
| 3 | Remainder released (BR-PAY-07) | 16.50 | CAPTURED | 0.00 |
| 4 | Clearing confirmed | 183.50 | SETTLED | 0.00 |

**Upper bound check:** with a 115% tolerance (FR-05), the highest valid capture on a EUR 200.00 authorization is EUR 230.00. A capture of EUR 230.01 returns `409 CAPTURE_EXCEEDS_LIMIT`.

## S-02 Refund Balance After Settlement

Starting point: EUR 183.50 captured and settled.

| Request | Amount (EUR) | Refunded so far | Remaining | Result |
|---------|--------------|-----------------|-----------|--------|
| R1 | 40.00 | 40.00 | 143.50 | PARTIALLY_REFUNDED |
| R2 | 100.00 | 140.00 | 43.50 | PARTIALLY_REFUNDED |
| R3 | 43.51 | 140.00 | 43.50 | Rejected, `REFUND_EXCEEDS_BALANCE` |
| R4 | 43.50 | 183.50 | 0.00 | REFUNDED |

## S-03 Timeout Then Retry

| Time | Actor | Event | Result |
|------|-------|-------|--------|
| 12:00:00 | Client | POST with key `k-77` for EUR 45.00 | Sent to acquirer |
| 12:00:25 | Gateway | No acquirer response | `202 PENDING_UNKNOWN`, reversal sent |
| 12:00:31 | Client | Retry with `k-77` | Same `202`, no new authorization |
| 12:00:40 | Acquirer | Reversal confirmed | State REVERSED, webhook sent |
| 12:00:52 | Client | New attempt with key `k-78` | New `paymentId`, approved |

**Outcome:** the customer is charged once. In the as-is flow the terminal shows a decline while the authorization may have succeeded, which is the root cause of PP-P01.

## S-04 Idempotency Conflict

| Attempt | Key | Amount (EUR) | Response |
|---------|-----|--------------|----------|
| 1 | `k-90` | 75.00 | `201`, payment created |
| 2 | `k-90` | 75.00 | `200`, original response |
| 3 | `k-90` | 57.00 | `422 IDEMPOTENCY_CONFLICT` |

## S-05 Offline Sale Declined at Sync

| Step | Detail |
|------|--------|
| Sale | EUR 38.00 at 18:40, terminal offline, within the EUR 50 floor limit |
| Queue | Customer receives a receipt, item held on terminal |
| Sync | 09:15 next day, within the 24h window |
| Issuer response | `51` insufficient funds |
| Result | DECLINED, merchant collection alert, loss sits with the merchant under BR-PAY-03 |

## S-06 Daily Merchant Settlement

| Line | Amount (EUR) |
|------|--------------|
| Captured sales, 412 transactions | 18,640.00 |
| Refunds | (365.00) |
| Chargebacks | (120.00) |
| Merchant service charge at 1.2% of sales | (223.68) |
| **Net payout** | **17,931.32** |

For a high-risk MCC on T+3 settlement (BR-MER-01), this payout arrives two business days later than the standard T+1 merchant, and appears in cash flow forecasts as a separate line.

## Edge Case Register

| ID | Situation | Expected Behavior | Rule |
|----|-----------|-------------------|------|
| E-01 | Void requested after capture | `409 INVALID_STATE`, guide user to refund | BR-PAY-01 |
| E-02 | Capture after authorization expiry | `409 INVALID_STATE`, new authorization needed | BR-PAY-06 |
| E-03 | Webhook endpoint down for 80h | Retries stop at 72h, event stays queryable, merchant alerted | NFR-03 |
| E-04 | Refund on day 181 | Rejected, `REFUND_WINDOW_CLOSED` | BR-PAY-05 |
| E-05 | Reversal itself times out | Retry reversal, raise ops case after 3 failures | BR-PAY-04 |
| E-06 | Clearing amount differs from capture | Payment flagged in auth versus capture mismatch report | BRD section 8 |
| E-07 | Offline queue reaches 24h with no signal | Merchant alert, items marked at-risk | BR-PAY-03 |
