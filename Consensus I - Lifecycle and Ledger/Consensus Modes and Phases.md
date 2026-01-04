# Consensus Modes and Phases

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

### Introduction

The consensus process operates as a state machine with two orthogonal dimensions: the **mode** (the node's relationship with the network) and the **phase** (the current stage of the consensus round). Understanding these states is crucial for debugging, monitoring, and developing consensus-related functionality.

This chapter provides a detailed breakdown of each mode and phase, their transitions, and how they affect node behavior.

### ConsensusMode: Node Operating States

The `ConsensusMode` enum defines how a node participates in consensus:

```cpp
enum class ConsensusMode {
    proposing,       // Actively proposing positions
    observing,       // Watching but not proposing
    wrongLedger,     // Out of sync with network
    switchedLedger   // Recently switched to different ledger
};
```

#### Mode: proposing

**Definition:** The node is actively participating in consensus by broadcasting proposals.

```
┌─────────────────────────────────────────┐
│            PROPOSING MODE               │
│                                         │
│  • Broadcasting own position to peers   │
│  • Voting on transaction inclusion      │
│  • Contributing to consensus outcome    │
│  • Full participant in the network      │
└─────────────────────────────────────────┘
```

**Requirements for Proposing:**
- Node is synchronized with network
- Has valid signing keys configured
- Connected to sufficient peers
- Running on correct ledger chain

**Behavior:**
- Creates and signs proposals
- Broadcasts position changes
- Participates in dispute resolution
- Contributes to the quorum

#### Mode: observing

**Definition:** The node monitors consensus but doesn't submit proposals.

```
┌─────────────────────────────────────────┐
│            OBSERVING MODE               │
│                                         │
│  • Receiving proposals from peers       │
│  • Tracking consensus progress          │
│  • NOT broadcasting own position        │
│  • Passive network participant          │
└─────────────────────────────────────────┘
```

**When Observing:**
- Node is configured as non-validator
- Insufficient peers for meaningful participation
- Deliberately passive (tracking node)
- Testing or development scenarios

**Behavior:**
- Collects peer proposals
- Builds local view of consensus
- Applies final ledger when consensus reached
- Does not influence outcome

#### Mode: wrongLedger

**Definition:** The node is operating on a different ledger than the network consensus.

```
┌─────────────────────────────────────────┐
│          WRONG LEDGER MODE              │
│                                         │
│  • Local LCL differs from network       │
│  • Proposals may be rejected by peers   │
│  • Attempting to resynchronize          │
│  • Potential fork detected              │
└─────────────────────────────────────────┘
```

**Causes:**
- Network partition recovery
- Missed ledger closes
- Database corruption or gaps
- Slow synchronization

**Recovery Actions:**
- Acquire correct ledger from peers
- Switch to network's preferred LCL
- Resume normal operation

#### Mode: switchedLedger

**Definition:** The node recently switched to a different ledger chain.

```
┌─────────────────────────────────────────┐
│        SWITCHED LEDGER MODE             │
│                                         │
│  • Just completed ledger switch         │
│  • Transitional state                   │
│  • Re-establishing consensus position   │
│  • Will transition to proposing/observing│
└─────────────────────────────────────────┘
```

**Behavior:**
- Temporary state after correction
- Clears stale proposal data
- Rebuilds peer position tracking
- Transitions to appropriate mode

### Mode Transitions

```
                    ┌──────────────┐
                    │  PROPOSING   │←─────────────────┐
                    └──────┬───────┘                  │
                           │                          │
              Network      │ Out of                   │ Resync
              issues       │ sync                     │ complete
                           ▼                          │
                    ┌──────────────┐                  │
         ┌─────────►│ WRONG LEDGER │──────────────────┤
         │          └──────┬───────┘                  │
         │                 │                          │
         │        Ledger   │ switch                   │
         │        acquired │                          │
         │                 ▼                          │
         │          ┌──────────────┐                  │
         │          │  SWITCHED    │──────────────────┘
         │          │   LEDGER     │
         │          └──────────────┘
         │
         │          ┌──────────────┐
         └──────────│  OBSERVING   │
                    └──────────────┘
```

### ConsensusPhase: Round Stages

The `ConsensusPhase` enum defines the current stage within a consensus round:

```cpp
enum class ConsensusPhase {
    open,      // Collecting transactions
    establish, // Exchanging proposals, resolving disputes
    accepted   // Consensus reached, ledger closed
};
```

#### Phase: open

**Definition:** The ledger is open and accepting new transactions.

```
┌─────────────────────────────────────────────────────────┐
│                    OPEN PHASE                           │
│                                                         │
│   Transactions   ──►  Open Ledger  ──►  Candidate Set  │
│                                                         │
│   • Accepting new transactions from clients             │
│   • Building initial transaction set                    │
│   • Waiting for close conditions                        │
│   • Collecting peer proposals for playback              │
└─────────────────────────────────────────────────────────┘
```

**Close Conditions:**
- Minimum time elapsed: ledgerMIN_CLOSE (2s)
- At least one transaction present OR idle timeout (ledgerIDLE_INTERVAL = 15s)
- Not opening too fast: openTime ≥ prevRoundTime/2
- Alternatively: >50% of validators already closed (follow network)

**Key Functions:**
- `shouldCloseLedger()`: Evaluates if closure conditions are met
- `playbackProposals()`: Replays peer proposals for consistency

**Timing:**
- Typical duration: 2-15 seconds
- Minimum: 2s (ledgerMIN_CLOSE)
- Maximum idle: 15s (ledgerIDLE_INTERVAL)

#### Phase: establish

**Definition:** Proposals are being exchanged and disputes are being resolved.

```
┌─────────────────────────────────────────────────────────┐
│                  ESTABLISH PHASE                        │
│                                                         │
│     ┌─────────┐        ┌─────────┐        ┌─────────┐  │
│     │ Node A  │◄──────►│ Node B  │◄──────►│ Node C  │  │
│     │Proposal │        │Proposal │        │Proposal │  │
│     └─────────┘        └─────────┘        └─────────┘  │
│           │                 │                  │        │
│           └─────────────────┼──────────────────┘        │
│                             ▼                           │
│                    ┌──────────────┐                     │
│                    │   Disputes   │                     │
│                    │  Resolution  │                     │
│                    └──────────────┘                     │
│                             │                           │
│                             ▼                           │
│                    ┌──────────────┐                     │
│                    │   Converge   │                     │
│                    │   Positions  │                     │
│                    └──────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

**Activities:**
- Exchange positions with peers
- Identify disputed transactions
- Vote on inclusion/exclusion
- Update local position based on peer input
- Check for consensus achievement

**Timing:**
- **Minimum duration:** ledgerMIN_CONSENSUS (1.95s) - must wait at least this long before consensus can be declared
- **Maximum duration:** ledgerMAX_CONSENSUS (15s) - max time to pause for laggards
- **Check interval:** ledgerGRANULARITY (1s) - how often state is checked and positions updated
- **Abandonment:** ledgerABANDON_CONSENSUS (120s) - absolute maximum before giving up
- **Typical duration:** 2-10 seconds in healthy network conditions

**Avalanche Thresholds:**

As time progresses through the establish phase, the threshold for including disputed transactions increases:

| State | Time Threshold | Agreement Required | Purpose |
|-------|----------------|-------------------|---------|
| init | 0% of prevRoundTime | 50% | Initial voting - easy to add transactions |
| mid | 50% of prevRoundTime | 65% | Mid-consensus - slightly harder |
| late | 85% of prevRoundTime | 70% | Late consensus - harder still |
| stuck | 200% of prevRoundTime | 95% | Stuck - very hard to change |

The time percentage is calculated as:
```
convergePercent = (currentRoundTime × 100) / max(prevRoundTime, avMIN_CONSENSUS_TIME)
```

This rising threshold forces the network to converge on a stable transaction set.

**Key Functions:**
- `updateOurPositions()`: Adjusts local votes based on peer input
- `haveConsensus()`: Checks if agreement threshold reached
- `createDisputes()`: Identifies transactions with disagreement

#### Phase: accepted

**Definition:** Consensus has been reached and the ledger is being finalized.

```
┌─────────────────────────────────────────────────────────┐
│                   ACCEPTED PHASE                        │
│                                                         │
│   Agreed Set  ──►  Build LCL  ──►  Notify System       │
│                                                         │
│   • Consensus achieved (or timeout)                     │
│   • Building Last Closed Ledger                         │
│   • Applying transactions to state                      │
│   • Preparing for next round                            │
└─────────────────────────────────────────────────────────┘
```

**Actions:**
- Build new ledger from agreed transactions
- Update ledger master with new LCL
- Process remaining transaction queue
- Begin next consensus round

**Key Functions:**
- `onAccept()`: Handles consensus acceptance
- `buildLCL()`: Constructs the Last Closed Ledger
- `doAccept()`: Finalizes acceptance and prepares next round

### Phase Transitions

```
                    ┌───────────────┐
       ┌───────────►│     OPEN      │
       │            └───────┬───────┘
       │                    │
       │           Close    │  shouldCloseLedger()
       │           conditions│  returns true
       │           met      │
       │                    ▼
       │            ┌───────────────┐
       │            │   ESTABLISH   │◄─────┐
       │            └───────┬───────┘      │
       │                    │              │
       │                    │   Update     │ No consensus yet
       │                    │   positions  │ (loop back)
       │                    │              │
       │           Consensus▼              │
       │           reached  ├──────────────┘
       │                    │
       │                    ▼
       │            ┌───────────────┐
       │            │   ACCEPTED    │
       │            └───────┬───────┘
       │                    │
       │                    │  Begin next round
       └────────────────────┘
```

### Timer-Driven Progression

The consensus process is driven by periodic timer events:

```cpp
void Consensus::timerEntry() {
    // Check current phase and take appropriate action
    switch (phase_) {
        case ConsensusPhase::open:
            phaseOpen();
            break;
        case ConsensusPhase::establish:
            phaseEstablish();
            break;
        case ConsensusPhase::accepted:
            // Prepare for next round
            break;
    }
}
```

**Timer Responsibilities:**
- Advance phase when conditions met
- Check for ledger issues (`checkLedger()`)
- Update proposal positions
- Detect stalls or timeouts

### Consensus State Outcomes

The `ConsensusState` enum captures the result of consensus attempts:

```cpp
enum class ConsensusState {
    No,       // No consensus reached
    MovedOn,  // Network moved on without this node
    Expired,  // Consensus timed out
    Yes       // Consensus successfully reached
};
```

**State Implications:**

| State | Meaning | Action |
| --- | --- | --- |
| No | Insufficient agreement | Continue voting |
| MovedOn | Network ahead of this node | Resync required |
| Expired | Timeout reached | Build best-effort ledger |
| Yes | Agreement achieved | Finalize and close |

### Monitoring and Debugging

**JSON Serialization:**

Consensus state can be serialized for monitoring:

```cpp
Json::Value getJson() const {
    Json::Value ret;
    ret["phase"] = to_string(phase_);
    ret["mode"] = to_string(mode_);
    ret["proposers"] = proposerCount_;
    ret["agreements"] = agreementCount_;
    // ... additional state
    return ret;
}
```

**RPC Access:**

The `consensus_info` RPC provides real-time consensus status:

```json
{
    "result": {
        "info": {
            "phase": "establish",
            "mode": "proposing",
            "proposers": 35,
            "current_ms": 2100
        }
    }
}
```

### Summary

**Consensus Modes:**

| Mode | Participation | Typical Scenario |
| --- | --- | --- |
| proposing | Active | Normal validator operation |
| observing | Passive | Tracking/non-validator nodes |
| wrongLedger | Recovery | Network partition recovery |
| switchedLedger | Transitional | After ledger correction |

**Consensus Phases:**

| Phase | Activity | Duration |
| --- | --- | --- |
| open | Collect transactions | 2-15s typical (min: 2s, idle timeout: 15s) |
| establish | Exchange proposals, resolve disputes | 2-10s typical (min: 1.95s, max: 15s) |
| accepted | Build ledger, prepare next round | Brief finalization (no fixed duration) |

**Key Takeaways:**

1. **Mode reflects node's network relationship**
2. **Phase reflects progress within a round**
3. **Timer events drive state transitions**
4. **States are observable via RPC**
5. **Recovery mechanisms handle edge cases**

In the next chapter, we'll trace through the complete consensus lifecycle from round initiation to ledger acceptance.
