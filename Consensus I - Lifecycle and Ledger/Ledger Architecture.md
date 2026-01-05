# Ledger Architecture

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

### Introduction

The XRP Ledger is a cryptographically-secured, immutable record of the complete state of the network at a specific point in time. Unlike traditional databases that modify records in place, the XRPL creates sequential snapshots where each ledger builds upon its predecessor.

This chapter explores the architectural principles that make the ledger both reliable and efficient, providing the foundation for understanding the implementation details that follow.

### The Ledger Chain Concept

**Sequential Progression:**

```
Genesis Ledger → Ledger 1 → Ledger 2 → ... → Current Ledger
     │               │           │                │
     └───────────────┴───────────┴────────────────┘
                   Cryptographic Links
```

**Properties:**

| Property | Description |
| --- | --- |
| **Cryptographic linking** | Each ledger references its predecessor via hash |
| **Monotonic sequence** | Ledger numbers always increase |
| **Temporal ordering** | Represents time progression in the network |
| **State evolution** | Each ledger is a state transition from the previous |

**Why Sequential Design?**

- Ensures deterministic ordering of events
- Prevents double-spending and race conditions
- Enables efficient synchronization between nodes
- Provides clear audit trail for all network activity

### Hierarchical Data Organization

**Three-Tier Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                     LEDGER STRUCTURE                        │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              TIER 1: LEDGER HEADER                  │   │
│   │                                                     │   │
│   │   • Sequence number                                 │   │
│   │   • Parent ledger hash                              │   │
│   │   • Transaction root hash                           │   │
│   │   • Account state root hash                         │   │
│   │   • Close time and flags                            │   │
│   │   • Total XRP in existence                          │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│            ┌──────────────┴──────────────┐                  │
│            ▼                             ▼                  │
│   ┌────────────────────┐    ┌────────────────────────┐      │
│   │ TIER 2: TX SET     │    │ TIER 3: STATE TREE     │      │
│   │                    │    │                        │      │
│   │ • All transactions │    │ • Account balances     │      │
│   │ • Canonical order  │    │ • Trust lines          │      │
│   │ • Metadata/results │    │ • Offers               │      │
│   │                    │    │ • Escrows, etc.        │      │
│   └────────────────────┘    └────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Merkle Tree Architecture

The state and transaction data are organized as Merkle trees, enabling efficient verification:

```
                        Root Hash
                       /         \
               Branch Hash    Branch Hash
                /      \        /      \
           Leaf Hash  Leaf Hash  ...   ...
               |          |
           Account    Account
            Data       Data
```

**Why Merkle Trees?**

| Benefit | Description |
| --- | --- |
| Compact verification | Verify specific data without downloading entire ledger |
| Tamper detection | Any change in data changes the root hash |
| Parallel processing | Different branches can be processed independently |
| Bandwidth optimization | Only transmit changed portions |

**Practical Application:**

A node can verify that a specific account balance is correct by checking only the path from that account to the root hash, rather than verifying the entire ledger.

### Data Integrity Mechanisms

**Multiple Layers of Protection:**

```
┌─────────────────────────────────────────────────────────────┐
│                 DATA INTEGRITY LAYERS                       │
│                                                             │
│   Layer 4: Chain Validation                                 │
│   ─────────────────────────                                 │
│   Each ledger references valid predecessor                  │
│                                                             │
│   Layer 3: Consensus Validation                             │
│   ─────────────────────────────                             │
│   Multiple validators must agree on content                 │
│                                                             │
│   Layer 2: Digital Signatures                               │
│   ─────────────────────────                                 │
│   Validator and transaction signatures                      │
│                                                             │
│   Layer 1: Cryptographic Hashing                            │
│   ────────────────────────────                              │
│   SHA-256 ensures data integrity                            │
└─────────────────────────────────────────────────────────────┘
```


#### Layer 1: Cryptographic Hashing

**Uses SHA-512Half to create unique fingerprints.** Each ledger contains two Merkle trees:
- **Transaction Map (`txMap_`)**: All transactions
- **State Map (`stateMap_`)**: All account states and objects

Hash calculation proceeds bottom-up: leaf nodes hash their data, inner nodes hash their children, producing a root hash.

The ledger header stores both root hashes and calculates the final ledger hash:
```cpp
header_.txHash = txMap_.getHash();
header_.accountHash = stateMap_.getHash();
header_.hash = calculateLedgerHash(header_);
```

**Key verification points:** When building a new ledger, hashes are computed FROM data (Ledger.cpp:327-332). When loading from database, the system verifies expected roots exist via `fetchRoot()` (Ledger.cpp:228-242). Network-acquired ledgers compare received vs expected hash (InboundLedger.cpp:811-819).

**Failures detected:** Missing nodes → network fetch. Hash mismatch → reject ledger. Internal corruption → abort via `invariants()` check.

---

#### Layer 2: Digital Signatures

**Proves authorization using ed25519/secp256k1 signatures.**

**Transaction signatures:** Every transaction must be signed by the account holder's private key. Invalid signatures result in `tefBAD_AUTH` rejection.

**Validator signatures:** Validators sign proposals during consensus (`RCLCxPeerPos`) and validations after building a ledger (`STValidation`). Each validation includes ledger hash, sequence, consensus hash, close time, and signature timestamp to prevent replay attacks.

**Key management:** Validators use a two-tier system—a master key (offline, long-term identity) and ephemeral keys (online, rotated regularly). The master key signs tokens authorizing ephemeral keys; each new token invalidates previous ones (see Exercise 1: Validator Keys Setup).

**Failures:** Invalid signatures → transaction/validation rejected. Revoked keys → all validations ignored. Unknown validators → not counted toward quorum.

---

#### Layer 3: Consensus Validation

**Provides Byzantine fault tolerance through validator quorum.**

A ledger requires validations from `floor(trusted_validators × 0.8) + 1` validators, tolerating up to 20% Byzantine failures.

**Process:** Collect validations for the ledger hash, filter out validators on the Negative UNL (unreliable nodes), check if quorum is met, and verify all validators agree on the same hash (LedgerMaster.cpp:932-942).

**Byzantine detection:** If a node builds a different ledger than the network majority, it enters `wrongLedger` mode (Consensus.h:1093), stops proposing, acquires the majority ledger from the network, and resumes consensus.

**Failures:** Insufficient validations → ledger remains unvalidated. Hash disagreement → enter `wrongLedger` mode. Network partition → wait for reconnection. The 80% quorum ensures agreement even with 20% compromised validators.

---

#### Layer 4: Chain Validation

**Ensures temporal consistency through parent-child relationships.**

Every ledger header references its predecessor (`parentHash`) and increments the sequence number (`seq = parent.seq + 1`). The system verifies: (1) parent exists, (2) parent hash matches, (3) sequence is continuous (LedgerMaster::checkAccept() line 922).

**Skip list:** Maintains references at exponential distances (parent, grandparent, great-grandparent) for efficient historical lookups via `Ledger::updateSkipList()`.

**Failures:** Parent hash mismatch → reject ledger. Missing parent → fetch from network. Sequence gap → fetch missing ledgers. Chain fork → `fixMismatch()` repairs (LedgerMaster.cpp:841).

**Protection:** Guarantees unbroken sequence from genesis; altering past ledgers breaks the chain and is immediately detectable.

---

#### How Layers Work Together

All four layers work sequentially during validation: **Layer 1** computes cryptographic hashes, **Layer 2** verifies signatures, **Layer 3** ensures validator agreement, and **Layer 4** confirms chain continuity. If ANY layer fails, the ledger is rejected. This defense-in-depth design means even if one layer is compromised (e.g., a stolen validator key), the other layers protect the system.

---

### The Ledger Lifecycle

**Four Phases of Ledger Creation:**

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  COLLECTION  │────►│   ASSEMBLY   │────►│  CONSENSUS   │────►│  VALIDATION  │
│              │     │              │     │              │     │              │
│ Gather       │     │ Apply txns   │     │ Validators   │     │ Seal and     │
│ pending      │     │ to previous  │     │ agree on     │     │ distribute   │
│ transactions │     │ ledger state │     │ canonical    │     │ to network   │
│              │     │              │     │ version      │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

**Phase Details:**

1. **Transaction Collection**
   - Gather pending transactions from network
   - Apply transaction queue ordering rules
   - Filter invalid or expired transactions

2. **Ledger Assembly**
   - Apply transactions to previous ledger state
   - Calculate new account balances and states
   - Generate transaction results and metadata

3. **Consensus Process**
   - Validators propose their assembled ledger
   - Network reaches agreement on canonical version
   - Disputed elements resolved through voting

4. **Validation and Finalization**
   - Final ledger is cryptographically sealed
   - Distributed to all network participants
   - Becomes immutable part of the chain

### State Transition Model

**How State Changes Work:**

```
                Current State + Transactions = New State

┌────────────────┐     ┌────────────────┐     ┌────────────────┐
│   Ledger N-1   │  +  │  Transaction   │  =  │   Ledger N     │
│                │     │     Set        │     │                │
│ Alice: 100 XRP │     │ Alice→Bob:     │     │ Alice: 90 XRP  │
│ Bob:   50 XRP  │     │ 10 XRP         │     │ Bob:   60 XRP  │
└────────────────┘     └────────────────┘     └────────────────┘
        │                                              │
        │              Ledger N-1 remains              │
        │                 UNCHANGED                    │
        └──────────────────────────────────────────────┘
```

**ACID Properties:**

| Property | Description |
| --- | --- |
| Atomicity | All changes in a ledger succeed or fail together |
| Consistency | State transitions follow strict rules |
| Isolation | Concurrent operations don't interfere |
| Durability | Committed changes are permanent |

### Transaction Ordering and Determinism

**Why Deterministic Ordering Matters:**

All nodes must process transactions in identical order to reach the same final state.

**Ordering Principles:**

```
┌─────────────────────────────────────────────────────────────┐
│              TRANSACTION ORDERING RULES                     │
│                                                             │
│   1. Canonical Sorting                                      │
│      └── Transactions sorted by cryptographic hash          │
│                                                             │
│   2. Fee Priority                                           │
│      └── Higher fee transactions processed first            │
│                                                             │
│   3. Account Sequence                                       │
│      └── Same-account transactions in sequence order        │
│                                                             │
│   4. Temporal Constraints                                   │
│      └── Respect time-based limitations                     │
└─────────────────────────────────────────────────────────────┘
```

### Multi-Tier Storage Architecture

**Storage Hierarchy:**

```
┌─────────────────────────────────────────────────────────────┐
│                    STORAGE TIERS                            │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  TIER 1: ACTIVE MEMORY (RAM)                        │   │
│   │  • Current ledger state                             │   │
│   │  • Recent transaction history                       │   │
│   │  • Frequently accessed account data                 │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  TIER 2: FAST STORAGE (SSD)                         │   │
│   │  • Recent ledger history (thousands of ledgers)     │   │
│   │  • Transaction indices and metadata                 │   │
│   │  • Account state snapshots                          │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│                           ▼                                 │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  TIER 3: ARCHIVE STORAGE (HDD/Cloud)                │   │
│   │  • Complete historical ledger data                  │   │
│   │  • Long-term transaction records                    │   │
│   │  • Compliance and audit trails                      │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Data Lifecycle Management

**Retention Policies:**

| Category | Retention | Access Speed | Contents |
| --- | --- | --- | --- |
| Hot Data | Current + ~256 ledgers | Immediate | Current state, recent txns |
| Warm Data | ~32,570 ledgers (~1 day) | Fast | Recent history |
| Cold Data | Configurable | Slower | Full historical data |

### Handling State Conflicts

**Conflict Scenarios:**

```
┌─────────────────────────────────────────────────────────────┐
│              CONFLICT RESOLUTION                            │
│                                                             │
│   Scenario 1: Insufficient Funds                            │
│   ─────────────────────────────                             │
│   Transaction requires more XRP than available              │
│   → Later transactions in ledger may fail                   │
│   → Account reserve requirements enforced                   │
│                                                             │
│   Scenario 2: Sequence Number Gaps                          │
│   ────────────────────────────────                          │
│   Transactions must be processed in order                   │
│   → Missing sequence numbers cause rejection                │
│   → Prevents replay attacks                                 │
│                                                             │
│   Scenario 3: Concurrent Modifications                      │
│   ───────────────────────────────────                       │
│   Multiple transactions affecting same objects              │
│   → Deterministic ordering resolves conflicts               │
│   → First transaction wins, others may fail                 │
└─────────────────────────────────────────────────────────────┘
```

### Ledger vs. Traditional Database

**Comparison:**

| Aspect | Traditional Database | XRP Ledger |
| --- | --- | --- |
| State Model | Modify in place | Immutable snapshots |
| History | May be lost | Always preserved |
| Verification | Trust the database | Cryptographic proof |
| Recovery | Restore from backup | Rebuild from chain |
| Consistency | Single authority | Distributed consensus |

### Immutability and Its Implications

**Once a ledger is validated:**

```
┌─────────────────────────────────────────────────────────────┐
│                IMMUTABILITY GUARANTEES                      │
│                                                             │
│   ✓ Data cannot be altered                                  │
│   ✓ Hash provides unique fingerprint                        │
│   ✓ Chain integrity is verifiable                           │
│   ✓ Historical queries return consistent results            │
│   ✓ Audit trail is complete and tamper-evident              │
└─────────────────────────────────────────────────────────────┘
```

**Code Enforcement:**

```cpp
class Ledger {
    bool mImmutable;

    void setImmutable() {
        mImmutable = true;
        stateMap_.setImmutable();
        txMap_.setImmutable();
    }
};
```

Only immutable ledgers can be stored in a `LedgerHolder` and published to the network.

### Summary

**Key Architectural Concepts:**

1. **Sequential Chain**: Ledgers form a cryptographically-linked chain
2. **Three-Tier Structure**: Header, transactions, and state tree
3. **Merkle Trees**: Enable efficient verification and synchronization
4. **State Transitions**: New ledgers created by applying transactions
5. **Multi-Layer Integrity**: Hashing, signatures, consensus, and chain validation
6. **Tiered Storage**: Hot, warm, and cold data with appropriate access speeds
7. **Immutability**: Validated ledgers cannot be modified

**Design Benefits:**

| Benefit | How Achieved |
| --- | --- |
| Auditability | Complete transaction history preserved |
| Consistency | All nodes have identical view |
| Recovery | Can rebuild state from any point |
| Trust | Cryptographic proof of all changes |
| Efficiency | Merkle trees minimize data transfer |

In the next chapter, we'll dive into the specific data structures that implement this architecture.
