# Ledger Data Structures

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

### Introduction

The XRP Ledger's reliability and performance depend on carefully designed data structures. This chapter explores the core classes that represent ledgers, manage their lifecycle, and provide efficient access to historical data.

Understanding these structures is essential for working with the codebase, debugging issues, and implementing new features.

### The Ledger Class

The `Ledger` class is the primary representation of a single ledger instance:

```cpp
// Ledger.h
class Ledger {
    // Metadata about the ledger
    LedgerInfo info_;

    // State tree (all account states)
    SHAMap stateMap_;

    // Transaction tree
    SHAMap txMap_;

    // Immutability flag
    bool mImmutable;

    // Protocol rules and amendments
    Rules rules_;
};
```

**Core Components:**

```
┌─────────────────────────────────────────────────────────────┐
│                     LEDGER CLASS                            │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                   LedgerInfo                        │   │
│   │   seq, hash, parentHash, accountHash, txHash,       │   │
│   │   closeTime, closeTimeResolution, closeFlags, ...   │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   ┌──────────────────────┐    ┌──────────────────────┐      │
│   │      stateMap_       │    │       txMap_         │      │
│   │                      │    │                      │      │
│   │   Account states     │    │   Transactions       │      │
│   │   Trust lines        │    │   + metadata         │      │
│   │   Offers             │    │                      │      │
│   │   Escrows            │    │                      │      │
│   │   ...                │    │                      │      │
│   └──────────────────────┘    └──────────────────────┘      │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                     Rules                           │   │
│   │   Active amendments, protocol version               │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Construction Methods

Ledgers can be created in several ways:

**1. Genesis Ledger:**

```cpp
Ledger::Ledger(
    create_genesis_t,
    Config const& config,
    std::vector<uint256> const& amendments,
    Family& family)
{
    // Initialize first ledger with:
    // - Sequence 1
    // - Initial XRP distribution
    // - Genesis account states
    // - Activated amendments
}
```

**2. From Previous Ledger:**

```cpp
Ledger::Ledger(
    Ledger const& prevLedger,
    NetClock::time_point closeTime)
{
    // Copy and evolve:
    // - Increment sequence
    // - Set parent hash to previous ledger's hash
    // - Apply new close time
    // - Inherit state (copy-on-write)
}
```

**3. From Serialized Data:**

```cpp
Ledger::Ledger(
    LedgerInfo const& info,
    Config const& config,
    Family& family)
{
    // Reconstruct from:
    // - Persisted header info
    // - Load SHAMaps from NodeStore
}
```

### LedgerInfo Structure

The header contains essential metadata:

```cpp
struct LedgerInfo {
    // Ledger identification
    LedgerIndex seq;          // Sequence number
    uint256 hash;             // This ledger's hash
    uint256 parentHash;       // Previous ledger's hash

    // Tree roots
    uint256 accountHash;      // State tree root
    uint256 txHash;           // Transaction tree root

    // Timing
    NetClock::time_point closeTime;
    NetClock::time_point parentCloseTime;
    NetClock::duration closeTimeResolution;
    int closeFlags;

    // Network state
    XRPAmount drops;          // Total XRP in existence
};
```

**Header Hashing:**

The ledger hash is computed from the serialized header:

```
Ledger Hash = SHA512Half(
    seq ||
    drops ||
    parentHash ||
    txHash ||
    accountHash ||
    parentCloseTime ||
    closeTime ||
    closeTimeResolution ||
    closeFlags
)
```

This creates a unique fingerprint for the entire ledger state.

### LedgerHolder

Thread-safe container for immutable ledger references:

```cpp
// LedgerHolder.h
class LedgerHolder {
    std::mutex mutex_;
    std::shared_ptr<Ledger const> ledger_;

public:
    // Set requires immutable ledger
    void set(std::shared_ptr<Ledger const> ledger) {
        assert(ledger->isImmutable());
        std::lock_guard lock(mutex_);
        ledger_ = std::move(ledger);
    }

    // Get returns shared pointer
    std::shared_ptr<Ledger const> get() const {
        std::lock_guard lock(mutex_);
        return ledger_;
    }

    bool empty() const {
        std::lock_guard lock(mutex_);
        return !ledger_;
    }
};
```

**Usage Pattern:**

```
┌─────────────────────────────────────────────────────────────┐
│                    LEDGER HOLDER                            │
│                                                             │
│   ┌──────────────┐                                          │
│   │   Thread 1   │──────┐                                   │
│   └──────────────┘      │                                   │
│                         ▼                                   │
│   ┌──────────────┐    ┌────────────────┐    ┌───────────┐   │
│   │   Thread 2   │───►│  LedgerHolder  │───►│  Ledger   │   │
│   └──────────────┘    │   (mutex)      │    │ (const)   │   │
│                         ▲               └───────────┘   │
│   ┌──────────────┐      │                                   │
│   │   Thread 3   │──────┘                                   │
│   └──────────────┘                                          │
│                                                             │
│   • Multiple readers can access concurrently                │
│   • Updates are serialized through mutex                    │
│   • Only immutable ledgers can be held                      │
└─────────────────────────────────────────────────────────────┘
```

### LedgerHistory

Manages the cache and retrieval of historical ledgers:

```cpp
// LedgerHistory.h
class LedgerHistory {
    // Cache: hash → ledger
    TaggedCache<uint256, Ledger const> m_ledgers_by_hash;

    // Index: sequence → hash
    std::map<LedgerIndex, uint256> mLedgersByIndex;

public:
    // Insert ledger into cache
    void insert(
        std::shared_ptr<Ledger const> const& ledger,
        bool validated);

    // Retrieve by sequence
    std::shared_ptr<Ledger const> getLedgerBySeq(
        LedgerIndex ledgerIndex);

    // Retrieve by hash
    std::shared_ptr<Ledger const> getLedgerByHash(
        LedgerHash const& ledgerHash);

    // Fix index mapping
    void fixIndex(
        LedgerIndex ledgerIndex,
        LedgerHash const& ledgerHash);
};
```

**Cache Organization:**

```
┌─────────────────────────────────────────────────────────────┐
│                   LEDGER HISTORY                            │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              m_ledgers_by_hash                      │   │
│   │         (TaggedCache<uint256, Ledger>)              │   │
│   │                                                     │   │
│   │   Hash A ────────► Ledger Object                    │   │
│   │   Hash B ────────► Ledger Object                    │   │
│   │   Hash C ────────► Ledger Object                    │   │
│   │   ...                                               │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              mLedgersByIndex                        │   │
│   │          (map<LedgerIndex, uint256>)                │   │
│   │                                                     │   │
│   │   Seq 100 ────────► Hash A                          │   │
│   │   Seq 101 ────────► Hash B                          │   │
│   │   Seq 102 ────────► Hash C                          │   │
│   │   ...                                               │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   Lookup by seq:  seq → hash → ledger                       │
│   Lookup by hash: hash → ledger                             │
└─────────────────────────────────────────────────────────────┘
```

### Immutability Enforcement

The system strictly enforces ledger immutability:

```cpp
class Ledger {
    bool mImmutable = false;

    void setImmutable() {
        // Mark ledger as immutable
        mImmutable = true;

        // Mark SHAMaps as immutable
        stateMap_.setImmutable();
        txMap_.setImmutable();

        // Calculate and cache hash
        info_.hash = calculateHash();
    }

    bool isImmutable() const {
        return mImmutable;
    }
};
```

**Immutability Rules:**

| Operation | Mutable Ledger | Immutable Ledger |
| --- | --- | --- |
| Modify state | Allowed | Forbidden |
| Add transactions | Allowed | Forbidden |
| Store in LedgerHolder | Forbidden | Allowed |
| Publish to network | Forbidden | Required |
| Cache in LedgerHistory | Forbidden | Allowed |

### SHAMap Integration

Ledgers use SHAMaps for state and transaction storage:

```
┌─────────────────────────────────────────────────────────────┐
│                  SHAMAP INTEGRATION                         │
│                                                             │
│   Ledger                                                    │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                                                     │   │
│   │   stateMap_ (SHAMap)                                │   │
│   │   ┌───────────────────────────────────────────┐     │   │
│   │   │              Root Hash                    │     │   │
│   │   │             /    |    \                   │     │   │
│   │   │        [Inner] [Inner] [Inner]            │     │   │
│   │   │           |       |       |               │     │   │
│   │   │        [Leaf]  [Leaf]  [Leaf]             │     │   │
│   │   │      AccountNode AccountNode ...          │     │   │
│   │   └───────────────────────────────────────────┘     │   │
│   │                                                     │   │
│   │   txMap_ (SHAMap)                                   │   │
│   │   ┌───────────────────────────────────────────┐     │   │
│   │   │              Root Hash                    │     │   │
│   │   │             /    |    \                   │     │   │
│   │   │        [Inner] [Inner] [Inner]            │     │   │
│   │   │           |       |       |               │     │   │
│   │   │        [Leaf]  [Leaf]  [Leaf]             │     │   │
│   │   │       TxNode  TxNode  TxNode              │     │   │
│   │   └───────────────────────────────────────────┘     │   │
│   │                                                     │   │
│   └─────────────────────────────────────────────────────┘   │
│                                                             │
│   accountHash = stateMap_.getHash()                         │
│   txHash = txMap_.getHash()                                 │
└─────────────────────────────────────────────────────────────┘
```

### The Rules Class

Manages protocol rules and amendments:

```cpp
class Rules {
    // Set of enabled amendments
    std::set<uint256> enabledAmendments_;

    // Precomputed feature flags
    bool featureEnabled_[MAX_FEATURES];

public:
    // Check if amendment is enabled
    bool enabled(uint256 const& amendment) const {
        return enabledAmendments_.count(amendment) > 0;
    }

    // Check precomputed feature
    bool enabled(Feature f) const {
        return featureEnabled_[static_cast<int>(f)];
    }
};
```

**Amendment Impact:**

Rules determine how transactions are processed:

```
┌─────────────────────────────────────────────────────────────┐
│                   RULES APPLICATION                         │
│                                                             │
│   Transaction Processing:                                   │
│                                                             │
│   if (ledger.rules().enabled(featureXYZ)) {                │
│       // Use new processing logic                           │
│   } else {                                                  │
│       // Use legacy processing logic                        │
│   }                                                         │
│                                                             │
│   This ensures consistent behavior across all nodes         │
│   regardless of when they joined the network.               │
└─────────────────────────────────────────────────────────────┘
```

### Key Operations

**Reading Account State:**

```cpp
std::shared_ptr<SLE const> Ledger::read(Keylet const& k) const {
    auto const& item = stateMap_.peekItem(k.key);
    if (!item)
        return nullptr;

    auto sle = std::make_shared<SLE>(
        SerialIter{item->data(), item->size()}, k.key);

    // Verify type matches keylet
    if (sle->getType() != k.type)
        return nullptr;

    return sle;
}
```

**Modifying State (Mutable Ledger Only):**

```cpp
void Ledger::rawInsert(std::shared_ptr<SLE> const& sle) {
    assert(!mImmutable);

    Serializer ss;
    sle->add(ss);

    auto item = std::make_shared<SHAMapItem>(
        sle->key(), std::move(ss.modData()));

    stateMap_.addItem(SHAMapNodeType::tnACCOUNT_STATE, std::move(item));
}
```

### Database Persistence

Ledgers are persisted through multiple mechanisms:

```
┌─────────────────────────────────────────────────────────────┐
│                  PERSISTENCE FLOW                           │
│                                                             │
│   Ledger Object                                             │
│       │                                                     │
│       ├──► LedgerInfo → SQL Database (ledger metadata)      │
│       │                                                     │
│       ├──► stateMap_ → NodeStore (state nodes)              │
│       │                                                     │
│       └──► txMap_ → NodeStore (transaction nodes)           │
│                                                             │
│   Retrieval:                                                │
│       │                                                     │
│       ├──► Load LedgerInfo from SQL                         │
│       │                                                     │
│       └──► Reconstruct SHAMaps from NodeStore               │
│            (nodes loaded on-demand)                         │
└─────────────────────────────────────────────────────────────┘
```

**Verification on Load:**

```cpp
// Before saving, verify integrity
assert(info_.accountHash == stateMap_.getHash());
assert(info_.txHash == txMap_.getHash());
```

### Summary

**Core Data Structures:**

| Structure | Purpose | Key Features |
| --- | --- | --- |
| Ledger | Single ledger instance | State + Tx maps, immutability |
| LedgerInfo | Header metadata | Hashing, identification |
| LedgerHolder | Thread-safe container | Mutex protection, const enforcement |
| LedgerHistory | Cache management | Hash/seq lookup, validation tracking |
| Rules | Protocol configuration | Amendment checking |

**Key Properties:**

1. **Immutability**: Validated ledgers cannot be modified
2. **Efficient Lookup**: Dual indexing by hash and sequence
3. **Thread Safety**: LedgerHolder provides safe concurrent access
4. **Persistence**: Multiple storage backends supported
5. **Integrity**: Hash verification at all levels

**Design Patterns:**

- Shared pointers for memory management
- Copy-on-write for efficient branching
- Const correctness for immutability
- Tagged caches for LRU eviction

In the next chapter, we'll explore how ledgers are acquired from the network and validated.
