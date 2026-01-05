# Ledger Acquisition and Validation

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

### Introduction

Nodes in the XRP Ledger network must maintain a consistent view of the ledger chain. This requires mechanisms to acquire missing ledgers from peers, validate their correctness, and integrate them into the local chain. This chapter covers the LedgerMaster orchestration layer, the InboundLedgers acquisition system, and the validation process.

### LedgerMaster Overview

The LedgerMaster is the central coordinator for ledger state management:

```
┌─────────────────────────────────────────────────────────────┐
│                     LEDGER MASTER                           │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                 Core State                          │   │
│   │                                                     │   │
│   │   mPubLedger      - Last published ledger           │   │
│   │   mValidLedger    - Last validated ledger           │   │
│   │   mLedgerHistory  - Historical ledger cache         │   │
│   │   mClosedLedger   - Last closed ledger              │   │
│   │   mCurrentLedger  - Working ledger (mutable)        │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                 Key Functions                       │   │
│   │                                                     │   │
│   │   doAdvance()       - Advance ledger state          │   │
│   │   fetchForHistory() - Acquire missing ledgers       │   │
│   │   checkAccept()     - Validate new ledger           │   │
│   │   tryAdvance()      - Continue state machine        │   │
│   │   storeLedger()     - Persist ledger                │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Ledger Types in LedgerMaster

```
┌─────────────────────────────────────────────────────────────┐
│                   LEDGER TYPES                              │
│                                                             │
│   Timeline:                                                 │
│                                                             │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────────┐    │
│   │Published│→ │Validated│→ │ Closed  │→ │  Current    │    │
│   │ Ledger  │  │ Ledger  │  │ Ledger  │  │  (Open)     │    │
│   └─────────┘  └─────────┘  └─────────┘  └─────────────┘    │
│       ↑            ↑            ↑             ↑             │
│   Streamed to  Network has   Consensus    Accepting         │
│   clients      validated     completed    new txns          │
│                                                             │
│   Published ≤ Validated ≤ Closed ≤ Current                  │
└─────────────────────────────────────────────────────────────┘
```

| Ledger Type | Description | Mutability |
| --- | --- | --- |
| Current | Open ledger accepting transactions | Mutable |
| Closed | Just closed, awaiting validation | Immutable |
| Validated | Network consensus achieved | Immutable |
| Published | Streamed to subscribers | Immutable |

### Gap Detection and Filling

When the node detects missing ledgers, it initiates acquisition:

```
┌─────────────────────────────────────────────────────────────┐
│                   GAP DETECTION                             │
│                                                             │
│   Local Chain:                                              │
│                                                             │
│   [Ledger 600] ──► [???] ──► [???] ──► [Ledger 603]         │
│                     601       602                           │
│                                                             │
│   Detection:                                                │
│   doAdvance() finds gap between 600 and 603                 │
│                                                             │
│   Filling Strategy:                                         │
│   1. Request ledger 602 first (work backwards)              │
│   2. When 602 complete, request 601                         │
│   3. Continue until gap closed                              │
│                                                             │
│   Why backwards?                                            │
│   - Can validate chain linkage immediately                  │
│   - 602's parentHash must equal 601's hash                  │
└─────────────────────────────────────────────────────────────┘
```

### InboundLedgers System

The InboundLedgers class manages ledger acquisition from peers:

```cpp
// InboundLedgers.h
class InboundLedgers {
public:
    // Start or continue acquisition
    std::shared_ptr<Ledger const> acquire(
        uint256 const& hash,
        std::uint32_t seq,
        InboundLedger::Reason reason);

    // Record acquisition failure
    void logFailure(uint256 const& hash, std::uint32_t seq);

    // Process incoming data from peer
    void gotData(
        std::weak_ptr<Peer> peer,
        uint256 const& hash,
        Blob const& data);
};
```

**Acquisition Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                  INBOUND LEDGERS                            │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │            InboundLedgersImp                        │   │
│   │                                                     │   │
│   │   mLedgers: map<hash, InboundLedger>               │   │
│   │   mRecentFailures: map<hash, failure_info>         │   │
│   │   mStash: pending data cache                        │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              InboundLedger                          │   │
│   │                                                     │   │
│   │   mHash: target ledger hash                         │   │
│   │   mSeq: target sequence number                      │   │
│   │   mLedger: assembled ledger (when complete)         │   │
│   │   mPeers: peers to request from                     │   │
│   │   mNeededStateNodes: missing state data             │   │
│   │   mNeededTxNodes: missing tx data                   │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Acquisition Process

**Step-by-Step Flow:**

```
┌─────────────────────────────────────────────────────────────┐
│              LEDGER ACQUISITION FLOW                        │
│                                                             │
│   Step 1: TRIGGERING                                        │
│   ──────────────────                                        │
│   LedgerMaster::doAdvance()                                 │
│       │                                                     │
│       ▼                                                     │
│   fetchForHistory(missing_seq)                              │
│       │                                                     │
│       ├── Try local storage first                           │
│       │                                                     │
│       └── If not found: InboundLedgers::acquire()           │
│                                                             │
│   Step 2: INITIALIZATION                                    │
│   ─────────────────────                                     │
│   InboundLedgers::acquire()                                 │
│       │                                                     │
│       ├── Check if already in progress                      │
│       │                                                     │
│       └── Create new InboundLedger if needed                │
│               │                                             │
│               ▼                                             │
│           InboundLedger::init()                             │
│               │                                             │
│               ├── Check local storage                       │
│               ├── Check fetch packs                         │
│               └── Request from peers                        │
│                                                             │
│   Step 3: DATA GATHERING                                    │
│   ─────────────────────                                     │
│   InboundLedger::gotData()                                  │
│       │                                                     │
│       ├── Queue incoming data                               │
│       │                                                     │
│       ▼                                                     │
│   processBatch()                                            │
│       │                                                     │
│       ├── Validate received nodes                           │
│       ├── Add to SHAMaps                                    │
│       └── Request any still-missing data                    │
│                                                             │
│   Step 4: COMPLETION                                        │
│   ────────────────                                          │
│   InboundLedger::done()                                     │
│       │                                                     │
│       ├── Mark ledger immutable                             │
│       ├── Store via LedgerMaster::storeLedger()             │
│       │                                                     │
│       ▼                                                     │
│   Schedule job:                                             │
│       ├── LedgerMaster::checkAccept()                       │
│       └── LedgerMaster::tryAdvance()                        │
└─────────────────────────────────────────────────────────────┘
```

### Validation Process

**Validation Quorum:**

```cpp
// Validation requirements
struct ValidationQuorum {
    std::size_t minVal;           // Minimum validations needed
    std::set<NodeID> trustedValidators;  // UNL validators
    std::set<NodeID> negativeUNL;        // Excluded validators
};
```

**checkAccept Flow:**

```
┌─────────────────────────────────────────────────────────────┐
│                CHECK ACCEPT FLOW                            │
│                                                             │
│   LedgerMaster::checkAccept(ledger)                         │
│       │                                                     │
│       ├── Can this ledger be current?                       │
│       │   └── Check sequence > last validated               │
│       │                                                     │
│       ├── Count trusted validations                         │
│       │   └── Filter by UNL, exclude negative UNL           │
│       │                                                     │
│       ├── validations >= minVal?                            │
│       │       │                                             │
│       │   NO  │  YES                                        │
│       │   ↓   ↓                                             │
│       │ Return  Continue                                    │
│       │                                                     │
│       ├── Mark ledger as validated                          │
│       │                                                     │
│       ├── Set as new validated ledger                       │
│       │                                                     │
│       ├── Handle fee voting                                 │
│       │                                                     │
│       ├── Check for amendment warnings                      │
│       │                                                     │
│       └── Call tryAdvance()                                 │
└─────────────────────────────────────────────────────────────┘
```

**Validation Checks:**

| Check | Purpose |
| --- | --- |
| Sequence check | Ensure forward progress |
| Trusted count | Verify UNL agreement |
| Minimum threshold | Require sufficient validations |
| Negative UNL | Exclude untrusted validators |

### tryAdvance State Machine

```
┌─────────────────────────────────────────────────────────────┐
│                   TRY ADVANCE                               │
│                                                             │
│   LedgerMaster::tryAdvance()                                │
│       │                                                     │
│       ├── Check for more ledgers to acquire                 │
│       │                                                     │
│       ├── Check for ledgers to publish                      │
│       │                                                     │
│       ├── Trigger gap filling if needed                     │
│       │                                                     │
│       └── Schedule next advancement check                   │
│                                                             │
│   State Transitions:                                        │
│                                                             │
│   [Gap Detected] → [Acquiring] → [Validating] → [Publishing]│
│         ↑              │              │              │      │
│         └──────────────┴──────────────┴──────────────┘      │
│                    (failures restart)                       │
└─────────────────────────────────────────────────────────────┘
```

### Ledger Storage

When a ledger is complete and validated:

```cpp
// LedgerMaster::storeLedger
void storeLedger(std::shared_ptr<Ledger const> ledger) {
    // 1. Verify integrity
    assert(ledger->info().accountHash == ledger->stateMap().getHash());
    assert(ledger->info().txHash == ledger->txMap().getHash());

    // 2. Store in history cache
    mLedgerHistory.insert(ledger, /*validated=*/true);

    // 3. Persist to database
    // Header → SQL
    // SHAMap nodes → NodeStore
}
```

### Publication Stream

Validated ledgers are published to clients:

```
┌─────────────────────────────────────────────────────────────┐
│                LEDGER PUBLICATION                           │
│                                                             │
│   LedgerMaster::findNewLedgersToPublish()                   │
│       │                                                     │
│       ▼                                                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              Publication Queue                      │   │
│   │                                                     │   │
│   │   Last Published: 100                               │   │
│   │   Last Validated: 105                               │   │
│   │                                                     │   │
│   │   Ready to publish: 101, 102, 103, 104, 105         │   │
│   └─────────────────────────────────────────────────────┘   │
│       │                                                     │
│       ▼                                                     │
│   For each ledger (in order):                               │
│       │                                                     │
│       ├── Check consecutive (no gaps)                       │
│       │                                                     │
│       ├── If gap found: acquire missing                     │
│       │                                                     │
│       └── Publish to subscribers                            │
│               │                                             │
│               ├── WebSocket clients                         │
│               ├── Streaming APIs                            │
│               └── Internal consumers                        │
└─────────────────────────────────────────────────────────────┘
```

### Error Handling and Recovery

**Acquisition Failures:**

```
┌─────────────────────────────────────────────────────────────┐
│              ERROR HANDLING                                 │
│                                                             │
│   Failure Type         │ Recovery Action                    │
│   ─────────────────────┼──────────────────────────────      │
│   Network timeout      │ Retry with different peers         │
│   Data corruption      │ Log, retry from scratch            │
│   Missing peers        │ Wait for peer connections          │
│   Invalid hash         │ Log failure, skip ledger           │
│   Database error       │ Retry persistence                  │
│                                                             │
│   Failure Tracking:                                         │
│                                                             │
│   InboundLedgers::logFailure(hash, seq)                     │
│       │                                                     │
│       ├── Record in mRecentFailures                         │
│       │                                                     │
│       ├── Implement backoff strategy                        │
│       │                                                     │
│       └── Eventually retry or abandon                       │
└─────────────────────────────────────────────────────────────┘
```

**Validation Mismatches:**

```cpp
// When built ledger doesn't match validated
if (builtHash != validatedHash) {
    JLOG(journal_.warn())
        << "Built ledger " << builtHash
        << " doesn't match validated " << validatedHash;

    // Trigger reacquisition
    acquireLedger(validatedHash, validatedSeq);
}
```

### Ledger Cleaning

The LedgerCleaner component maintains ledger integrity:

```
┌─────────────────────────────────────────────────────────────┐
│                 LEDGER CLEANER                              │
│                                                             │
│   Purpose:                                                  │
│   • Check for missing or inconsistent nodes                 │
│   • Fix transactions with missing metadata                  │
│   • Reacquire corrupted data                                │
│                                                             │
│   Operation:                                                │
│                                                             │
│   LedgerCleaner::doClean()                                  │
│       │                                                     │
│       ├── Check configured ledger range                     │
│       │                                                     │
│       ├── For each ledger:                                  │
│       │       │                                             │
│       │       ├── Verify state tree nodes                   │
│       │       │                                             │
│       │       ├── Verify transaction nodes                  │
│       │       │                                             │
│       │       └── Fix or reacquire if needed                │
│       │                                                     │
│       └── Track failures for retry                          │
│                                                             │
│   Constraints:                                              │
│   • Runs at low priority                                    │
│   • Avoids high server load periods                         │
│   • Configurable via RPC                                    │
└─────────────────────────────────────────────────────────────┘
```

### Summary

**Key Components:**

| Component | Role |
| --- | --- |
| LedgerMaster | Central coordinator |
| InboundLedgers | Network acquisition |
| InboundLedger | Single ledger assembly |
| LedgerHistory | Cache management |
| LedgerCleaner | Integrity maintenance |

**Acquisition Flow:**

1. **Detection**: doAdvance() finds missing ledgers
2. **Request**: InboundLedgers::acquire() starts acquisition
3. **Gathering**: InboundLedger collects data from peers
4. **Assembly**: SHAMaps built from received nodes
5. **Validation**: checkAccept() verifies sufficient validations
6. **Storage**: Persist to database and cache
7. **Publication**: Stream to subscribers

**Key Properties:**

- Backwards filling for chain validation
- Quorum-based acceptance
- Automatic retry on failure
- Continuous integrity checking
- Efficient caching and storage

In the next chapter, we'll explore how transactions flow through the system via RPC and peer connections.
