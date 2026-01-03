# Consensus Lifecycle

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

### Introduction

This chapter traces through the complete consensus lifecycle, from the initiation of a new round to the acceptance of a validated ledger. Understanding this flow is essential for debugging consensus issues, implementing protocol changes, and reasoning about network behavior.

We follow the execution path through the actual codebase, showing how each component interacts to achieve distributed agreement.

### High-Level Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                     CONSENSUS LIFECYCLE                              │
│                                                                      │
│  Application.cpp                                                     │
│       │                                                              │
│       ▼                                                              │
│  NetworkOPs::beginConsensus()                                        │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    CONSENSUS ROUND                          │    │
│  │                                                             │    │
│  │   preStartRound() ──► startRound() ──► startRoundInternal() │    │
│  │         │                                                   │    │
│  │         ▼                                                   │    │
│  │   ┌──────────┐    ┌────────────┐    ┌──────────┐           │    │
│  │   │   OPEN   │───►│ ESTABLISH  │───►│ ACCEPTED │           │    │
│  │   └──────────┘    └────────────┘    └──────────┘           │    │
│  │         │               │                │                  │    │
│  │     timerEntry()    timerEntry()     onAccept()            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                              │
│       ▼                                                              │
│  NetworkOPs::endConsensus()                                          │
│       │                                                              │
│       ▼                                                              │
│  Loop back to beginConsensus()                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Stage 1: Initiating Consensus

**Entry Point: Application.cpp**

Consensus begins at application startup:

```cpp
// Application.cpp
// Start first consensus round
if (!m_networkOPs->beginConsensus(
        m_ledgerMaster->getClosedLedger()->info().hash, {}))
{
    JLOG(m_journal.fatal()) << "Unable to start consensus";
    return false;
}
```

**Key Actions:**
- Passes the hash of the last closed ledger
- NetworkOPs coordinates the consensus initiation
- Failure to start consensus is fatal

### Stage 2: Pre-Start Preparation

**Function: RCLConsensus::Adaptor::preStartRound**

Before a round begins, the system prepares:

```
┌─────────────────────────────────────────────────────────────┐
│                   PRE-START ROUND                           │
│                                                             │
│   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐  │
│   │    Verify    │───►│    Clean     │───►│   Prepare   │  │
│   │    Sync      │    │    State     │    │    Data     │  │
│   └──────────────┘    └──────────────┘    └─────────────┘  │
│                                                             │
│   • Check node synchronization                              │
│   • Verify ledger integrity                                 │
│   • Reset previous round state                              │
│   • Initialize data structures                              │
└─────────────────────────────────────────────────────────────┘
```

**Checks Performed:**
- Is the node synchronized with the network?
- Is the previous ledger valid and complete?
- Are all required data structures ready?

### Stage 3: Starting the Round

**Function: Consensus::startRound**

This function establishes the round parameters:

```cpp
void startRound(
    NetClock::time_point const& now,
    typename Ledger_t::ID const& prevLedgerID,
    Ledger_t prevLedger,
    hash_set<NodeID_t> const& nowUntrusted,
    bool proposing)
{
    // Initialize round parameters
    prevLedgerID_ = prevLedgerID;
    previousLedger_ = prevLedger;

    // Set initial mode
    mode_ = proposing ? ConsensusMode::proposing : ConsensusMode::observing;

    // Enter internal initialization
    startRoundInternal(now, prevLedgerID, prevLedger, proposing);
}
```

**Parameters Established:**
- Previous ledger hash and object
- Initial mode (proposing or observing)
- Timer initialization
- Peer position tracking

### Stage 4: Open Phase

**Function: Consensus::startRoundInternal → ConsensusPhase::open**

The ledger opens to accept transactions:

```
┌─────────────────────────────────────────────────────────────┐
│                      OPEN PHASE                             │
│                                                             │
│   Incoming           ┌──────────────┐        Proposal      │
│   Transactions ────► │ Open Ledger  │ ────► Playback       │
│                      └──────────────┘                       │
│                             │                               │
│                             ▼                               │
│                      ┌──────────────┐                       │
│                      │   timerEntry │                       │
│                      │     Loop     │                       │
│                      └──────┬───────┘                       │
│                             │                               │
│              shouldCloseLedger() == true?                   │
│                             │                               │
│                    YES      │      NO                       │
│                    ┌────────┴────────┐                      │
│                    ▼                 ▼                      │
│              closeLedger()     Continue waiting             │
└─────────────────────────────────────────────────────────────┘
```

**Key Activities:**

**playbackProposals():**
- Replays peer proposals received before round started
- Ensures consistency with network state
- Deduplicates and validates proposals

**timerEntry() in open phase:**
- Called periodically to check progress
- Evaluates close conditions via `checkLedger()`

### Stage 5: Close Decision

**Function: shouldCloseLedger**

Determines if the ledger should close:

```cpp
bool shouldCloseLedger(
    bool anyTransactions,
    std::size_t previousProposers,
    std::size_t proposersClosed,
    std::size_t proposersValidated,
    std::chrono::milliseconds previousTime,
    std::chrono::milliseconds currentTime,
    std::chrono::milliseconds openTime,
    std::chrono::seconds idleInterval,
    ConsensusParms const& parms,
    beast::Journal j)
{
    // Evaluate multiple factors...
}
```

**Close Conditions:**

| Factor | Condition |
| --- | --- |
| Time elapsed | Minimum time since last close |
| Transactions | At least one transaction present |
| Peer activity | Peers are closing their ledgers |
| Idle timeout | Maximum open time reached |

### Stage 6: Closing the Ledger

**Function: Consensus::closeLedger**

Transitions from open to establish phase:

```
┌─────────────────────────────────────────────────────────────┐
│                    CLOSE LEDGER                             │
│                                                             │
│   ┌─────────────────┐                                       │
│   │   Open Ledger   │                                       │
│   │   Transactions  │                                       │
│   └────────┬────────┘                                       │
│            │                                                │
│            ▼                                                │
│   ┌─────────────────┐                                       │
│   │ Build Initial   │                                       │
│   │    Proposal     │                                       │
│   └────────┬────────┘                                       │
│            │                                                │
│            ▼                                                │
│   ┌─────────────────┐                                       │
│   │ Notify Peers    │                                       │
│   │ of Closure      │                                       │
│   └────────┬────────┘                                       │
│            │                                                │
│            ▼                                                │
│   Phase = ESTABLISH                                         │
└─────────────────────────────────────────────────────────────┘
```

**Actions:**
- Finalizes transaction set for the ledger
- Creates initial consensus proposal
- Broadcasts closure to peers
- Transitions to establish phase

### Stage 7: Establish Phase

**Function: Consensus::phaseEstablish**

The core of consensus—exchanging proposals and resolving disputes:

```
┌─────────────────────────────────────────────────────────────┐
│                   ESTABLISH PHASE                           │
│                                                             │
│   ┌───────────────────────────────────────────────────────┐ │
│   │              ITERATIVE VOTING LOOP                    │ │
│   │                                                       │ │
│   │   timerEntry()                                        │ │
│   │       │                                               │ │
│   │       ▼                                               │ │
│   │   updateOurPositions()                                │ │
│   │       │                                               │ │
│   │       ├──► Evaluate peer proposals                    │ │
│   │       ├──► Adjust disputed transaction votes          │ │
│   │       └──► Broadcast updated position                 │ │
│   │                                                       │ │
│   │       │                                               │ │
│   │       ▼                                               │ │
│   │   haveConsensus()                                     │ │
│   │       │                                               │ │
│   │       ├──► checkConsensus()                           │ │
│   │       └──► checkConsensusReached()                    │ │
│   │                                                       │ │
│   │       │                                               │ │
│   │   ConsensusState?                                     │ │
│   │       │                                               │ │
│   │   No ─┴─► Loop back to timerEntry()                   │ │
│   │   Yes ──► Proceed to accept                           │ │
│   └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Key Functions:**

**updateOurPositions():**
- Processes peer proposals
- Adjusts votes on disputed transactions
- Updates local candidate transaction set

**shouldPause():**
- Checks if consensus should pause
- Handles lagging or non-participating peers
- Prevents premature advancement

**haveConsensus() → checkConsensus() → checkConsensusReached():**
- Evaluates if agreement threshold met
- Considers timing constraints
- Returns consensus state (Yes/No/MovedOn/Expired)

### Stage 8: Consensus Checking

**Consensus Evaluation Logic:**

```cpp
ConsensusState checkConsensus(
    std::size_t proposers,
    std::size_t agreements,
    std::chrono::milliseconds elapsed,
    ConsensusParms const& parms)
{
    // Check minimum time elapsed
    if (elapsed < parms.ledgerMIN_CONSENSUS)
        return ConsensusState::No;

    // Check proposer count
    if (proposers < parms.minProposers)
        return ConsensusState::No;

    // Check agreement percentage
    if (checkConsensusReached(proposers, agreements, parms))
        return ConsensusState::Yes;

    // Check maximum time
    if (elapsed >= parms.ledgerMAX_CONSENSUS)
        return ConsensusState::Expired;

    return ConsensusState::No;
}
```

**Agreement Threshold:**

```
checkConsensusReached():
    agreement_pct = (agreements / proposers) * 100

    Required percentage depends on avalanche state:
      init:  80%
      mid:   70%
      late:  60%
      stuck: 50%
```

### Stage 9: Acceptance

**Function: RCLConsensus::Adaptor::onAccept**

When consensus is reached:

```
┌─────────────────────────────────────────────────────────────┐
│                      ON ACCEPT                              │
│                                                             │
│   Consensus          ┌──────────────┐       ┌───────────┐  │
│   Result ──────────► │   buildLCL   │─────► │  Notify   │  │
│                      └──────────────┘       │  System   │  │
│                             │               └───────────┘  │
│                             ▼                              │
│                      ┌──────────────┐                      │
│                      │ LedgerMaster │                      │
│                      │::consensusBuilt                     │
│                      └──────────────┘                      │
│                             │                              │
│                             ▼                              │
│                      ┌──────────────┐                      │
│                      │Apply Disputed│                      │
│                      │ Transactions │                      │
│                      └──────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

**Sub-Functions:**

**buildLCL():**
- Constructs Last Closed Ledger from agreed transactions
- Applies transactions in canonical order
- Produces validated ledger object

**LedgerMaster::consensusBuilt():**
- Updates system's view of current ledger
- Triggers downstream processing

**Apply Disputed Transactions:**
- Attempts to apply transactions that weren't included
- Queues for next round if applicable

### Stage 10: Final Acceptance

**Function: RCLConsensus::Adaptor::doAccept**

Finalizes the acceptance process:

```
┌─────────────────────────────────────────────────────────────┐
│                      DO ACCEPT                              │
│                                                             │
│   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐  │
│   │ BuildLedger  │───►│  Process     │───►│  Accept     │  │
│   │ ::buildLedger│    │  TxQueue     │    │ OpenLedger  │  │
│   └──────────────┘    └──────────────┘    └─────────────┘  │
│                                                             │
│   Constructs the       Updates fee        Finalizes open   │
│   ledger object        metrics, removes   ledger with      │
│   from agreed set      included txns      local txns       │
└─────────────────────────────────────────────────────────────┘
```

**Key Actions:**

**BuildLedger::buildLedger:**
- Final ledger construction
- State persistence

**app_.getTxQ().processClosedLedger:**
- Updates fee metrics
- Removes included transactions
- Manages deferred transactions

**app_.openLedger().accept:**
- Applies local transactions not in closed ledger
- Prepares open ledger for next round

### Stage 11: End Consensus

**Function: NetworkOPsImp::endConsensus**

Completes the round and prepares for the next:

```
┌─────────────────────────────────────────────────────────────┐
│                    END CONSENSUS                            │
│                                                             │
│   checkLastClosedLedger()                                   │
│       │                                                     │
│       ├─── Match? ──► Continue normally                     │
│       │                                                     │
│       └─── Mismatch? ──► switchLastClosedLedger()          │
│                              │                              │
│                              ├──► clearNeedNetworkLedger()  │
│                              ├──► processClosedLedger()     │
│                              └──► accept open ledger        │
│                                                             │
│   m_ledgerMaster.switchLCL()                                │
│       │                                                     │
│       ▼                                                     │
│   setMode() ──► Update operational mode                     │
│       │                                                     │
│       ▼                                                     │
│   Prepare for next round                                    │
└─────────────────────────────────────────────────────────────┘
```

### Stage 12: Begin Next Round

**Function: NetworkOPsImp::beginConsensus**

The cycle continues:

```cpp
// Begin next consensus round
reportConsensusStateChange();  // Log state change
RCLConsensus::startRound();    // Initialize next round
// Loop back to Stage 2
```

### Complete Lifecycle Diagram

```
┌────────────────────────────────────────────────────────────────────┐
│                  COMPLETE CONSENSUS CYCLE                          │
│                                                                    │
│   Application Start                                                │
│         │                                                          │
│         ▼                                                          │
│   ┌──────────────────┐                                             │
│   │  beginConsensus  │◄─────────────────────────────────────────┐  │
│   └────────┬─────────┘                                          │  │
│            │                                                    │  │
│            ▼                                                    │  │
│   ┌──────────────────┐                                          │  │
│   │  preStartRound   │                                          │  │
│   └────────┬─────────┘                                          │  │
│            │                                                    │  │
│            ▼                                                    │  │
│   ┌──────────────────┐                                          │  │
│   │    startRound    │                                          │  │
│   └────────┬─────────┘                                          │  │
│            │                                                    │  │
│            ▼                                                    │  │
│   ┌──────────────────┐    Timer                                 │  │
│   │   OPEN PHASE     │◄────────────┐                            │  │
│   │                  │             │                            │  │
│   │  playbackProposals             │                            │  │
│   │  timerEntry()    │─────────────┘                            │  │
│   │  shouldCloseLedger ──► closeLedger()                        │  │
│   └────────┬─────────┘                                          │  │
│            │                                                    │  │
│            ▼                                                    │  │
│   ┌──────────────────┐    Timer                                 │  │
│   │ ESTABLISH PHASE  │◄────────────┐                            │  │
│   │                  │             │                            │  │
│   │  timerEntry()    │─────────────┘                            │  │
│   │  updateOurPositions                                         │  │
│   │  haveConsensus() ──► onAccept()                             │  │
│   └────────┬─────────┘                                          │  │
│            │                                                    │  │
│            ▼                                                    │  │
│   ┌──────────────────┐                                          │  │
│   │  ACCEPTED PHASE  │                                          │  │
│   │                  │                                          │  │
│   │  buildLCL()      │                                          │  │
│   │  doAccept()      │                                          │  │
│   └────────┬─────────┘                                          │  │
│            │                                                    │  │
│            ▼                                                    │  │
│   ┌──────────────────┐                                          │  │
│   │   endConsensus   │──────────────────────────────────────────┘  │
│   └──────────────────┘                                             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Summary

**Lifecycle Stages:**

| Stage | Function | Purpose |
| --- | --- | --- |
| 1 | Application.cpp | Entry point |
| 2 | preStartRound | Preparation and validation |
| 3 | startRound | Parameter initialization |
| 4 | Open phase | Transaction collection |
| 5 | shouldCloseLedger | Close decision |
| 6 | closeLedger | Transition to establish |
| 7 | Establish phase | Proposal exchange |
| 8 | checkConsensus | Agreement evaluation |
| 9 | onAccept | Consensus acceptance |
| 10 | doAccept | Final processing |
| 11 | endConsensus | Round completion |
| 12 | beginConsensus | Next round |

**Key Takeaways:**

1. **Timer-driven**: Periodic events drive state transitions
2. **Iterative**: Establish phase loops until consensus
3. **Adaptive**: Avalanche mechanism adjusts thresholds
4. **Resilient**: Handles network issues and recovery
5. **Continuous**: Rounds follow each other seamlessly

In the next chapter, we'll explore the ledger architecture that stores the results of consensus.
