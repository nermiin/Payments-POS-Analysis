# RTM — Payments & POS

| Objective | Business Rule | FR | User Story |
|-----------|---------------|-----|------------|
| No duplicate charges | BR-PAY-02 | FR-02 | US-PAY-01 |
| Unified platform | BR-PAY-01 | FR-07 | US-PAY-01 |
| Offline visibility | BR-PAY-03 | FR-06 | US-PAY-03 |
| Hotel use cases | BR-PAY-01 | FR-05 | US-PAY-02 |
| Merchant self-service | — | FR-04 | US-PAY-04 |
| Real-time order updates | — | FR-03 | US-PAY-05 |
| Timeout clarity | — | FR-01 | US-PAY-06 |
| Merchant onboarding | BR-MER-01 | — | (Epic MER-ONB) |

## Coverage Review

Requirements review found FR-08 untraced. It is closed below, together with the approved change requests.

| Objective | Rule or CR | FR | Test |
|-----------|------------|-----|------|
| PCI scope control | none | FR-08 | Compliance attestation review |
| Timeout clarity | BR-PAY-04 (CR-01) | FR-01 | TC-03 |
| Post-settlement refunds | BR-PAY-05 (CR-02) | FR-04 | TC-06, TC-07 |
| Hotel use cases | BR-PAY-06 (CR-03), BR-PAY-07 (CR-05) | FR-05 | TC-04, TC-05 |
| No duplicate charges | BR-PAY-02 amended (CR-04) | FR-02 | TC-01, TC-02 |
| Clear decline reasons | PP-P06 (CR-06) | FR-09 | TC-11 |
| PSD2 SCA | Regulatory gap (CR-07) | FR-10 | TC-12 |
