# RPC and Peer Transaction Lifecycle

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

### Introduction

Transactions enter the XRP Ledger network through two primary channels: direct RPC submission from clients and relay from peer nodes. Understanding how transactions flow through these paths is essential for debugging submission issues, optimizing transaction throughput, and implementing client applications.

This chapter traces the complete journey of a transaction from submission to inclusion in a validated ledger.

### Transaction Entry Points

```
┌─────────────────────────────────────────────────────────────┐
│              TRANSACTION ENTRY POINTS                       │
│                                                             │
│   ┌─────────────────┐          ┌─────────────────┐          │
│   │   RPC Client    │          │   Peer Node     │          │
│   │   (WebSocket,   │          │   (rippled)     │          │
│   │    JSON-RPC)    │          │                 │          │
│   └────────┬────────┘          └────────┬────────┘          │
│            │                            │                   │
│            ▼                            ▼                   │
│   ┌─────────────────┐          ┌─────────────────┐          │
│   │    doSubmit     │          │ PeerImp::       │          │
│   │                 │          │ onMessage       │          │
│   └────────┬────────┘          └────────┬────────┘          │
│            │                            │                   │
│            └──────────┬─────────────────┘                   │
│                       │                                     │
│                       ▼                                     │
│            ┌─────────────────────┐                          │
│            │  NetworkOPs::       │                          │
│            │  processTransaction │                          │
│            └─────────────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

### Path 1: RPC Submission

**Client submits via submit RPC:**

```
┌─────────────────────────────────────────────────────────────┐
│                  RPC SUBMISSION PATH                        │
│                                                             │
│   Step 1: RPC Handler                                       │
│   ───────────────────                                       │
│   Client Request:                                           │
│   {                                                         │
│       "method": "submit",                                   │
│       "params": [{                                          │
│           "tx_blob": "1200002280000000..."                  │
│       }]                                                    │
│   }                                                         │
│           │                                                 │
│           ▼                                                 │
│   doSubmit()                                                │
│       │                                                     │
│       ├── Deserialize transaction blob                      │
│       ├── Basic format validation                           │
│       └── Forward to NetworkOPs                             │
│                                                             │
│   Step 2: NetworkOPs Processing                             │
│   ─────────────────────────────                             │
│   NetworkOPs::processTransaction()                          │
│       │                                                     │
│       └── Queue for batch processing                        │
│                                                             │
│   Step 3: Batch Processing                                  │
│   ───────────────────────                                   │
│   NetworkOPs::transactionBatch()                            │
│       │                                                     │
│       └── Process queued transactions                       │
└─────────────────────────────────────────────────────────────┘
```

### Path 2: Peer Relay

**Transaction received from peer:**

```
┌─────────────────────────────────────────────────────────────┐
│                   PEER RELAY PATH                           │
│                                                             │
│   Step 1: Peer Message Handler                              │
│   ────────────────────────────                              │
│   PeerImp::onMessage(TMTransaction)                         │
│       │                                                     │
│       ▼                                                     │
│   PeerImp::handleTransaction()                              │
│       │                                                     │
│       ├── Check if we've seen this transaction              │
│       ├── Verify basic structure                            │
│       └── Forward if new                                    │
│                                                             │
│   Step 2: Transaction Checking                              │
│   ────────────────────────────                              │
│   PeerImp::checkTransaction()                               │
│       │                                                     │
│       ├── Validate signature (if required)                  │
│       ├── Check transaction flags                           │
│       └── Determine if transaction is valid                 │
│                                                             │
│   Step 3: NetworkOPs Processing                             │
│   ─────────────────────────────                             │
│   NetworkOPs::processTransaction()                          │
│       │                                                     │
│       └── Continue to common processing path                │
└─────────────────────────────────────────────────────────────┘
```

### Common Processing Path

**After entry (RPC or Peer):**

```
┌─────────────────────────────────────────────────────────────┐
│             COMMON PROCESSING PATH                          │
│                                                             │
│   NetworkOPs::processTransaction()                          │
│       │                                                     │
│       ▼                                                     │
│   doTransactionSync()                                       │
│       │                                                     │
│       ▼                                                     │
│   doTransactionSyncBatch()                                  │
│       │                                                     │
│       ▼                                                     │
│   NetworkOPs::apply()                                       │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │            TRANSACTION QUEUE                        │   │
│   │                                                     │   │
│   │   app_.getTxQ().apply()                             │   │
│   │       │                                             │   │
│   │       ├── Check fee requirements                    │   │
│   │       ├── Check account sequence                    │   │
│   │       ├── Queue or apply directly                   │   │
│   │       └── Return result                             │   │
│   └─────────────────────────────────────────────────────┘   │
│       │                                                     │
│       ▼                                                     │
│   tryDirectApply()                                          │
│       │                                                     │
│       └── Attempt immediate application                     │
└─────────────────────────────────────────────────────────────┘
```

### Transaction Application

**The apply process (preflight → preclaim → doApply):**

```
┌─────────────────────────────────────────────────────────────┐
│              TRANSACTION APPLICATION                        │
│                                                             │
│   ripple::apply(tx, view, flags)                            │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  PREFLIGHT                                          │   │
│   │  ──────────                                         │   │
│   │  • Validate transaction format                      │   │
│   │  • Check required fields present                    │   │
│   │  • Verify signature                                 │   │
│   │  • Check fee structure                              │   │
│   │  • Stateless validation only                        │   │
│   │                                                     │   │
│   │  Result: tesSUCCESS or tem* error                   │   │
│   └─────────────────────────────────────────────────────┘   │
│       │                                                     │
│       ▼ (if preflight passes)                               │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  PRECLAIM                                           │   │
│   │  ────────                                           │   │
│   │  • Check account exists                             │   │
│   │  • Verify account sequence                          │   │
│   │  • Check sufficient XRP for fee + reserve           │   │
│   │  • Validate against current ledger state            │   │
│   │  • Stateful but non-modifying                       │   │
│   │                                                     │   │
│   │  Result: tesSUCCESS or tec*/ter* error              │   │
│   └─────────────────────────────────────────────────────┘   │
│       │                                                     │
│       ▼ (if preclaim passes)                                │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  DO APPLY                                           │   │
│   │  ────────                                           │   │
│   │  • Execute transaction logic                        │   │
│   │  • Modify ledger state                              │   │
│   │  • Generate metadata                                │   │
│   │  • Consume fee                                      │   │
│   │                                                     │   │
│   │  Result: tesSUCCESS or tec* error                   │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Transaction Queue (TxQ)

**Queue management:**

```
┌─────────────────────────────────────────────────────────────┐
│                 TRANSACTION QUEUE                           │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    TxQ State                        │   │
│   │                                                     │   │
│   │   byFee:     Priority queue (fee-ordered)           │   │
│   │   byAccount: Map of account → queued txns           │   │
│   │   held:      Transactions waiting on sequence       │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   Incoming Transaction                                      │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────────────┐                                       │
│   │ Fee sufficient? │                                       │
│   └────────┬────────┘                                       │
│            │                                                │
│        YES │  NO                                            │
│            │   └──► Reject (telINSUF_FEE_P)                 │
│            ▼                                                │
│   ┌─────────────────┐                                       │
│   │ Sequence valid? │                                       │
│   └────────┬────────┘                                       │
│            │                                                │
│    CURRENT │  FUTURE                                        │
│            │   └──► Queue in held (wait for prior)          │
│            ▼                                                │
│   ┌─────────────────┐                                       │
│   │  Apply to open  │                                       │
│   │     ledger      │                                       │
│   └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

### Network Relay

**After successful application:**

```
┌─────────────────────────────────────────────────────────────┐
│                   NETWORK RELAY                             │
│                                                             │
│   app_.overlay().relay(tx)                                  │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              Overlay Network                        │   │
│   │                                                     │   │
│   │        ┌──────────────────────────┐                 │   │
│   │        │      This Node           │                 │   │
│   │        │    (received tx)         │                 │   │
│   │        └──────────┬───────────────┘                 │   │
│   │                   │                                 │   │
│   │         ┌─────────┼─────────┐                       │   │
│   │         ▼         ▼         ▼                       │   │
│   │     [Peer 1]  [Peer 2]  [Peer 3]                    │   │
│   │         │         │         │                       │   │
│   │         ▼         ▼         ▼                       │   │
│   │       (relay)   (relay)   (relay)                   │   │
│   │         │         │         │                       │   │
│   │         ▼         ▼         ▼                       │   │
│   │     [More]    [More]    [More]                      │   │
│   │     [Peers]   [Peers]   [Peers]                     │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   Relay Logic:                                              │
│   • Send to connected peers                                 │
│   • Avoid sending back to source                            │
│   • Track relayed transactions (prevent loops)              │
│   • Respect peer capabilities                               │
└─────────────────────────────────────────────────────────────┘
```

### Complete Transaction Journey

```
┌─────────────────────────────────────────────────────────────┐
│           COMPLETE TRANSACTION JOURNEY                      │
│                                                             │
│   1. SUBMISSION                                             │
│      └── RPC client or peer sends transaction               │
│                                                             │
│   2. ENTRY                                                  │
│      └── doSubmit() or PeerImp::onMessage()                 │
│                                                             │
│   3. VALIDATION                                             │
│      └── Format and signature checks                        │
│                                                             │
│   4. PROCESSING                                             │
│      └── NetworkOPs::processTransaction()                   │
│                                                             │
│   5. QUEUE                                                  │
│      └── TxQ evaluates fee and sequence                     │
│                                                             │
│   6. APPLICATION                                            │
│      └── preflight → preclaim → doApply                     │
│                                                             │
│   7. RELAY                                                  │
│      └── Broadcast to peer network                          │
│                                                             │
│   8. CONSENSUS                                              │
│      └── Included in candidate transaction set              │
│                                                             │
│   9. INCLUSION                                              │
│      └── Added to closed ledger                             │
│                                                             │
│   10. VALIDATION                                            │
│       └── Ledger validated by network                       │
│                                                             │
│   11. FINALITY                                              │
│       └── Transaction is final and irreversible             │
└─────────────────────────────────────────────────────────────┘
```

### Result Codes

**Transaction result categories:**

| Prefix | Category | Description |
| --- | --- | --- |
| tes | Success | Transaction succeeded |
| tec | Claim | Fee claimed but transaction failed |
| tef | Failure | Transaction failed, fee not claimed |
| tel | Local | Local error, not submitted |
| tem | Malformed | Transaction malformed |
| ter | Retry | Temporary failure, retry possible |

**Common Results:**

```
┌─────────────────────────────────────────────────────────────┐
│                  RESULT CODES                               │
│                                                             │
│   SUCCESS:                                                  │
│   tesSUCCESS         Transaction applied successfully       │
│                                                             │
│   CLAIM (fee consumed, tx failed):                          │
│   tecINSUFF_FEE      Insufficient fee for load              │
│   tecNO_DST          Destination doesn't exist              │
│   tecUNFUNDED        Insufficient XRP                       │
│                                                             │
│   FAILURE (fee not consumed):                               │
│   tefPAST_SEQ        Sequence already used                  │
│   tefMAX_LEDGER      LastLedgerSequence passed              │
│                                                             │
│   LOCAL (not submitted):                                    │
│   telINSUF_FEE_P     Fee below network minimum              │
│                                                             │
│   MALFORMED (invalid tx):                                   │
│   temINVALID         Invalid transaction format             │
│   temBAD_SIGNATURE   Invalid signature                      │
│                                                             │
│   RETRY (temporary):                                        │
│   terPRE_SEQ         Waiting for prior sequence             │
│   terQUEUED          Queued for later processing            │
└─────────────────────────────────────────────────────────────┘
```

### processClosedLedger

**When a ledger closes:**

```
┌─────────────────────────────────────────────────────────────┐
│             PROCESS CLOSED LEDGER                           │
│                                                             │
│   app_.getTxQ().processClosedLedger(ledger)                 │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  Queue Update Operations                            │   │
│   │                                                     │   │
│   │  1. Remove applied transactions                     │   │
│   │     └── Transactions in closed ledger               │   │
│   │                                                     │   │
│   │  2. Update fee metrics                              │   │
│   │     └── Recalculate based on ledger load            │   │
│   │                                                     │   │
│   │  3. Expire old transactions                         │   │
│   │     └── LastLedgerSequence passed                   │   │
│   │                                                     │   │
│   │  4. Retry held transactions                         │   │
│   │     └── Sequences now valid                         │   │
│   │                                                     │   │
│   │  5. Queue deferred transactions                     │   │
│   │     └── For next consensus round                    │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Open Ledger Accept

**Preparing for next round:**

```
┌─────────────────────────────────────────────────────────────┐
│              OPEN LEDGER ACCEPT                             │
│                                                             │
│   app_.openLedger().accept(...)                             │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  Open Ledger Preparation                            │   │
│   │                                                     │   │
│   │  1. Apply local transactions                        │   │
│   │     └── Not included in closed ledger               │   │
│   │                                                     │   │
│   │  2. Process retries                                 │   │
│   │     └── Transactions that can now succeed           │   │
│   │                                                     │   │
│   │  3. Update open ledger state                        │   │
│   │     └── Ready for new submissions                   │   │
│   │                                                     │   │
│   │  4. Reset for next round                            │   │
│   │     └── Clear temporary state                       │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Summary

**Entry Points:**

| Path | Entry Function | Source |
| --- | --- | --- |
| RPC | doSubmit() | Client applications |
| Peer | PeerImp::onMessage() | Network peers |

**Processing Stages:**

1. **Entry**: doSubmit / PeerImp::onMessage
2. **Validation**: checkTransaction
3. **Processing**: NetworkOPs::processTransaction
4. **Queue**: TxQ::apply
5. **Application**: preflight → preclaim → doApply
6. **Relay**: overlay().relay()
7. **Consensus**: Included in proposals
8. **Finality**: Validated ledger

**Key Functions:**

| Function | Purpose |
| --- | --- |
| preflight | Stateless validation |
| preclaim | Stateful validation |
| doApply | Execute and modify state |
| TxQ::apply | Fee and sequence management |
| relay | Network propagation |

**Design Principles:**

- Early rejection of invalid transactions
- Fee-based prioritization
- Sequence-ordered processing
- Efficient network propagation
- Clear result codes for debugging

**Source Code References:**

- [Submit.cpp](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/rpc/handlers/Submit.cpp) - RPC submit handler (doSubmit)
- [NetworkOPs.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/misc/NetworkOPs.h) - NetworkOPs interface
- [NetworkOPs.cpp](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/misc/NetworkOPs.cpp) - NetworkOPs implementation
- [PeerImp.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/overlay/detail/PeerImp.h) - Peer implementation header
- [PeerImp.cpp](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/overlay/detail/PeerImp.cpp) - Peer message handling
- [TxQ.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/misc/TxQ.h) - Transaction queue
- [applySteps.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/tx/applySteps.h) - Transaction apply phases
- [Overlay.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/overlay/Overlay.h) - Network overlay and relay
- [TER.h](https://github.com/XRPLF/rippled/blob/develop/include/xrpl/protocol/TER.h) - Transaction result codes

This completes our exploration of the consensus and ledger architecture module.
