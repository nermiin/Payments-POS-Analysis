# Data Requirements and Test Plan: Payments & POS

## 1. Canonical Payment Record

| Field | Type | Mandatory | Rule |
|-------|------|-----------|------|
| `paymentId` | UUID | Yes | Same identifier across POS, eCom, mobile (FR-07) |
| `channel` | Enum | Yes | POS, ECOM, MOBILE |
| `state` | Enum | Yes | See [state machine](08-to-be-process-design.md) |
| `amountAuthorized` | Decimal | Yes | Minor-unit precision of currency |
| `amountCaptured` | Decimal | No | Up to 115% of authorized, where the MCC allows |
| `amountRefunded` | Decimal | Yes | Default 0, never above `amountCaptured` |
| `currency` | ISO 4217 | Yes | Three-letter code |
| `merchantId` / `terminalId` | String | Yes / POS only | Acquirer-assigned MID and TID |
| `mcc` | String(4) | Yes | Drives capture tolerance, expiry, settlement delay |
| `idempotencyKey` | String | Yes | Stored with a request hash |
| `authCode` / `rrn` | String | When approved | Needed for reversal and dispute lookup |
| `panToken` | String | Yes | Full PAN never stored or logged in clear text (FR-08) |
| `scaOutcome` | Enum | eCom EEA | AUTHENTICATED, EXEMPT_LOW_VALUE, EXEMPT_TRA, OUT_OF_SCOPE |

## 2. ISO 8583 Reference

| MTI | Meaning | Gateway operation |
|-----|---------|-------------------|
| 0100 / 0110 | Authorization request and response | authorize |
| 0200 / 0210 | Financial request and response | single-message sale |
| 0400 / 0410 | Reversal request and response | timeout, void |
| 0800 / 0810 | Network management | echo test, sign-on |

| Data element | Content |
|--------------|---------|
| DE2 | Primary account number |
| DE3 | Processing code |
| DE4 | Transaction amount |
| DE11 | System trace audit number (STAN) |
| DE37 | Retrieval reference number (RRN) |
| DE38 | Authorization code |
| DE39 | Response code |
| DE41 / DE42 | Terminal ID / merchant ID |
| DE49 | Currency code |

## 3. Decline Mapping (FR-09)

Customer-safe wording deliberately hides lost, stolen and fraud reasons, so a fraudster learns nothing from the response.

| Code | Issuer meaning | Merchant reason | Customer message | Retry? |
|------|----------------|-----------------|------------------|--------|
| 00 | Approved | APPROVED | Payment successful | n/a |
| 05 | Do not honor | ISSUER_DECLINED | Please try another card | No |
| 14 | Invalid card number | INVALID_CARD | Please check your card details | After correction |
| 41 / 43 | Lost / stolen card | PICKUP_CARD | Please try another card | Never |
| 51 | Insufficient funds | INSUFFICIENT_FUNDS | Please try another card | Later |
| 54 | Expired card | EXPIRED_CARD | Please check your card details | After correction |
| 55 | Incorrect PIN | INCORRECT_PIN | Incorrect PIN, please try again | Yes, limited |
| 61 | Exceeds amount limit | LIMIT_EXCEEDED | Please try another card | Later |
| 91 | Issuer unavailable | ISSUER_UNAVAILABLE | Please try again shortly | Yes |
| 96 | System malfunction | SYSTEM_ERROR | Please try again shortly | Yes |

## 4. Webhook Catalog (FR-03)

| Event | Fires on | Key payload |
|-------|----------|-------------|
| `payment.authorized` | AUTHORIZED | paymentId, amount, authCode |
| `payment.captured` | CAPTURED | paymentId, amountCaptured |
| `payment.reversed` | REVERSED | paymentId, reason |
| `payment.refunded` | Any refund | paymentId, refundId, amount, balance |
| `payment.settled` | SETTLED | paymentId, settlementDate, batchId |
| `terminal.offline_aging` | Queue over 20h | terminalId, count, oldestAt |

Every event carries a unique `eventId`, so that merchants can ignore redelivery during the 72h retry period.

## 5. UAT Plan

| TC | Covers | Input | Expected |
|----|--------|-------|----------|
| TC-01 | BR-PAY-02 | Same key and body twice | One authorization, HTTP 200 on replay |
| TC-02 | BR-PAY-02 | Same key, different amount | HTTP 422 `IDEMPOTENCY_CONFLICT` |
| TC-03 | BR-PAY-04 | Acquirer stub silent for 30s | `PENDING_UNKNOWN`, reversal sent |
| TC-04 | BR-PAY-07 | Capture 183.50 of 200.00 | CAPTURED, 16.50 released |
| TC-05 | FR-05 | Capture 230.01 of 200.00 | 409, state unchanged |
| TC-06 | BR-PAY-05 | Refund after SETTLED | Accepted, balance reduced |
| TC-07 | BR-PAY-05 | Refund above balance | Rejected, balance unchanged |
| TC-08 | BR-PAY-01 | Void a CAPTURED payment | 409 `INVALID_STATE` |
| TC-09 | BR-PAY-03 | Offline sale EUR 50.01 | Declined at terminal |
| TC-10 | BR-PAY-03 | Offline sale declined at sync | Merchant alert raised |
| TC-11 | FR-09 | Response code 43 | Customer sees generic message only |
| TC-12 | FR-10 | EEA eCom EUR 120, no exemption | 3DS challenge presented |
| TC-13 | NFR-03 | Endpoint returns 500 | Retries with backoff for 72h, same `eventId` |
| TC-14 | NFR-01 | 500 TPS domestic load | Authorization p95 below 800ms |

**Exit criteria:** all Must scenarios pass, no open severity 1 or 2 defects, and a pilot with three merchants reconciles to the cent over five settlement days.
