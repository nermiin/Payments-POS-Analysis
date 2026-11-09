# As-Is Process Analysis — POS Payment Flow

## Swimlane — Card-Present Sale

```mermaid
flowchart TB
    subgraph Merchant
        M1[Cashier enters amount] --> M2[Customer taps card]
    end

    subgraph Terminal
        M2 --> T1[Terminal calls acquirer — direct]
        T1 --> T2{Approved?}
        T2 -->|No| T3[Decline — generic message]
        T2 -->|Yes| T4[Print receipt]
    end

    subgraph eCom["eCom, separate stack"]
        E1[Web checkout] --> E2[Different gateway]
        E2 --> E3[Separate merchant portal]
    end

    subgraph Finance
        T4 --> F1[Terminal batch close 22:00]
        F1 --> F2[Manual CSV to accounting]
        E3 --> F3[Second CSV — mismatched MIDs]
    end
```

## Failure Scenarios (Current)

| Scenario | Current Behavior | Customer/Merchant Impact |
|----------|------------------|--------------------------|
| Auth timeout | Terminal shows decline; auth may succeed | Double attempt risk |
| Offline mode | Store-and-forward — no central visibility | Delayed settlement surprises |
| Partial capture | Not supported — full void/re-auth | Hotel/pre-auth use cases broken |
| Refund | Only via acquirer portal — 2–3 days | CS calls, poor NPS |

## To-Be Summary

Single gateway API with idempotency keys, unified merchant dashboard, and explicit auth/capture/refund state machine.

See [Pain Points](07-pain-points.md) for prioritized gaps.
