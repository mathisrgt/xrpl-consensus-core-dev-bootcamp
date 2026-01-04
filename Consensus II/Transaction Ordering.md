# Transaction Ordering

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

## Introduction

Transaction ordering is a critical component of XRPL's Byzantine Fault Tolerant consensus mechanism. It ensures all validators agree on the exact order of transactions within each ledger, maintaining consistency, fairness, and predictability across the network.

### What is Transaction Ordering?

**Definition:** Transaction ordering is the deterministic process by which XRPL arranges transactions within a ledger to ensure all validators process them in identical sequence.

**Core Function:**
- Ensures all validators agree on exact order of transactions
- Guarantees identical transaction ordering across all honest validators
- Prevents manipulation of transaction execution order
- Enables deterministic ledger state transitions

### Why Transaction Ordering Matters

**Consistency:**
- All nodes must process transactions in identical sequence
- Prevents divergent ledger states
- Ensures network-wide agreement

**Fairness:**
- Prevents manipulation of transaction execution order
- No user can game the system to get priority
- Equal treatment for all accounts

**Predictability:**
- Enables deterministic ledger state transitions
- Results are reproducible and verifiable
- Same inputs always produce same outputs

**Network Integrity:**
- Maintains consensus despite network delays
- Handles validator differences gracefully
- Prevents double-spending and other attacks

### Key Challenges Addressed

**Network Asynchrony:**
- Validators receive transactions at different times
- Network delays vary across global infrastructure
- Must achieve consistent ordering despite timing differences

**Validator Independence:**
- Each validator builds its own transaction set
- Different validators may see different transactions
- Must reach agreement on common set

**Dispute Resolution:**
- Handling disagreements about transaction inclusion
- Resolving conflicts about transaction ordering
- Achieving consensus despite differences

**Performance:**
- Balancing thoroughness with speed requirements
- Processing thousands of transactions per second
- Minimizing consensus round duration

***

## Canonical Transaction Ordering

### The CanonicalTXSet Class

**Location:** `rippled/src/xrpld/app/misc/CanonicalTXSet.h` and `rippled/src/xrpld/app/misc/CanonicalTXSet.cpp`

**Purpose:** Maintains a set of transactions in a deterministic, canonical order for processing in the XRPL ledger.

The `CanonicalTXSet` class is central to transaction ordering, ensuring all validators arrive at the same transaction sequence.

### Ordering Mechanism: Full Sort Key

Transactions are ordered using a composite key with the following precedence:

#### 1. Salted Account Key (uint256) - Primary Sort

**Purpose:** Creates unpredictable but deterministic ordering

**Mechanism:**
```cpp
uint256 CanonicalTXSet::accountKey(AccountID const& account)
{
    uint256 ret = beast::zero;
    memcpy(ret.begin(), account.begin(), account.size());
    ret ^= salt_;  // XOR with ledger-specific salt
    return ret;
}
```

**How It Works:**
- Account ID is XORed with a random salt value
- Salt is derived from ledger-specific data (e.g., parent ledger hash)
- Same salt used by all validators for deterministic results
- Different salt for each ledger prevents ordering predictability

**Why Salting Matters:**

**Anti-Gaming Measure:**
- Prevents users from predicting their position in queue
- Account addresses chosen for strategic advantage become useless
- Cannot create accounts to get consistent priority

**Fairness:**
- No account can consistently get priority
- Each ledger randomizes ordering fairly
- All accounts have equal opportunity

**Security:**
- Prevents strategic manipulation of transaction timing
- Makes it impossible to game the fee market
- Protects against certain attack vectors

#### 2. Sequence Proxy (SeqProxy) - Secondary Sort

**Purpose:** Orders transactions from the same account by their sequence number or ticket

**Mechanism:**
- Uses `SeqProxy` which handles both sequence numbers and tickets
- Ensures proper ordering within a single account
- Maintains account transaction chronology

**Why This Matters:**
- Account transactions must execute in order
- Prevents later transactions from executing before earlier ones
- Maintains account state consistency

#### 3. Transaction ID (uint256) - Tertiary Sort

**Purpose:** Final tie-breaker to ensure deterministic ordering

**Mechanism:**
- Uses transaction hash as last resort for ordering
- Guarantees unique, deterministic ordering even for identical sequence numbers
- Lexicographic comparison of transaction IDs

**When Used:**
- Identical salted account key and sequence proxy
- Ensures no ambiguity in final ordering
- Provides complete determinism

### Transaction Insertion

**Process:**

```cpp
void CanonicalTXSet::insert(std::shared_ptr<STTx const> const& txn)
{
    map_.insert(std::make_pair(
        Key(accountKey(txn->getAccountID(sfAccount)),
            txn->getSeqProxy(),
            txn->getTransactionID()),
        txn));
}
```

**Steps:**

1. Extract account ID from transaction
2. Calculate salted account key
3. Get sequence proxy (sequence or ticket)
4. Get transaction ID
5. Create composite key
6. Insert into ordered map

**Result:**
- Transaction placed in deterministic position
- Same transaction always placed in same position
- All validators arrive at identical ordering

### Set Updates and Modifications

**Replacement Logic:**
- If transaction with same account and sequence exists, new transaction may replace old
- Replacement only valid if it meets criteria (e.g., higher fee)
- Enforced by external logic, not CanonicalTXSet itself

**Removal Handling:**
- When transaction is applied to ledger, it's removed from set
- Next valid transaction for account is promoted
- Set maintains proper ordering after removals

**Salt Reset:**
- Set can be reset with new salt value
- Used when starting new consensus round
- Prevents ordering manipulation across rounds

### Benefits of Canonical Ordering

**Determinism:**
- Same inputs always produce same ordering
- Validators independently reach same result
- No coordination needed for ordering agreement

**Efficiency:**
- Reduces consensus rounds needed for agreement
- Validators propose similar transaction sets
- Fewer disputes to resolve

**Security:**
- Prevents strategic manipulation
- Fair treatment for all participants
- Resistant to various attack vectors

**Predictability:**
- Users can understand ordering logic
- Transparent and auditable process
- Reproducible results

***

## Transaction Set Construction and Proposal

### Initial Transaction Collection

**Sources of Transactions:**

**Network Submissions:**
- Transactions received from connected peers
- Relayed across the peer-to-peer network
- Arrive asynchronously from various sources

**Local Submissions:**
- Transactions submitted directly to this node
- From RPC clients or local applications
- Given same treatment as network transactions

**Peer Relays:**
- Transactions forwarded by other validators
- Help ensure transaction coverage across network
- Reduce transaction propagation delays

### Proposal Set Building: RCLConsensus::Adaptor::onClose

**Location:** `rippled/src/xrpld/app/consensus/RCLConsensus.cpp`

**Purpose:** Prepares the initial transaction set and proposal for the next consensus round.

**Process:**

1. **Gather Open Transactions:**
   - Collect transactions from open ledger
   - Pull from transaction queue (TxQ)
   - Consider fee levels and priorities

2. **Validation Phase:**
   - Basic format and signature verification
   - Check transaction well-formedness
   - Verify cryptographic signatures

3. **Preliminary Filtering:**
   - Remove obviously invalid transactions
   - Eliminate duplicates
   - Apply business logic checks

4. **Apply Canonical Ordering:**
   - Create CanonicalTXSet with current salt
   - Insert transactions in canonical order
   - Generate deterministic transaction sequence

5. **Capacity Management:**
   - Respect ledger size limits
   - Account for processing capacity
   - Ensure ledger can be built and validated in time

6. **Fee Prioritization:**
   - Higher fee transactions get preference
   - Within ordering constraints
   - Balance fairness with economic incentives

7. **Account Limits:**
   - Enforce per-account transaction limits per ledger
   - Prevent any account from monopolizing ledger space
   - Maintain fairness across accounts

8. **Finalize Proposal:**
   - Generate transaction set hash
   - Create proposal message
   - Sign with validator private key

### Proposal Broadcasting

**Peer Distribution:**
- Share proposed transaction set with other validators
- Use overlay network for efficient propagation
- Include transaction set hash in proposal message

**Compact Representation:**
- Full transaction data not sent in every proposal
- Transaction set hash used for compact reference
- Peers can request full set if needed

**Timing Coordination:**
- Align with consensus round timing requirements
- Proposals sent early in establish phase
- Allow time for dispute resolution

**Redundancy Handling:**
- Manage duplicate proposals from multiple sources
- Use suppression IDs to prevent relay loops
- Efficient handling of redundant data

***

## Consensus Process and Dispute Management

### DisputedTx Lifecycle

**Location:** `rippled/src/xrpld/consensus/DisputedTx.h`

Disputes arise when validators propose different transaction sets. The `DisputedTx` class tracks and resolves these differences.

#### Identification Phase: Creation

**When Disputes Are Created:**
- During proposal comparison between validators
- When a transaction appears in some proposals but not others
- Handled by `Consensus::createDisputes()`

**Process:**

1. Compare local transaction set with peer proposals
2. Identify transactions in local set but not peer's set (or vice versa)
3. Create `DisputedTx` object for each difference
4. Initialize voting tracking for dispute

**Data Tracked:**
- Transaction being disputed
- Votes "yes" (include) from each peer
- Votes "no" (exclude) from each peer
- Current vote counts
- Dispute creation time

#### Evaluation Phase: Voting

**Voting Mechanism:**

Each validator votes on whether to include or exclude the disputed transaction:

**setVote() Method:**
```cpp
void setVote(NodeID_t const& peer, bool votesYes)
{
    if (votesYes)
        ++yays_;
    else
        ++nays_;
    votes_[peer] = votesYes;
}
```

**Vote Updating:**
- As proposals arrive, votes are recorded
- Each peer can vote yes or no
- Votes can change during consensus round
- Vote changes tracked to detect stalls

**Avalanche Mechanism:**

The threshold for including a disputed transaction increases over time:

**updateVote() Method:**
- Calculates current consensus percentage
- Determines required threshold based on avalanche state
- Compares peer votes to threshold
- Updates local vote accordingly

**Avalanche States:**
- **init (0% time):** 50% agreement needed
- **mid (50% time):** 65% agreement needed
- **late (85% time):** 70% agreement needed
- **stuck (200% time):** 95% agreement needed

**Why Avalanche Works:**
- Rising threshold forces convergence
- Validators adjust votes to match majority
- Eventually all validators agree
- Provides deterministic resolution

#### Resolution Phase: Consensus

**Determining Inclusion:**

A transaction is included in the final set if:
1. It meets the avalanche threshold for the current time
2. Sufficient validators vote "yes"
3. No persistent disagreement exists

**Stalled Disputes:**

**stalled() Method:**
- Determines if consensus on transaction has stalled
- Occurs when votes are unequivocally above/below threshold
- Either over 80% "yes" or under 20% "yes"
- Prevents manipulation by minority

**When Disputes Are Resolved:**
- When consensus is reached on final transaction set
- When avalanche mechanism forces agreement
- When validators update positions to match majority

#### Cleanup Phase

**Dispute Removal:**
- Resolved disputes removed from tracking
- Memory freed for new disputes
- Only active disputes maintained

**Final Transaction Set:**
- Built from agreed-upon transactions
- All validators use identical set
- Applied to ledger deterministically

### Dispute Detection Mechanisms

**Proposal Comparison:**
- Identify transactions in some but not all proposals
- Compare transaction set hashes
- Request missing transactions if needed

**Threshold Analysis:**
- Determine if sufficient validator support exists
- Calculate percentage of validators including transaction
- Compare to required threshold

**Validity Assessment:**
- Re-evaluate transaction correctness
- Check if transaction is still valid
- Consider changed ledger state

**Network Consensus:**
- Gauge overall network agreement level
- Track validator positions
- Identify consensus direction

### Resolution Strategies

**Majority Rule:**
- Include transactions supported by validator majority
- Follow avalanche thresholds
- Converge on most popular position

**Conservative Approach:**
- Exclude disputed transactions when uncertain
- Prefer safety over liveness
- Avoid controversial inclusions

**Retry Mechanism:**
- Allow disputed transactions to be reconsidered later
- Return to transaction queue
- Can be included in future ledger

**Finality Assurance:**
- Ensure decisions are binding and consistent
- Once consensus reached, decision is final
- All validators apply same set

***

## Consensus State Determination

### checkConsensus: Parameters, Thresholds, Timeouts, and Return States

**Location:** `rippled/src/xrpld/consensus/Consensus.cpp` (lines 157-251)

**Purpose:** Determines the consensus state and whether agreement has been reached.

### Parameters

The function considers several inputs:

**Current State:**
- `prevProposers` - Number of proposers in previous round
- `currentProposers` - Number of proposers in current round
- `currentAgree` - Number of proposers agreeing with us
- `currentFinished` - Number of proposers who moved on

**Timing:**
- `previousAgreeTime` - How long previous round took
- `currentAgreeTime` - How long current round has taken so far

**Configuration:**
- `parms` - Consensus parameters (thresholds, timeouts)
- `proposing` - Whether we are proposing (affects quorum calculation)
- `stalled` - Whether all disputes are stalled

### Consensus Thresholds

**Supermajority Requirement:**
- **minCONSENSUS_PCT:** 80% validator agreement needed
- Ensures network safety against Byzantine validators
- Provides buffer against network issues and partitions

**Safety Margin:**
- 20% tolerance for Byzantine or offline validators
- Prevents minority from blocking consensus
- Balances safety with liveness

**Dynamic Timing:**
- Minimum consensus time: `ledgerMIN_CONSENSUS` (1.95s)
- Maximum consensus time: `ledgerMAX_CONSENSUS` (15s)
- Abandonment timeout: `ledgerABANDON_CONSENSUS` (120s)

### State Transition Logic

**Agreement Assessment:**

```cpp
if (checkConsensusReached(
        currentAgree,
        currentProposers,
        proposing,
        parms.minCONSENSUS_PCT,
        currentAgreeTime > parms.ledgerMAX_CONSENSUS,
        stalled,
        clog))
{
    return ConsensusState::Yes;
}
```

**Threshold Comparison:**
- Calculate percentage: `(agreeing * 100) / total`
- Compare to `minCONSENSUS_PCT` (80%)
- Account for self if proposing

**State Advancement:**
- Move to accepted phase when thresholds met
- Build final ledger
- Notify application layer

**Fallback Procedures:**
- If consensus cannot be reached, may expire
- Network may move on without full agreement
- Handles timeout scenarios

### Return States

**ConsensusState::No:**
- Consensus has not been reached yet
- Continue voting and updating positions
- Most common state during establish phase

**ConsensusState::Yes:**
- Consensus successfully reached
- 80%+ agreement on transaction set
- Or all disputes stalled at threshold

**ConsensusState::MovedOn:**
- 80% of validators moved on to next ledger
- This node fell behind network
- Must catch up

**ConsensusState::Expired:**
- Consensus process timed out
- Exceeded `ledgerABANDON_CONSENSUS` (120s)
- Fallback to best-effort ledger

### Validator Participation Tracking

**Active Validator Set:**
- Identify currently participating validators
- Count proposers in current round
- Compare to previous round

**Response Monitoring:**
- Track validator proposal submissions
- Monitor vote updates
- Detect non-responsive validators

**Weight Calculation:**
- All trusted validators have equal weight
- Non-trusted validators excluded from count
- Negative UNL validators excluded

**Timeout Handling:**
- Validators who don't respond treated as non-participating
- Don't count toward quorum
- May be added to Negative UNL if consistent

***

## Transaction Queue (TxQ) Ordering

### Fee Level Prioritization

**Location:** `rippled/src/xrpld/app/misc/TxQ.h`

**OrderCandidates Comparator:**

```cpp
bool operator()(MaybeTx const& lhs, MaybeTx const& rhs) const {
    if (lhs.feeLevel == rhs.feeLevel)
        return (lhs.txID ^ MaybeTx::parentHashComp) < (rhs.txID ^ MaybeTx::parentHashComp);
    return lhs.feeLevel > rhs.feeLevel;
}
```

**Fee Level Concept:**

**Base Fee:**
- Minimum fee required for transaction inclusion
- Set by network consensus
- Varies based on network load

**Fee Escalation:**
- Higher fees increase transaction priority within queue
- Transactions sorted by fee level
- Market mechanism for prioritization

**Dynamic Adjustment:**
- Fee requirements change based on network load
- When ledger is full, higher fees needed
- Market finds equilibrium price

**Tie-Breaking:**
- When fee levels are equal, use transaction ID XOR parent hash
- Ensures deterministic ordering
- Prevents manipulation

**Market Mechanism:**
- Users compete through fee levels for inclusion
- Higher demand increases required fees
- Supply-demand equilibrium

### Per-Account Transaction Limits

**Sequence Enforcement:**
- Transactions from same account must execute in sequence order
- Gaps in sequence numbers create blockers
- Ensures proper account state progression

**Queue Depth Limits:**
- Each account limited to `maximumTxnPerAccount` transactions in queue
- Prevents any account from monopolizing queue space
- Default limit prevents abuse

**Fairness Mechanism:**
- Ensures equitable access across all accounts
- No single account can dominate queue
- Balanced resource allocation

**Resource Protection:**
- Prevents queue exhaustion attacks
- Limits memory consumption per account
- Protects node resources

### Queue Management Strategies

**Priority Ordering:**
- Sort by fee level within canonical ordering constraints
- Higher fees get priority
- But still must respect account sequence

**Capacity Planning:**
- Balance queue size with processing capabilities
- Monitor queue depth
- Adjust acceptance criteria based on load

**Aging Policies:**
- Handle long-queued transactions appropriately
- May drop very old transactions
- Prevent unbounded queue growth

**Overflow Handling:**
- When queue full, drop lowest fee transactions
- Manage queue when demand exceeds capacity
- Clear space for higher value transactions

***

## Transaction Blockers and Retries

### Transaction Blocker Concepts

**Location:** `rippled/src/xrpld/app/misc/detail/TxQ.cpp`

**Dependency Tracking:**

**What Is a Blocker?**
- A transaction that prevents subsequent transactions from being processed
- Usually due to missing prior sequence number
- Creates dependency chain

**Example:**
- Account has sequence 100
- Receives transactions for sequences 101, 103, 104
- Transaction 101 must execute before 103 and 104
- If 101 is missing, 103 and 104 are "blocked"

**Account State Requirements:**
- Prerequisite conditions must be met
- Sufficient balance for fees and operations
- Proper sequence numbering
- Valid account state

**Sequence Gap Handling:**
- Detect missing sequence numbers in chains
- Hold later transactions until gaps filled
- Prevent out-of-order execution

**Resource Availability:**
- Verify sufficient account resources exist
- Check balance covers fees and reserves
- Ensure operations can complete

### Retry Mechanisms

**Temporary vs. Permanent Failures:**

**Temporary Failures:**
- Insufficient fee (can be retried with higher fee)
- Sequence gaps (retry when gap filled)
- Temporary resource shortages

**Permanent Failures:**
- Invalid signature
- Malformed transaction
- Impossible operations

**Backoff Strategies:**
- Implement intelligent retry timing
- Don't retry immediately
- Exponential backoff for repeated failures

**Retry Limits:**
- Each transaction has `retriesAllowed` count
- Prevents infinite retry loops
- Eventually drop failed transactions

**Success Tracking:**
- Monitor retry success rates
- Optimize retry policies
- Learn from patterns

### Queue Maintenance

**Periodic Cleanup:**
- Remove expired or invalid transactions
- Free memory from old transactions
- Maintain queue health

**State Synchronization:**
- Keep queue consistent with ledger state
- Remove transactions that became invalid
- Update based on ledger changes

**Memory Management:**
- Prevent unbounded queue growth
- Enforce size limits
- Prioritize valuable transactions

**Performance Monitoring:**
- Track queue efficiency metrics
- Monitor fill rates
- Optimize parameters

### Error Recovery

**Graceful Degradation:**
- Maintain service during partial failures
- Continue processing what's possible
- Degrade functionality rather than fail

**State Recovery:**
- Rebuild queue state after system restarts
- Recover from crashes
- Restore consistent state

**Consistency Checks:**
- Verify queue integrity periodically
- Detect and fix corruption
- Maintain data structure invariants

**Fallback Procedures:**
- Alternative processing when primary mechanisms fail
- Emergency modes for unusual conditions
- Ensure continuous operation

***

## Supporting Classes and Utilities

### RCLCxTx

**Location:** `rippled/src/xrpld/app/consensus/RCLCxTx.h`

**Purpose:** Adapts a SHAMapItem transaction for consensus

**Functionality:**
- Wraps transaction for consensus processing
- Provides common interface
- Handles XRPL-specific transaction details

### RCLTxSet

**Location:** `rippled/src/xrpld/app/consensus/RCLCxTx.h`

**Purpose:** Adapts a SHAMap to represent a set of transactions

**Functionality:**
- Transaction set representation
- Efficient storage and lookup
- Merkle tree structure for verification

### ConsensusProposal

**Location:** `rippled/src/xrpld/consensus/ConsensusProposal.h`

**Purpose:** Represents a proposal made by a node during consensus

**Contains:**
- Proposed transaction set hash
- Close time
- Sequence number
- Signature

***

## Summary

### Key Takeaways

- **Canonical ordering** ensures deterministic transaction sequencing across all validators
- **Salted account keys** prevent gaming of transaction order
- **Avalanche mechanism** resolves disputes and forces convergence
- **Fee-based prioritization** within canonical constraints enables market-driven inclusion
- **Per-account limits** and blockers maintain fairness and consistency
- **Robust retry and recovery** mechanisms handle failures gracefully

### The Big Picture

Transaction ordering in XRPL represents a sophisticated balance between fairness, efficiency, and determinism. By combining cryptographically-salted ordering, fee-based prioritization, and the avalanche consensus mechanism, the system achieves:

- **Predictable ordering** that cannot be gamed
- **Fast consensus** on transaction sets despite network asynchrony
- **Fair treatment** for all accounts and users
- **Robust handling** of disputes and failures
- **Efficient processing** of thousands of transactions per second

This comprehensive ordering system is fundamental to XRPL's ability to provide fast, fair, and secure transaction processing at global scale.

***

## References to Source Code

- `rippled/src/xrpld/app/misc/CanonicalTXSet.h` - Canonical transaction set header
- `rippled/src/xrpld/app/misc/CanonicalTXSet.cpp` - Canonical ordering implementation
- `rippled/src/xrpld/app/consensus/RCLConsensus.cpp` - Consensus proposal creation
- `rippled/src/xrpld/app/consensus/RCLCxTx.h` - Transaction and set adaptors
- `rippled/src/xrpld/consensus/Consensus.h` - Generic consensus template
- `rippled/src/xrpld/consensus/Consensus.cpp` - Consensus state determination
- `rippled/src/xrpld/consensus/ConsensusTypes.h` - Consensus data structures
- `rippled/src/xrpld/consensus/DisputedTx.h` - Dispute tracking and resolution
- `rippled/src/xrpld/consensus/ConsensusParms.h` - Consensus parameters
- `rippled/src/xrpld/app/misc/TxQ.h` - Transaction queue header
- `rippled/src/xrpld/app/misc/detail/TxQ.cpp` - Transaction queue implementation
- `rippled/src/xrpld/consensus/ConsensusProposal.h` - Proposal structure
