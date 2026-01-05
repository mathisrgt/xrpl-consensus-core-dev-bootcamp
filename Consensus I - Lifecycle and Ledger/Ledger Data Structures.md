# Ledger Data Structures

[← Back to Consensus I - Lifecycle and Ledger](./)

***

### Introduction

The XRP Ledger's reliability and performance depend on carefully designed data structures. This chapter explores the core classes that represent ledgers, manage their lifecycle, and provide efficient access to historical data.

The XRPL ledger is the authoritative record of the network's state at a given point in time. It contains all account balances, offers, escrows, and other objects, as well as a record of all transactions included in that ledger.

Every server always has an open ledger. All received new transactions are applied to the open ledger. The open ledger can't close until consensus is reached on the previous ledger and either there is at least one transaction or the ledger's close time has been reached.

Understanding these structures is essential for:
- Working with the codebase effectively
- Debugging ledger-related issues
- Implementing new features that interact with ledger state
- Understanding how consensus and validation work

### The Ledger Class

The `Ledger` class is the primary representation of a single ledger instance. It manages both the state (account balances, offers, escrows, etc.) and transaction data for a specific ledger.

**Location:** [LedgerHeader.h](https://github.com/XRPLF/rippled/blob/develop/include/xrpl/protocol/LedgerHeader.h), [Ledger.cpp](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/ledger/Ledger.cpp)

```cpp
// Ledger.h (actual source code)
class Ledger final : public std::enable_shared_from_this<Ledger>,
                     public DigestAwareReadView,
                     public TxsRawView
{
private:
    // Ledger metadata (sequence, hashes, close time, etc.)
    LedgerHeader header_;

    // State tree (all account states, trust lines, offers, escrows, etc.)
    SHAMap mutable stateMap_;

    // Transaction tree (all transactions and their metadata)
    SHAMap mutable txMap_;

    // Immutability flag - once true, ledger cannot be modified
    bool mImmutable;

    // Protocol rules and enabled amendments for this ledger
    Rules rules_;

    // Fee schedule for this ledger
    Fees fees_;

    // ... additional private members ...
};
```

**Key Points:**

- **Immutability**: Once `mImmutable` is set to `true` via `setImmutable()`, the ledger cannot be modified. Only immutable ledgers can be stored in a `LedgerHolder` or published to the network.
- **SHAMaps**: Both `stateMap_` and `txMap_` are Merkle trees. The root hash of each tree is stored in `header_.accountHash` and `header_.txHash` respectively.
- **Rules**: The `rules_` object determines which amendments are active, affecting how transactions are processed.

**Core Components:**

```
┌─────────────────────────────────────────────────────────────┐
│                     LEDGER CLASS                            │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                 LedgerHeader                        │   │
│   │   seq, hash, parentHash, accountHash, txHash,       │   │
│   │   closeTime, closeTimeResolution, closeFlags, ...   │   │
│   │   drops (total XRP in existence)                    │   │
│   └─────────────────────────────────────────────────────┘   │
│                           │                                 │
│            ┌──────────────┴──────────────┐                  │
│            ▼                             ▼                  │
│   ┌────────────────────┐    ┌────────────────────────┐      │
│   │  stateMap_ (SHAMap)│    │   txMap_ (SHAMap)      │      │
│   │                    │    │                        │      │
│   │ • Account roots    │    │ • Transactions         │      │
│   │ • Trust lines      │    │ • Transaction metadata │      │
│   │ • Offers           │    │   (results, effects)   │      │
│   │ • Escrows          │    │                        │      │
│   │ • AMM, Payment Ch  │    │                        │      │
│   │ • ...all objects   │    │                        │      │
│   └────────────────────┘    └────────────────────────┘      │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                   Rules & Fees                      │   │
│   │   rules_: Active amendments and protocol rules      │   │
│   │   fees_: Base fee, reserve requirements             │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Construction Methods

Ledgers can be created in several ways, depending on the source of data:

**1. Genesis Ledger:**

```cpp
// Ledger.cpp - Constructor signature
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

Used to create the very first ledger in a new network.

**2. From Previous Ledger:**

```cpp
// Ledger.cpp - Constructor signature
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

This is the most common constructor used during consensus to build the next ledger.

**3. From Serialized Data:**

```cpp
// Ledger.cpp - Constructor signature
Ledger::Ledger(
    LedgerHeader const& info,
    Config const& config,
    Family& family)
{
    // Reconstruct from:
    // - Persisted header info
    // - Load SHAMaps from NodeStore
}
```

Used when loading historical ledgers from the database or acquiring them from peers.

### LedgerHeader Structure

The `LedgerHeader` struct contains all the metadata that uniquely identifies a ledger and its state. **This is the data that hashes to the ledger's hash.**

**Location:** [LedgerHeader.h](https://github.com/XRPLF/rippled/blob/develop/include/xrpl/protocol/LedgerHeader.h)

```cpp
// LedgerHeader.h (actual source code)
struct LedgerHeader
{
    //
    // For all ledgers
    //
    LedgerIndex seq = 0;                              // Sequence number
    NetClock::time_point parentCloseTime = {};         // When parent closed

    //
    // For closed ledgers
    //
    uint256 hash = beast::zero;                        // This ledger's hash
    uint256 txHash = beast::zero;                      // Transaction tree root
    uint256 accountHash = beast::zero;                 // State tree root
    uint256 parentHash = beast::zero;                  // Previous ledger's hash

    XRPAmount drops = beast::zero;                     // Total XRP in existence

    bool mutable validated = false;                    // Has been validated?
    bool accepted = false;                             // Has been accepted?

    int closeFlags = 0;                                // Flags for ledger close
    NetClock::duration closeTimeResolution = {};       // Close time resolution (2-120s)
    NetClock::time_point closeTime = {};               // When this ledger closed
};
```

**Key Header Fields:**

| Field | Purpose |
|-------|---------|
| `seq` | Ledger sequence number, incrementing from genesis |
| `parentHash` | Cryptographic link to previous ledger |
| `accountHash` | Root hash of state tree (from `stateMap_.getHash()`) |
| `txHash` | Root hash of transaction tree (from `txMap_.getHash()`) |
| `drops` | Total XRP in existence (decreases as fees are burned) |
| `closeTime` | When this ledger closed (consensus-agreed time) |
| `parentCloseTime` | Parent ledger's close time |
| `closeTimeResolution` | Granularity of close time (2-120 seconds) |
| `closeFlags` | Indicates if close time had consensus |
| `validated` | Set to true once ledger receives quorum validations |

**Header Hashing:**

The ledger hash uniquely identifies the ledger and is computed by serializing and hashing the header fields:

```cpp
// From calculateLedgerHash() - conceptual representation
Ledger Hash = SHA512Half(
    HashPrefix::ledgerMaster ||  // Prefix for ledger headers
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

This creates a unique, tamper-evident fingerprint for the entire ledger state. Any change to the state tree, transaction tree, or metadata produces a different hash.

**Note:** The "ledger base" refers to a query/response that includes the ledger header and may also contain the root node of the state tree. This is used during ledger acquisition from peers.

### LedgerHolder

A thread-safe container that holds an immutable ledger. **Only immutable ledgers can be held** - this is enforced at runtime.

**Location:** [LedgerHolder.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/ledger/LedgerHolder.h)

```cpp
// LedgerHolder.h (actual source code)
class LedgerHolder : public CountedObject<LedgerHolder>
{
public:
    // Update the held ledger (MUST be immutable!)
    void set(std::shared_ptr<Ledger const> ledger)
    {
        if (!ledger)
            LogicError("LedgerHolder::set with nullptr");
        if (!ledger->isImmutable())  // Runtime check!
            LogicError("LedgerHolder::set with mutable Ledger");

        std::lock_guard sl(m_lock);
        m_heldLedger = std::move(ledger);
    }

    // Return the (immutable) held ledger
    std::shared_ptr<Ledger const> get()
    {
        std::lock_guard sl(m_lock);
        return m_heldLedger;
    }

    // Check if a ledger is held
    bool empty()
    {
        std::lock_guard sl(m_lock);
        return m_heldLedger == nullptr;
    }

private:
    std::mutex m_lock;
    std::shared_ptr<Ledger const> m_heldLedger;
};
```

**Why immutability matters:**

- Multiple threads can safely read from an immutable ledger without locks
- Once set, the ledger's state never changes, preventing race conditions
- LedgerMaster uses LedgerHolders to track `mValidLedger`, `mClosedLedger`, etc.

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
│                       └────────────────┘    └───────────┘   │
│                         ▲                                   │
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

Manages the cache and retrieval of historical ledgers.

**Location:** [LedgerHistory.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/ledger/LedgerHistory.h), [LedgerHistory.cpp](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/ledger/LedgerHistory.cpp)

```cpp
// LedgerHistory.h (simplified from actual source)
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

**Key Operations:**

The `insert()` method (from LedgerHistory.cpp) adds ledgers to both caches and ensures the index-to-hash mapping is correct. The `fixIndex()` method is used to correct any inconsistencies in the index mapping that may occur during acquisition.

### Immutability Enforcement

**Once a ledger is finalized, it can never be changed.** This is enforced both at compile time (via `const` correctness) and runtime (via the `mImmutable` flag).

```cpp
void Ledger::setImmutable(bool rehash)
{
    // Force update of SHAMap root hashes
    if (!mImmutable && rehash)
    {
        // Get final root hashes from SHAMaps
        header_.txHash = txMap_.getHash().as_uint256();
        header_.accountHash = stateMap_.getHash().as_uint256();
    }

    // Calculate final ledger hash from header
    if (rehash)
        header_.hash = calculateLedgerHash(header_);

    // Lock down everything - no more modifications allowed
    mImmutable = true;
    txMap_.setImmutable();    // SHAMap becomes read-only
    stateMap_.setImmutable(); // SHAMap becomes read-only

    // Validate fee object exists (unless very early ledger)
    setup();
}
```

**Immutability Lifecycle:**

```
┌─────────────────────────────────────────────────────────────┐
│ LEDGER MUTABILITY LIFECYCLE                                 │
│                                                             │
│  1. Construction → MUTABLE                                  │
│     • Ledger(prevLedger, closeTime)                         │
│     • Can modify state, add transactions                    │
│     • NOT safe for concurrent access                        │
│                                                             │
│  2. Build & Consensus → STILL MUTABLE                       │
│     • Apply transactions to state                           │
│     • Flush dirty SHAMap nodes to database                  │
│     • Calculate final hashes                                │
│                                                             │
│  3. setImmutable() called → IMMUTABLE                       │
│     • Header hashes locked in                               │
│     • SHAMaps become read-only                              │
│     • Safe for concurrent reads                             │
│     • Can be stored in LedgerHolder                         │
│                                                             │
│  4. Validation → VALIDATED                                  │
│     • header_.validated = true                              │
│     • Received quorum of trusted validations                │
│     • Now part of canonical ledger history                  │
└─────────────────────────────────────────────────────────────┘
```

**Immutability Rules:**

| Operation | Mutable Ledger | Immutable Ledger |
| --- | --- | --- |
| Modify state (`rawInsert`, `rawReplace`) | ✅ Allowed | ❌ Forbidden |
| Add transactions | ✅ Allowed | ❌ Forbidden |
| Store in `LedgerHolder` | ❌ Runtime error | ✅ Required |
| Publish to network | ❌ Forbidden | ✅ Required |
| Cache in `LedgerHistory` | ❌ Not typical | ✅ Allowed |
| Read operations (`read`, `peek`) | ✅ Allowed | ✅ Allowed |

### SHAMap Integration

Ledgers use SHAMaps for state and transaction storage. The SHAMap is a Merkle-Patricia trie covered in detail in later modules. Here we focus on how the Ledger class interacts with it:

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

When `setImmutable(true)` is called, the final root hashes are retrieved from the SHAMaps and stored in the LedgerHeader.

### The Rules Class

The `Rules` class encapsulates which amendments (protocol upgrades) are enabled for a specific ledger. **All nodes must agree on which amendments are active for deterministic transaction processing.**

**Location:** [Rules.h](https://github.com/XRPLF/rippled/blob/develop/include/xrpl/protocol/Rules.h)

```cpp
// Rules.h (actual source code - uses pimpl idiom)
class Rules
{
private:
    class Impl;  // Implementation hidden in .cpp file

    // Shared pointer makes Rules cheap to copy
    std::shared_ptr<Impl const> impl_;

public:
    // Construct rules from a set of enabled amendment IDs
    explicit Rules(std::unordered_set<uint256, beast::uhash<>> const& presets);

    // Check if a specific amendment is enabled
    bool enabled(uint256 const& feature) const;

    // Rules are cheap to copy due to shared_ptr
    Rules(Rules const&) = default;
    Rules& operator=(Rules const&) = default;

    // Compare two rule sets
    bool operator==(Rules const&) const;
};
```

**How Rules Are Used:**

Throughout transaction processing, the code checks if specific amendments are enabled to determine which logic path to use:

```cpp
// Example from payment processing
if (view.rules().enabled(featureFlowCross))
{
    // Use new payment engine introduced by FlowCross amendment
    return flowCross(view, deliver, account, src, dst, ...);
}
else
{
    // Use legacy payment engine
    return legacyPaymentEngine(view, deliver, ...);
}
```

**Amendment Activation:**

```
┌─────────────────────────────────────────────────────────────┐
│           AMENDMENT LIFECYCLE PER LEDGER                    │
│                                                             │
│  1. Amendment Proposed                                      │
│     • Validators vote yes/no on amendments                  │
│     • Votes recorded in validation messages                 │
│                                                             │
│  2. Amendment Gains Majority (80%+ for 2 weeks)             │
│     • Flag ledger arrives (every 256 ledgers)               │
│     • Amendment marked as "enabled"                         │
│     • Rules object updated for next ledger                  │
│                                                             │
│  3. Rules Object Created for Each Ledger                    │
│     • makeRulesGivenLedger() reads enabled amendments       │
│     • Creates Rules object with that ledger's active set    │
│     • All transaction processing uses these rules           │
│                                                             │
│  4. Deterministic Processing                                │
│     • Same ledger + same rules = same result                │
│     • Historical replays work correctly                     │
│     • All validators reach identical state                  │
└─────────────────────────────────────────────────────────────┘
```

**Why This Matters:**

- **Consensus requires determinism**: All validators must process transactions identically
- **Amendments change behavior**: Different amendment sets → different results
- **Rules track the truth**: The `Rules` object for ledger N reflects exactly which amendments were active when ledger N was built

See **Amendments** in Consensus II for full details on the amendment system.

### Key Operations

The Ledger class provides methods to read and modify state objects. All operations work through the `stateMap_` SHAMap.

**Reading State:**

```cpp
// Ledger.cpp:414-431
std::shared_ptr<SLE const> Ledger::read(Keylet const& k) const
{
    // 1. Look up item in state map
    auto const& item = stateMap_.peekItem(k.key);
    if (!item)
        return nullptr;  // Object doesn't exist

    // 2. Deserialize to Serialized Ledger Entry (SLE)
    auto sle = std::make_shared<SLE>(SerialIter{item->slice()}, item->key());

    // 3. Verify type matches expectation
    if (!k.check(*sle))
        return nullptr;

    return sle;
}
```

The `read()` method uses a `Keylet` for type-safe lookups. A Keylet combines a key with a type checker, ensuring you retrieve the correct object type (e.g., `keylet::account(accountID)` for account roots).

**Modifying State (Mutable Ledgers Only):**

```cpp
// Insert new object
void rawInsert(std::shared_ptr<SLE> const& sle);

// Update existing object
void rawReplace(std::shared_ptr<SLE> const& sle);

// Delete object
void rawErase(std::shared_ptr<SLE> const& sle);
```

These methods:
- Only work on mutable ledgers (before `setImmutable()` is called)
- Serialize the SLE to binary format
- Add/update/remove items in `stateMap_`
- Throw `LogicError` if the operation fails (duplicate key, missing key, etc.)

### Database Persistence

Ledgers are stored using a two-tier system:

```
┌─────────────────────────────────────────────────────────────┐
│                  PERSISTENCE ARCHITECTURE                   │
│                                                             │
│   Ledger Object                                             │
│       │                                                     │
│       ├──► LedgerHeader → SQL Database                      │
│       │    • Sequence, hashes, close time                   │
│       │    • Fast header lookups by seq or hash             │
│       │                                                     │
│       └──► stateMap_ & txMap_ → NodeStore                   │
│            • Individual tree nodes stored by hash           │
│            • Shared across ledgers (deduplication)          │
│            • Lazy loading on access                         │
└─────────────────────────────────────────────────────────────┘
```

**Why Two Storage Systems?**

1. **SQL Database**: Stores ledger headers for fast sequential access and indexing
2. **NodeStore**: Stores SHAMap tree nodes for efficient content-addressable storage

When loading a ledger:
1. Retrieve header from SQL database by sequence or hash
2. Construct empty SHAMaps with root hashes from header
3. Load tree nodes from NodeStore on-demand as they're accessed

**Integrity Verification:**

Before persisting, the system verifies hash consistency:

```cpp
assert(header_.accountHash == stateMap_.getHash().as_uint256());
assert(header_.txHash == txMap_.getHash().as_uint256());
```

This ensures the header accurately represents the tree contents. Mismatches indicate data corruption, Byzantine attacks, or implementation bugs.

**Implementation:** See [Node.cpp](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/rdb/backend/detail/Node.cpp) and [SQLiteDatabase.cpp](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/rdb/backend/detail/SQLiteDatabase.cpp) for details.

### Summary

**Core Data Structures:**

| Structure | Purpose | Key Features |
| --- | --- | --- |
| Ledger | Single ledger instance | State + Tx maps, immutability |
| LedgerHeader | Header metadata | Hashing, identification |
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
- Pimpl idiom for Rules (hides implementation details)

**Source Code References:**

- [Ledger.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/ledger/Ledger.h) - Ledger class definition
- [Ledger.cpp](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/ledger/Ledger.cpp) - Ledger implementation
- [LedgerHeader.h](https://github.com/XRPLF/rippled/blob/develop/include/xrpl/protocol/LedgerHeader.h) - LedgerHeader structure
- [LedgerHolder.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/ledger/LedgerHolder.h) - Thread-safe holder
- [LedgerHistory.h](https://github.com/XRPLF/rippled/blob/develop/src/xrpld/app/ledger/LedgerHistory.h) - History cache
- [Rules.h](https://github.com/XRPLF/rippled/blob/develop/include/xrpl/protocol/Rules.h) - Amendment rules

In the next chapter, we'll explore how ledgers are acquired from the network and validated.
