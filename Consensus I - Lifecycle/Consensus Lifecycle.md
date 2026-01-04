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
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │                    CONSENSUS ROUND                          │     │
│  │                                                             │     │
│  │   preStartRound() ──► startRound() ──► startRoundInternal() │     │
│  │         │                                                   │     │
│  │         ▼                                                   │     │
│  │   ┌──────────┐    ┌────────────┐    ┌──────────┐            │     │
│  │   │   OPEN   │───►│ ESTABLISH  │───►│ ACCEPTED │            │     │
│  │   └──────────┘    └────────────┘    └──────────┘            │     │
│  │         │               │                │                  │     │
│  │     timerEntry()    timerEntry()     onAccept()             │     │
│  └─────────────────────────────────────────────────────────────┘     │
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
// Application.cpp line 1385
// Start first consensus round
if (!m_networkOPs->beginConsensus(
        m_ledgerMaster->getClosedLedger()->header().hash, {}))
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
│   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐   │
│   │    Verify    │───►│    Clean     │───►│   Prepare   │   │
│   │    Sync      │    │    State     │    │    Data     │   │
│   └──────────────┘    └──────────────┘    └─────────────┘   │
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
      hash_set<NodeID_t> const& nowTrusted)
  {
      // Adaptor determines if we should propose
      bool proposing = adaptor_.preStartRound(prevLedger, nowTrusted);

      // Set initial mode
      ConsensusMode mode = proposing ?
          ConsensusMode::proposing : ConsensusMode::observing;

      // Enter internal initialization
      startRoundInternal(now, prevLedgerID, prevLedger, mode, clog);
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
│   Incoming           ┌──────────────┐       Proposal        │
│   Transactions ────► │ Open Ledger  │ ────► Playback        │
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

- Replays peer proposals received for this ledger that arrived early
- Recovers proposals from up to 10 recent proposals per peer
- Filters by prevLedgerID to find relevant proposals
- Allows fast catch-up when switching ledgers

**Example:**

```
Node behind network (on ledger 1000, network on 1001):
    ├─ Peer proposals for 1001 arrive → saved but ignored
    ├─ Node finishes 1000, startRound(1001) called
    ├─ playbackProposals() runs:
    │  └─ Finds saved proposals for 1001
    │  └─ Processes them immediately
    └─ Node instantly has peer positions → faster sync!
```

**timerEntry() in open phase:**

- Called periodically (~1 second via ledgerGRANULARITY)
- Evaluates close conditions via `checkLedger()`
- Calls `shouldCloseLedger()` to determine if ledger should close

**Example:**

```
t=0s:  Open phase starts
t=1s:  timerEntry() → shouldCloseLedger() → NO (too early)
t=2s:  timerEntry() → shouldCloseLedger() → YES (conditions met)
 └─ closeLedger() → phase = ESTABLISH
```

**Close conditions checked:**

- Minimum time elapsed (2s)
- Has transactions OR idle timeout
- Not opening too fast (≥ prevRoundTime/2)
- Enough peers have closed

### Stage 5: Close Decision

**Function: shouldCloseLedger**

Determines if the ledger should close:

```cpp
  // Key logic from shouldCloseLedger() in Consensus.cpp

  // If majority of peers already closed, follow them
  if ((proposersClosed + proposersValidated) > (prevProposers / 2)) {
      JLOG(j.trace()) << "Others have closed";
      return true;  // Close with the network
  }

  // No transactions? Only close after idle interval
  if (!anyTransactions) {
      return timeSincePrevClose >= idleInterval;  // Default 15s
  }

  // Enforce minimum ledger open time
  if (openTime < parms.ledgerMIN_CLOSE) {  // 2 seconds
      JLOG(j.debug()) << "Must wait minimum time before closing";
      return false;
  }

  // Don't close faster than half the previous consensus time
  // (allows slower validators to keep up)
  if (openTime < (prevRoundTime / 2)) {
      JLOG(j.debug()) << "Ledger has not been open long enough";
      return false;
  }

  return true;  // All conditions met
```

**Close Decision Process:**

**Step 1: Safety Checks**

- If timing is unusual (prevRoundTime or timeSincePrevClose out of range) → **CLOSE immediately**

**Step 2: Network Coordination**

- If (proposersClosed + proposersValidated) > prevProposers/2 → **CLOSE** (follow majority)

**Step 3: Transaction-Based Decision**

*Path A: No Transactions*

- Close only if: timeSincePrevClose ≥ ledgerIDLE_INTERVAL (15s)
- Purpose: Maintain regular ledger cadence even when idle

*Path B: Has Transactions*

- **Must satisfy:** openTime ≥ ledgerMIN_CLOSE (2s)
- **Must satisfy:** openTime ≥ prevRoundTime/2
- If both pass → **CLOSE**

**Key Parameters:**

| Parameter | Value | Purpose |
|-----------|-------|---------|
| ledgerMIN_CLOSE | 2s | Minimum time ledger must stay open |
| ledgerIDLE_INTERVAL | 15s | Maximum idle time before forcing close |
| prevRoundTime/2 | Dynamic | Throttle to prevent racing ahead |

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
│   │   No  ┴─► Loop back to timerEntry()                   │ │
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
      std::size_t prevProposers,
      std::size_t currentProposers,
      std::size_t currentAgree,
      std::size_t currentFinished,
      std::chrono::milliseconds previousAgreeTime,
      std::chrono::milliseconds currentAgreeTime,
      bool stalled,
      ConsensusParms const& parms,
      bool proposing)
  {
      // Check minimum time elapsed
      if (currentAgreeTime <= parms.ledgerMIN_CONSENSUS)
          return ConsensusState::No;

      // Check if we have reduced participation
      if (currentProposers < prevProposers * 3/4) {
          if (currentAgreeTime < previousAgreeTime + parms.ledgerMIN_CONSENSUS)
              return ConsensusState::No;
      }

      // Check if we reached 80% agreement on transaction set
      if (checkConsensusReached(
              currentAgree,
              currentProposers,
              proposing,
              parms.minCONSENSUS_PCT,  // Always 80%
              currentAgreeTime > parms.ledgerMAX_CONSENSUS,
              stalled))
          return ConsensusState::Yes;

      // Check if 80% of peers moved on without us
      if (checkConsensusReached(
              currentFinished,
              currentProposers,
              false,
              parms.minCONSENSUS_PCT,  // Always 80%
              currentAgreeTime > parms.ledgerMAX_CONSENSUS,
              false))
          return ConsensusState::MovedOn;

      // Check if consensus has taken too long
      // Timeout is bounded: min(max(prevTime × 10, 15s), 120s)
      std::chrono::milliseconds const maxAgreeTime =
          previousAgreeTime * parms.ledgerABANDON_CONSENSUS_FACTOR;  // 10

      if (currentAgreeTime > std::clamp(
                                 maxAgreeTime,
                                 parms.ledgerMAX_CONSENSUS,      // 15s minimum
                                 parms.ledgerABANDON_CONSENSUS)) // 120s maximum
          return ConsensusState::Expired;

      return ConsensusState::No;
  }
```

**Agreement Threshold:**

```cpp
    checkConsensusReached():
      agreement_pct = (agreeing / total) * 100
```

Required: 80% (fixed - minCONSENSUS_PCT)

This checks if 80% of validators have the SAME complete transaction set (same hash).

Two checks performed:

1. Do 80% agree with OUR position? → Yes
2. Did 80% move to next ledger? → MovedOn

**Important Distinction:**

The 80% threshold here is for final consensus (comparing complete transaction set hashes).

This is separate from avalanche voting (50%→65%→70%→95%) which determines which individual transactions to include during updateOurPositions().

**Consensus States:**

  | State   | Meaning                   | Action                 |
  |---------|---------------------------|------------------------|
  | No      | Not enough agreement yet  | Continue voting        |
  | Yes     | 80%+ agree with us        | Accept consensus!      |
  | MovedOn | 80%+ moved to next ledger | We're behind, catch up |
  | Expired | Took too long             | Bow out of consensus   |

### Stage 9: Acceptance

**Function: RCLConsensus::Adaptor::onAccept**

When consensus is reached:

```
┌─────────────────────────────────────────────────────────────┐
│                      ON ACCEPT                              │
│                                                             │
│   Consensus          ┌──────────────┐       ┌───────────┐   │
│   Result ──────────► │   buildLCL   │─────► │  Notify   │   │
│                      └──────────────┘       │  System   │   │
│                             │               └───────────┘   │
│                             ▼                               │
│                      ┌────────────────┐                     │
│                      │ LedgerMaster   │                     │
│                      │::consensusBuilt│                     │
│                      └────────────────┘                     │
│                             │                               │
│                             ▼                               │
│                      ┌──────────────┐                       │
│                      │Apply Disputed│                       │
│                      │ Transactions │                       │
│                      └──────────────┘                       │
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
│       └─── Mismatch? ──► switchLastClosedLedger()           │
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
3. **Convergent**: Avalanche mechanism's rising thresholds force consensus
4. **Resilient**: Handles network issues and recovery
5. **Continuous**: Rounds follow each other seamlessly

In the next chapter, we'll explore the ledger architecture that stores the results of consensus.
