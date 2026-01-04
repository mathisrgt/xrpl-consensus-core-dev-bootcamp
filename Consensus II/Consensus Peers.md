# Consensus Peers

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

## Introduction

Consensus Peers are specialized network participants that actively participate in the XRPL consensus process. They serve as the critical bridge between the network overlay and the consensus algorithm, enabling distributed agreement on ledger state across the network without central authority.

### What Are Consensus Peers?

**Definition:** Consensus peers are network validators that exchange proposals, track consensus state, and coordinate transaction validation during each consensus round.

**Core Responsibilities:**
- **Proposal Management:** Handle incoming consensus proposals from network peers
- **State Coordination:** Track and synchronize consensus states across participants
- **Dispute Resolution:** Manage conflicting views and facilitate agreement
- **Network Integration:** Interface between consensus logic and peer-to-peer communication

### Why Consensus Peers Matter

- **Decentralization:** Enable distributed decision-making without central authority
- **Reliability:** Provide fault tolerance through redundant validation
- **Consistency:** Ensure all network participants agree on transaction ordering
- **Performance:** Optimize consensus efficiency through intelligent peer management

***

## Peer Proposal Handling

### Understanding Proposals

A **proposal** is a formal suggestion for the next ledger state submitted by validators. Each proposal:
- Contains a proposed transaction set and ledger close information
- Must be cryptographically signed and verified
- Is submitted during specific phases of the consensus round
- Influences the final transaction set included in the ledger

### Proposal Processing Pipeline

The proposal handling process involves three main functions in the codebase:

#### 1. RCLConsensus::peerProposal

**Location:** `rippled/src/xrpld/app/consensus/RCLConsensus.cpp`

**Functionality:**
- Entry point for processing a new peer proposal
- Acquires a lock on `mutex_` for thread safety
- Delegates to `consensus_.peerProposal(now, newProposal)`

This is the XRPL-specific entry point that receives proposals from the network layer.

#### 2. Consensus<Adaptor>::peerProposal

**Location:** `rippled/src/xrpld/consensus/Consensus.h`

**Functionality:**
- Logs the incoming proposal
- Extracts the peer ID
- Updates `recentPeerPositions_` for this peer (maintains a deque of up to 10 recent proposals)
- Calls `peerProposalInternal(now, newPeerPos)` and returns its result

This function maintains a rolling history of recent proposals for each peer, which is useful for tracking peer behavior and detecting anomalies.

#### 3. Consensus<Adaptor>::peerProposalInternal

**Location:** `rippled/src/xrpld/consensus/Consensus.h`

**Functionality:**

This is where the detailed proposal validation and processing occurs:

1. **Consensus Phase Check:** If `phase_ == ConsensusPhase::accepted`, returns `false` (proposals not accepted after consensus is reached)

2. **Update Time:** Sets `now_ = now;`

3. **Extract Proposal and Peer ID**

4. **Ledger ID Check:** If `newPeerProp.prevLedger() != prevLedgerID_`, logs and returns `false` (proposal is for a different ledger)

5. **Dead Node Check:** If `deadNodes_.find(peerID) != deadNodes_.end()`, logs and returns `false` (peer has bowed out)

6. **Proposal Sequence Check:** If the peer already has a position in `currPeerPositions_` and the new proposal's sequence number is not greater, returns `false` (prevents replay attacks)

7. **Bow Out Handling:** If `newPeerProp.isBowOut()`:
   - Logs the bow out
   - Removes the peer from `currPeerPositions_`
   - Adds to `deadNodes_`
   - Removes votes from all disputes for this peer
   - Returns `true`

8. **Update/Insert Peer Position:** Updates or inserts the peer's position in `currPeerPositions_`

9. **Initial Proposal Handling:** If `newPeerProp.isInitial()`, increments the count for the peer's close time in `rawCloseTimes_.peers`

10. **Transaction Set Handling:** If the transaction set referenced by the proposal is not in `acquired_`, attempts to acquire it via `adaptor_.acquireTxSet`. If already acquired and there is a consensus result, updates disputes for this peer.

11. **Return:** Returns `true` if the proposal was processed

### Trust-Based Handling

Proposals are processed differently based on trust:

**Trusted Proposals** (from validators in the Unique Node List):
- Higher priority processing
- Direct influence on consensus decisions
- Automatic relay to other network participants

**Untrusted Proposals** (from non-UNL validators):
- Limited processing based on network configuration
- May be relayed depending on network policies
- Used for network awareness but not consensus decisions

### Proposal Lifecycle Management

- **Tracking:** Monitor proposal status throughout consensus round
- **Suppression:** Prevent duplicate proposal propagation using suppression IDs
- **Expiration:** Remove outdated proposals from consideration based on `proposeFRESHNESS` parameter (20s)
- **Archival:** Maintain historical record for analysis and debugging

***

## Peer State Tracking

The consensus algorithm maintains several data structures to track peer state throughout the consensus process.

### State Information Categories

The system tracks four main categories of peer information:

- **Consensus Position:** Current validator stance on ledger proposals
- **Network Status:** Connection quality and communication reliability
- **Validation History:** Track record of previous consensus participation
- **Timing Metrics:** Response times and synchronization accuracy

### Key Data Structures

#### recentPeerPositions_

**Type:** `hash_map<NodeID_t, std::deque<PeerPosition_t>>`

**Purpose:** Maintains a rolling history (up to 10) of recent proposals from each peer

**Usage:** Updated in `Consensus<Adaptor>::peerProposal` whenever a new proposal is received

**Why it matters:** Provides historical context for analyzing peer behavior patterns and detecting anomalies

#### currPeerPositions_

**Type:** `hash_map<NodeID_t, PeerPosition_t>`

**Purpose:** Tracks the current (most recent) proposal from each peer

**Usage:** Updated in `Consensus<Adaptor>::peerProposalInternal` when a new proposal is accepted

**Why it matters:** This is the active set of positions used for consensus calculations

#### deadNodes_

**Type:** `hash_set<NodeID_t>`

**Purpose:** Tracks peers that have "bowed out" of consensus or are otherwise disqualified

**Usage:** Updated in `Consensus<Adaptor>::peerProposalInternal` when a peer bows out

**Why it matters:** Prevents continued processing of proposals from peers that have explicitly withdrawn from the consensus round

### Dynamic State Management

- **Real-time Updates:** Continuously monitor and update peer states as new proposals arrive
- **State Transitions:** Track changes in peer consensus positions over time
- **Availability Monitoring:** Detect and respond to peer disconnections
- **Performance Metrics:** Measure and evaluate peer contribution quality

### Trust and Reputation Systems

- **UNL Membership:** Distinguish between trusted and untrusted peers
- **Historical Performance:** Weight peer input based on past reliability
- **Behavioral Analysis:** Identify and respond to anomalous peer behavior
- **Dynamic Adjustment:** Adapt trust levels based on ongoing performance

### State Synchronization

- **Consensus Alignment:** Ensure peers share common understanding of current state
- **Conflict Detection:** Identify discrepancies between peer states through dispute tracking
- **Recovery Mechanisms:** Handle state inconsistencies and synchronization failures
- **Optimization:** Minimize state tracking overhead while maintaining accuracy

***

## Dispute Management

Disputes arise when different validators propose different transaction sets. The consensus algorithm must detect and resolve these disputes to reach agreement.

### Types of Disputes

- **Transaction Set Conflicts:** Different views on which transactions to include
- **Timing Disagreements:** Disputes over ledger close timing
- **Validation Conflicts:** Competing proposals for the same ledger position
- **Network Partitions:** Temporary splits in network connectivity

### Creating Disputes: createDisputes()

**Location:** `rippled/src/xrpld/consensus/Consensus.h`

**Functionality:**

1. Asserts that `result_` is set (there is an active consensus result)
2. Checks if the transaction set has already been compared; if so, returns
3. If the sets are identical, returns (no disputes to create)
4. Compares the local and incoming transaction sets, logging differences
5. For each difference:
   - Asserts the difference is valid
   - Retrieves the transaction from the appropriate set
   - If a dispute for this transaction already exists, skips
   - Constructs a new `Dispute_t` (DisputedTx) for the transaction
   - For each current peer position, updates the peer's vote in the dispute
   - Shares the disputed transaction with peers
   - Inserts the dispute into `result_->disputes`

### The DisputedTx Class

**Location:** `rippled/src/xrpld/consensus/DisputedTx.h`

**Purpose:** Manages the state and voting process for transactions that are disputed during consensus

**Key Methods:**

- **`setVote(NodeID_t const& peer, bool votesYes)`:** Records or updates a peer's vote, updating `yays_` and `nays_` counters

- **`unVote(NodeID_t const& peer)`:** Removes a peer's vote from the dispute

- **`updateVote(int percentTime, bool proposing, ConsensusParms const& p)`:** Updates the local node's vote based on peer votes and consensus parameters. This implements the avalanche mechanism where the threshold for accepting a transaction increases over time.

- **`stalled(ConsensusParms const& p, bool proposing, int peersUnchanged) const`:** Determines if consensus on the transaction has stalled (when agreement has reached the minimum threshold in either direction)

### Updating Disputes: updateDisputes()

**Location:** `rippled/src/xrpld/consensus/Consensus.h`

**Functionality:**

1. Asserts that `result_` is set
2. If the transaction set has not been compared, calls `createDisputes`
3. For each dispute:
   - Calls `setVote(node, other.exists(d.tx().id()))` to update the peer's vote
   - If the vote changes, resets `peerUnchangedCounter_` to 0

### Resolution Strategies

- **Majority Consensus:** Follow the position supported by most trusted validators
- **Weighted Voting:** Consider validator reliability through UNL membership
- **Timeout Mechanisms:** Resolve disputes through time-based fallback procedures (avalanche thresholds)
- **Escalation Protocols:** Handle persistent disputes through the "stuck" avalanche state (95% threshold)

### Dispute Prevention

- **Proactive Communication:** Maintain clear channels between consensus participants through the overlay network
- **Standardization:** Ensure consistent interpretation of consensus rules
- **Monitoring Systems:** Early detection of potential dispute conditions through proposal tracking
- **Network Health:** Maintain robust connectivity to prevent partition-based disputes

***

## Consensus State Transitions

The consensus process progresses through distinct phases, coordinated by peer input and timing constraints.

### Consensus Round Phases

As defined in the `ConsensusPhase` enum:

- **Open Phase:** Collect and evaluate proposed transaction sets
- **Establish Phase:** Build consensus on the preferred transaction set through iterative voting
- **Accepted Phase:** Finalize agreement and close the ledger

### Checking for Consensus: haveConsensus()

**Location:** `rippled/src/xrpld/consensus/Consensus.h`

**Functionality:**

1. Asserts that `result_` is set
2. Counts how many peers agree/disagree with the local proposal by iterating through `currPeerPositions_`
3. Determines if consensus is stalled by checking all disputes
4. Calls `checkConsensus` to determine the consensus state (No/MovedOn/Expired/Yes)
5. Handles each consensus state:
   - `No`: Returns false (continue voting)
   - `Expired`: If not enough rounds have passed, continues; otherwise, logs and calls `leaveConsensus`
   - `MovedOn`: Logs error (network moved on without this node)
   - Otherwise: Returns true (consensus reached)

### Leaving Consensus: leaveConsensus()

**Location:** `rippled/src/xrpld/consensus/Consensus.h`

**Functionality:**

1. If in `proposing` mode and not already bowed out:
   - Marks the position as "bowed out" and shares with the network
2. Switches to `observing` mode and logs the action

This allows a node to gracefully withdraw from consensus if it cannot keep up or has issues.

### Establish Phase Processing: phaseEstablish()

**Location:** `rippled/src/xrpld/consensus/Consensus.h`

**Functionality:**

1. Logs entry into the establish phase and asserts `result_` is set
2. Increments counters for unchanged peers and establish rounds
3. Updates round timing and proposer count
4. Calculates convergence percentage (how far through the consensus time we are)
5. Enforces minimum consensus time (ledgerMIN_CONSENSUS = 1.95s)
6. Calls `updateOurPositions` to adjust votes based on peer input
7. Checks if the node should pause or if consensus has been reached via `haveConsensus()`
8. Checks for close time consensus
9. If all conditions are met, finalizes consensus, transitions to `accepted` phase, and notifies the application layer

### Updating Our Positions: updateOurPositions()

**Location:** `rippled/src/xrpld/consensus/Consensus.h`

**Functionality:**

1. Asserts that `result_` is set and retrieves consensus parameters
2. Computes cutoff times for peer and local proposals using `proposeFRESHNESS` parameter
3. Iterates over current peer positions:
   - Removes stale proposals (older than cutoff time) and their votes from disputes
   - Tallies close time votes for non-stale proposals
4. Updates the local node's transaction set and position if necessary:
   - Adjusts votes on disputed transactions based on avalanche thresholds
   - Logs and shares changes to our position
   - Updates disputes accordingly

This is the heart of the consensus algorithm where the node adjusts its position based on what peers are proposing.

### State Transition Triggers

- **Time-based:** Automatic progression based on consensus timing (ledgerGRANULARITY = 1s timer ticks)
- **Threshold-based:** Advance when sufficient agreement is reached (80% supermajority)
- **Event-driven:** Respond to specific network or consensus events (bow outs, ledger switches)
- **Fallback Conditions:** Handle exceptional situations requiring special transitions (timeouts, expiration)

### Peer Coordination During Transitions

- **Synchronization:** Ensure all peers transition states together through shared timing parameters
- **Communication:** Broadcast state changes to relevant network participants via proposals
- **Validation:** Verify that transitions are legitimate through proposal sequence numbers
- **Recovery:** Handle peers that fail to transition properly through dead node tracking

***

## Network Communication

### Communication Patterns

- **Broadcast:** Send information to all connected peers simultaneously
- **Targeted:** Direct communication with specific peers or validator subsets
- **Relay:** Forward messages through the network to reach distant peers
- **Request-Response:** Interactive communication for specific information needs (e.g., transaction set acquisition)

### Sharing Proposals: RCLConsensus::Adaptor::share

**Location:** `rippled/src/xrpld/app/consensus/RCLConsensus.cpp`

**Functionality:**

1. Constructs a `TMProposeSet` protocol message from a peer's proposal, including:
   - Sequence number
   - Close time
   - Transaction hash
   - Previous ledger hash
   - Public key
   - Signature

2. Relays this proposal to peers using the overlay network:
   ```cpp
   app_.overlay().relay(prop, peerPos.suppressionID(), peerPos.publicKey());
   ```

3. The suppression ID is used to prevent duplicate relays of the same proposal across the network

### Overlay Network Integration

**Overlay:** Manages peer connections, message broadcasting, and relaying in the XRPL peer-to-peer network

**Location:** `rippled/src/xrpld/overlay/detail/OverlayImpl.h`

**PeerSet:** Abstract interface for managing sets of network peers, adding peers, sending protocol messages, and retrieving peer IDs

**Location:** `rippled/src/xrpld/overlay/PeerSet.h`

**PeerImp:** Implements the core logic for a peer connection, including message sending/receiving, resource usage, and protocol handling

**Location:** `rippled/src/xrpld/overlay/detail/PeerImp.h`

### Message Types and Purposes

- **Proposals (TMProposeSet):** Consensus position announcements from validators
- **Validations (TMValidation):** Confirmations of agreed-upon ledger states
- **Transaction Sets (TMHaveTransactionSet/TMGetObjectByHash):** Exchange of transaction data
- **Status Updates:** Peer state and availability information

### Network Optimization Strategies

- **Message Suppression:** Prevent redundant message propagation using suppression IDs
- **Priority Queuing:** Ensure critical consensus messages receive priority
- **Bandwidth Management:** Optimize network resource utilization
- **Latency Minimization:** Reduce communication delays affecting consensus timing

### Reliability and Fault Tolerance

- **Redundant Paths:** Multiple communication routes between peers through the overlay network
- **Error Detection:** Identify and handle communication failures
- **Retry Mechanisms:** Transaction set acquisition can retry on failure
- **Graceful Degradation:** Maintain functionality despite network issues (nodes can bow out rather than fail)

***

## Supporting Data Structures

### ConsensusParms

**Location:** `rippled/src/xrpld/consensus/ConsensusParms.h`

Encapsulates configuration parameters for the consensus process, including:
- Timing parameters (ledgerMIN_CONSENSUS, ledgerMAX_CONSENSUS, etc.)
- Thresholds (minCONSENSUS_PCT = 80%)
- Avalanche cutoffs and progression rules

### ConsensusCloseTimes

**Location:** `rippled/src/xrpld/consensus/ConsensusTypes.h`

Tracks proposed close times from peers and self, used to reach agreement on when the ledger should close.

### ConsensusResult

**Location:** `rippled/src/xrpld/consensus/ConsensusTypes.h`

Encapsulates the result of a consensus round, including:
- The transaction set
- Proposal position
- Disputes
- Compared sets
- Round timing
- Consensus state
- Proposer count

### ConsensusMode and ConsensusPhase

**Location:** `rippled/src/xrpld/consensus/ConsensusTypes.h`

Enumerations defining:
- **Modes:** proposing, observing, wrongLedger, switchedLedger
- **Phases:** open, establish, accepted

### RCLCxPeerPos

**Location:** `rippled/src/xrpld/app/consensus/RCLCxPeerPos.h`

Represents a peer's position (proposal) in the consensus process, including:
- Public key
- Signature
- Suppression ID
- Proposal object with ledger hashes and transaction set hash

***

## Integration and Design Principles

### System Integration Points

- **Consensus Algorithm:** Direct interface with core consensus logic through the Adaptor pattern
- **Network Overlay:** Integration with peer-to-peer communication layer
- **Ledger Management:** Coordination with ledger creation and validation
- **Transaction Processing:** Interface with transaction handling systems through transaction set acquisition

### Key Design Principles

- **Modularity:** Clear separation between consensus logic (generic Consensus class) and XRPL-specific logic (RCLConsensus Adaptor)
- **Scalability:** Efficient handling of large numbers of network participants through hash maps and optimized data structures
- **Reliability:** Robust operation despite network and peer failures (bow outs, dead nodes, timeouts)
- **Performance:** Optimized for low-latency consensus decision-making through parallel processing and efficient algorithms

### Benefits to XRPL Network

- **Decentralized Governance:** No single point of control or failure
- **Fast Settlement:** Rapid consensus enables quick transaction finality (typically 3-5 seconds)
- **Network Resilience:** Fault tolerance through distributed validation and automatic recovery
- **Transparent Operation:** Open and verifiable consensus process

***

## Summary

### Key Takeaways

- **Consensus Peers** are the critical bridge between network communication and consensus decisions
- **Multi-layered approach** handles proposals, tracks states, manages disputes, and coordinates transitions
- **Trust-based system** distinguishes between UNL and non-UNL validators
- **Robust design** ensures network reliability and performance despite various challenges

### The Big Picture

Consensus Peers functionality represents the sophisticated orchestration layer that enables XRPL's distributed consensus mechanism to operate efficiently and reliably across a global network of validators and participants. By carefully managing peer proposals, tracking state, resolving disputes through the avalanche mechanism, and coordinating state transitions, the system achieves fast, reliable, and decentralized agreement on ledger state.

***

## References to Source Code

- `rippled/src/xrpld/app/consensus/RCLConsensus.cpp` - XRPL-specific consensus implementation
- `rippled/src/xrpld/app/consensus/RCLConsensus.h` - RCLConsensus class definition
- `rippled/src/xrpld/app/consensus/RCLCxPeerPos.h` - Peer position representation
- `rippled/src/xrpld/consensus/Consensus.h` - Generic consensus template implementation
- `rippled/src/xrpld/consensus/ConsensusTypes.h` - Consensus data structures and enums
- `rippled/src/xrpld/consensus/ConsensusParms.h` - Consensus parameters
- `rippled/src/xrpld/consensus/DisputedTx.h` - Dispute management
- `rippled/src/xrpld/overlay/detail/OverlayImpl.h` - Overlay network implementation
- `rippled/src/xrpld/overlay/PeerSet.h` - Peer set interface
- `rippled/src/xrpld/overlay/detail/PeerImp.h` - Peer connection implementation
