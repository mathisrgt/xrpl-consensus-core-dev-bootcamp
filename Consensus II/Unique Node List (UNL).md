# Unique Node List (UNL) and Negative UNL

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

## Introduction

The Unique Node List (UNL) forms the foundation of XRPL's consensus mechanism by defining the set of trusted validators that participate in consensus. The Negative UNL (N-UNL) extends this system by providing a mechanism to temporarily disable unreliable validators without permanent removal, ensuring network reliability while maintaining decentralization.

### What is the UNL?

**Unique Node List (UNL):**
- Set of trusted validators that participate in consensus
- Determines which nodes can propose and validate ledgers
- Each node operator independently chooses their UNL
- Typically published by trusted organizations

**Negative UNL (N-UNL):**
- Mechanism to temporarily disable unreliable validators
- Subset of UNL validators that are excluded from consensus participation
- Provides automatic recovery without permanent removal
- Managed through on-ledger consensus process

### Core Purpose

**Security Maintenance:**
- Only reliable validators participate in consensus
- Prevents malicious or faulty nodes from disrupting network

**Automatic Recovery:**
- Temporarily unreliable validators can be re-enabled
- No permanent exclusion maintains validator diversity

**Network Resilience:**
- Prevents disruption from validator failures
- Maintains consensus even with poor-performing nodes

**Decentralization Preservation:**
- Avoids permanent validator exclusion
- Maintains distributed nature of the network

***

## Negative UNL Architecture

### System Components

The Negative UNL system consists of several key components:

**Validator Scoring System:**
- Tracks reliability by counting validations over time
- Measures participation rates during consensus rounds
- Built using validation history from recent ledgers

**Candidate Selection Process:**
- Identifies underperforming validators for disabling
- Identifies recovered validators for re-enabling
- Uses performance thresholds to determine eligibility

**Deterministic Voting Mechanism:**
- Uses cryptographic randomness for fair selection
- Prevents bias in validator management decisions
- Ensures all nodes arrive at the same decision

**Automatic Management:**
- Operates without human intervention
- Self-regulating system maintains network health
- Integrated into normal consensus process

***

## Negative UNL Ledger Object

### Structure

**Location:** Stored in the ledger state

The Negative UNL ledger object stores:
- **`sfDisabledValidators`:** List of validator public keys currently disabled
- **`sfValidatorToDisable`:** Validator pending disable action
- **`sfValidatorToReEnable`:** Validator pending re-enable action

### Lifecycle

1. **Initialization:** The Negative UNL is empty or contains previously disabled validators from the last validated ledger

2. **Voting:** Each consensus round on a flag ledger, the `NegativeUNLVote::doVoting` method evaluates validator performance and proposes modifications

3. **Modification:** If a validator is to be disabled or re-enabled, a `ttUNL_MODIFY` transaction is created and added to the ledger's transaction set

4. **Application:** Once the transaction is included in a validated ledger, the Negative UNL is updated accordingly via `applyUNLModify`

### Accessors

**Ledger Methods:**
- `Ledger::negativeUNL()` - Returns the set of currently disabled validators
- `Ledger::validatorToDisable()` - Returns validator pending disable
- `Ledger::validatorToReEnable()` - Returns validator pending re-enable

These methods allow the consensus engine and other components to query the current Negative UNL state.

***

## NegativeUNLVote Class

**Location:**
- `rippled/src/xrpld/app/misc/NegativeUNLVote.cpp`
- `rippled/src/xrpld/app/misc/NegativeUNLVote.h`

The `NegativeUNLVote` class manages the entire voting process for the Negative UNL.

### Key Responsibilities

1. **Collect and Score Validator Performance:** Build a score table of validator reliability over a configurable interval using validation history

2. **Identify Candidates:** Determine which validators should be disabled (added to Negative UNL) or re-enabled (removed from Negative UNL) based on reliability scores and protocol thresholds

3. **Deterministic Selection:** Use cryptographic randomness (previous ledger hash) to fairly select candidates for action

4. **Transaction Construction:** Create and add `ttUNL_MODIFY` transactions to the ledger to modify the Negative UNL

5. **Track New Validators:** Maintain a grace period for new validators to avoid disabling them prematurely

6. **Logging and Auditability:** Log all relevant actions and errors for debugging and monitoring

***

## Performance Monitoring and Scoring

### Validator Scoring: buildScoreTable()

**Location:** `rippled/src/xrpld/app/misc/NegativeUNLVote.cpp`

**Purpose:** Constructs a score table mapping each validator NodeID in the current UNL to the number of trusted validations they have issued over a recent interval of ledgers.

**Process:**

1. Determines the scoring interval (typically `FLAG_LEDGER_INTERVAL` ledgers)
2. Iterates through recent ledgers to collect validation data
3. Counts validations per validator
4. Returns a map of `NodeID -> validation_count`

**Edge Cases:**
- If there is insufficient ledger history, returns `std::nullopt` and no voting occurs
- If the local node has too few or too many validations, returns `std::nullopt` to ensure voting fairness

**Why This Matters:**

The score table provides an objective measurement of validator reliability. By tracking actual validation submissions over time, the system can identify validators that are consistently participating versus those that are unreliable or offline.

### Continuous Monitoring

**Performance Tracking:**
- System tracks validator participation over `FLAG_LEDGER_INTERVAL` periods
- Records validation submissions for each consensus round
- Builds comprehensive reliability metrics

**Performance Assessment:**
- Compares actual vs. expected validation counts
- Identifies patterns of reliability or unreliability
- Calculates participation percentages

***

## Candidate Selection and Voting

### Finding Candidates: findAllCandidates()

**Location:** `rippled/src/xrpld/app/misc/NegativeUNLVote.cpp`

**Purpose:** Identifies which validators are candidates to be disabled (added to Negative UNL) or re-enabled (removed from Negative UNL).

**Candidate Identification:**

**Disable Candidates:**
- Validators with validation count below the low water mark
- Not already on the Negative UNL
- Not pending disable
- Not in the new validator grace period

**Re-enable Candidates:**
- Validators currently on the Negative UNL
- Validation count above the high water mark
- Not pending re-enable

**Edge Cases:**
- If the Negative UNL is full (25% of UNL), no disables are proposed
- If no candidates are found, no modification is proposed
- New validators have a grace period before they can be disabled

### Key Thresholds and Parameters

The Negative UNL system uses several critical parameters defined in `NegativeUNLVote.cpp`:

```cpp
static constexpr size_t negativeUNLLowWaterMark = FLAG_LEDGER_INTERVAL * 50 / 100;
static constexpr size_t negativeUNLHighWaterMark = FLAG_LEDGER_INTERVAL * 80 / 100;
static constexpr size_t negativeUNLMinLocalValsToVote = FLAG_LEDGER_INTERVAL * 90 / 100;
static constexpr size_t newValidatorDisableSkip = FLAG_LEDGER_INTERVAL * 2;
static constexpr float negativeUNLMaxListed = 0.25;
```

**Low Water Mark:**
- Threshold: 50% of `FLAG_LEDGER_INTERVAL`
- Validators below this may be disabled
- Indicates consistently poor performance

**High Water Mark:**
- Threshold: 80% of `FLAG_LEDGER_INTERVAL`
- Disabled validators above this may be re-enabled
- Shows recovery and reliable participation

**Minimum Local Validations to Vote:**
- Threshold: 90% of `FLAG_LEDGER_INTERVAL`
- Local node must have this many validations to participate in voting
- Ensures only well-synchronized nodes vote

**New Validator Disable Skip:**
- Grace period: 2 × `FLAG_LEDGER_INTERVAL`
- New validators are protected from being disabled during this period
- Allows time for validators to establish themselves

**Safety Limits:**
- Maximum 25% of UNL can be on Negative UNL
- Prevents excessive network disruption
- Ensures sufficient validators remain active

### Deterministic Selection: choose()

**Location:** `rippled/src/xrpld/app/misc/NegativeUNLVote.cpp`

**Purpose:** Deterministically selects a single NodeID from a list of candidates using a randomizing pad (typically the previous ledger hash).

**Process:**

1. Takes a list of candidate NodeIDs
2. Uses the previous ledger hash as a cryptographic random source
3. Combines each candidate's ID with the random pad
4. Selects the candidate with the highest resulting hash

**Why Deterministic Selection?**

- **Fairness:** No node can bias the selection process
- **Consensus:** All nodes make the same selection independently
- **Transparency:** Selection is based on public ledger data
- **Security:** Prevents gaming or manipulation of the process

***

## Transaction Construction and Application

### Creating UNL Modify Transactions: addTx()

**Location:** `rippled/src/xrpld/app/misc/NegativeUNLVote.cpp`

**Purpose:** Constructs and adds a Negative UNL modification transaction to the SHAMap of transactions for the next ledger.

**Transaction Structure:**

The `ttUNL_MODIFY` transaction contains:
- **Transaction type:** `ttUNL_MODIFY`
- **Validator public key:** The validator to disable or re-enable
- **Action:** ToDisable or ToReEnable
- **Ledger sequence:** The ledger sequence for which this modification is intended

**Process:**

1. Constructs a `STTx` object with the appropriate fields
2. Adds the transaction to the SHAMap (transaction set)
3. Logs the action for auditability

**Failure Mode:**
- If the transaction cannot be added to the SHAMap, a warning is logged and no change occurs
- This can happen if the SHAMap is full or corrupted

### Voting Process: doVoting()

**Location:** `rippled/src/xrpld/app/misc/NegativeUNLVote.cpp`

**Purpose:** Orchestrates the entire Negative UNL voting process for a consensus round.

**Process:**

1. **Check if Voting is Needed:**
   - Only vote on flag ledgers
   - Ensure `featureNegativeUNL` amendment is enabled

2. **Build Score Table:**
   - Call `buildScoreTable()` to get validator performance metrics
   - If score table cannot be built, exit without voting

3. **Find Candidates:**
   - Call `findAllCandidates()` to identify disable/re-enable candidates
   - Separate into disable and re-enable lists

4. **Prioritize Re-enables:**
   - If there are re-enable candidates, prefer those first
   - This helps restore network capacity

5. **Select Candidate:**
   - Use `choose()` to deterministically select one candidate
   - Create appropriate transaction (disable or re-enable)

6. **Add Transaction:**
   - Call `addTx()` to add the modification transaction to the ledger

**Edge Cases:**
- If no score table can be built, no modification is proposed
- If no candidates are found, no modification is proposed
- Only one validator is modified per flag ledger to ensure gradual changes

### Ledger Application: applyUNLModify()

**Location:** `rippled/src/xrpld/app/tx/detail/Change.cpp`

**Purpose:** Processes a `UNL_MODIFY` transaction to either disable or re-enable a validator in the Negative UNL.

**Process:**

1. **Validation Checks:**
   - Verify it's a flag ledger
   - Check transaction fields are present and valid
   - Ensure ledger sequence matches

2. **Disable Action:**
   - Verify validator is not already disabled
   - Verify validator is not pending disable
   - Add validator to Negative UNL
   - Clear any pending re-enable for this validator

3. **Re-enable Action:**
   - Verify validator is currently on Negative UNL
   - Verify validator is not pending re-enable
   - Remove validator from Negative UNL
   - Clear any pending disable for this validator

4. **State Update:**
   - Modify the Negative UNL ledger object
   - Update pending disable/re-enable fields

**Failure Modes:**

Transaction is not applied if:
- Not a flag ledger
- Transaction fields are missing or invalid
- Disabling a validator already disabled or pending disable
- Re-enabling a validator not in Negative UNL or already pending re-enable
- Ledger sequence does not match

***

## Quorum Dynamics

### Adaptive Consensus Requirements

The presence of validators on the Negative UNL affects the quorum calculation for consensus.

**Base Quorum Calculation:**

Formula: `max(80% of effective UNL, 60% of total UNL)`

This ensures sufficient validator participation while accounting for disabled validators.

**Effective UNL:**
- Total UNL minus validators on Negative UNL
- Represents currently active validator set
- Used for quorum calculations

**Dynamic Adjustment:**
- Quorum automatically adjusts as validators are disabled/enabled
- Maintains consensus viability under changing conditions
- Prevents network halt from insufficient participation

**Safety Mechanisms:**
- Minimum viable consensus requirements maintained
- Maximum 25% can be disabled prevents excessive quorum reduction
- Ensures network can always reach consensus

***

## Consensus Integration

### Integration with Consensus Process

The Negative UNL voting and transaction process is integrated into the consensus round:

**Voting Trigger:**
- `doVoting()` is called during ledger closing on flag ledgers
- Integrated into the consensus finalization process
- Runs alongside fee and amendment voting

**Consensus Engine Updates:**
- Validator list and Negative UNL are updated in application state
- Consensus engine ensures only trusted, non-disabled validators are counted
- Quorum calculations automatically adjust for disabled validators

**RPC and API Exposure:**
- Negative UNL is exposed via RPC commands for monitoring
- Internal APIs provide access to current N-UNL state
- Diagnostics and debugging tools can query N-UNL status

### Interaction with Fee and Amendment Voting

**Parallel Voting:**
- Negative UNL voting is performed alongside fee and amendment voting
- During a voting ledger, if `featureNegativeUNL` is enabled, all voting types run concurrently
- Ensures all governance changes are considered in the same consensus round

**Transaction Ordering:**
- Fee, amendment, and UNL modify transactions are all pseudo-transactions
- Processed during ledger close, not from transaction queue
- Applied in deterministic order

***

## State Management and Recovery

### Consensus State Transitions

Consensus in XRPL is tracked using the following states:

**ConsensusState Enum:**
- **No:** Consensus has not been reached
- **Yes:** Consensus has been reached
- **MovedOn:** The network has moved on to a new ledger
- **Expired:** The consensus process has expired

**State Transitions and N-UNL Impact:**

**No State:**
- *Triggered by:* Insufficient agreement among validators
- *Action:* Continue collecting validations, may attempt consensus in subsequent rounds
- *N-UNL Impact:* Disabled validators not counted toward agreement

**Yes State:**
- *Triggered by:* Sufficient agreement (meeting quorum) among validators
- *Action:* Ledger is accepted, network moves forward
- *N-UNL Impact:* Quorum calculated based on effective UNL (excluding disabled)

**MovedOn State:**
- *Triggered by:* Network progresses despite lack of full consensus
- *Action:* Validators accept new ledger
- *N-UNL Impact:* May occur if too many validators disabled

**Expired State:**
- *Triggered by:* Consensus process times out
- *Action:* Process abandoned, network may recover or resync
- *N-UNL Impact:* Could result from excessive disabled validators

### Edge Cases and Recovery

**Consensus Cannot Be Reached:**
- If consensus fails (No or Expired state):
  - Network continues attempting consensus in subsequent rounds
  - May move on to new ledger (MovedOn state)
  - Temporary divergence until consensus restored

**Negative UNL is Full:**
- Maximum size: 25% of total UNL (`negativeUNLMaxListed = 0.25`)
- If N-UNL is full:
  - No additional validators can be added
  - Re-enables are prioritized to free space
  - Network may struggle if more validators fail

**No Eligible Candidates:**
- If no validators meet disable/re-enable criteria:
  - No modification transaction is proposed
  - System waits for next flag ledger
  - Status quo is maintained

**Insufficient Ledger History:**
- If `buildScoreTable()` cannot gather enough data:
  - No voting occurs
  - Prevents premature or unfair disable decisions
  - Waits for more history to accumulate

***

## Example Scenario

### Disabling an Unreliable Validator

**Scenario:** The network detects that validator `A` is consistently missing validations.

**Step 1: Monitoring**
- Validator `A` submits only 45 out of 100 expected validations during the scoring period
- This is below the low water mark (50%)

**Step 2: Scoring**
- During the next flag ledger, `NegativeUNLVote::doVoting` is called
- `buildScoreTable()` constructs reliability metrics for all UNL validators
- Validator `A` has a score of 45, below `negativeUNLLowWaterMark` (50)

**Step 3: Candidate Selection**
- `findAllCandidates()` identifies `A` as a disable candidate
- Checks that `A` is not already disabled or pending disable
- Checks that N-UNL is not full (< 25% of UNL)

**Step 4: Deterministic Selection**
- `choose()` deterministically selects `A` from the candidate list using previous ledger hash
- All nodes independently arrive at the same selection

**Step 5: Transaction Construction**
- `addTx()` creates a `ttUNL_MODIFY` transaction to disable `A`:
  ```cpp
  addTx(seq, A, ToDisable, initialSet);
  ```
- Transaction is added to the ledger's transaction set

**Step 6: Consensus**
- Ledger with the transaction is proposed and goes through normal consensus
- Once validated, ledger becomes immutable

**Step 7: Application**
- `applyUNLModify()` processes the transaction
- Validator `A` is added to the Negative UNL
- `A` is excluded from consensus calculations until re-enabled

**Step 8: Recovery**
- If `A` later recovers and submits > 80% validations
- On a future flag ledger, `A` becomes a re-enable candidate
- Process repeats to remove `A` from Negative UNL

***

## Benefits and Design Goals

### System Advantages

**Automatic Resilience:**
- Network self-heals from validator issues
- No manual intervention required
- Maintains network health automatically

**Fairness and Transparency:**
- Deterministic selection prevents bias
- All actions recorded on-ledger and auditable
- Every node follows the same rules

**Network Stability:**
- Gradual changes prevent sudden disruptions (one validator per flag ledger)
- Maintains consensus continuity
- Smooth adaptation to changing conditions

**Decentralization Preservation:**
- No central authority controls participation
- Maintains distributed validator ecosystem
- Temporary, not permanent, exclusion

**Performance Optimization:**
- Removes unreliable validators from quorum calculations
- Improves consensus speed and reliability
- Prevents network stalls from offline validators

***

## Supporting Classes and Utilities

### ValidatorList

**Purpose:** Manages the trusted validator set, Negative UNL, and quorum calculation

**Key Methods:**
- Updates UNL based on published lists
- Calculates effective UNL (total - disabled)
- Provides quorum requirements
- Tracks validator trust levels

### Ledger

**Purpose:** Stores the Negative UNL state and provides accessors

**Key Methods:**
- `negativeUNL()` - Returns current disabled validators
- `validatorToDisable()` - Returns pending disable
- `validatorToReEnable()` - Returns pending re-enable
- Manages N-UNL ledger object

### RCLConsensus

**Purpose:** Integrates Negative UNL voting into the consensus process

**Key Responsibilities:**
- Calls `doVoting()` during flag ledgers
- Coordinates with fee and amendment voting
- Ensures voting occurs at appropriate times

### SHAMap

**Purpose:** Holds the set of transactions for each ledger

**N-UNL Usage:**
- Stores `ttUNL_MODIFY` transactions
- Ensures transactions are included in ledger
- Provides transaction set management

***

## Summary

### Key Takeaways

- **UNL** defines the set of trusted validators for each node
- **Negative UNL** provides automatic, temporary validator disabling
- **Deterministic voting** ensures all nodes agree on changes
- **Gradual adaptation** maintains network stability
- **Automatic recovery** preserves decentralization

### The Big Picture

The Unique Node List and Negative UNL system creates a robust, self-maintaining network that automatically manages validator reliability without sacrificing decentralization. By combining deterministic voting, performance monitoring, and automatic recovery, XRPL ensures both security and distributed operation. The system gracefully handles validator failures, recovers from network issues, and maintains consensus even under adverse conditions—all without central control or manual intervention.

***

## References to Source Code

- `rippled/src/xrpld/app/misc/NegativeUNLVote.cpp` - Negative UNL voting implementation
- `rippled/src/xrpld/app/misc/NegativeUNLVote.h` - NegativeUNLVote class definition
- `rippled/src/xrpld/app/tx/detail/Change.cpp` - applyUNLModify implementation
- `rippled/src/xrpld/app/misc/detail/ValidatorList.cpp` - Validator list management
- `rippled/src/xrpld/app/ledger/Ledger.cpp` - Ledger N-UNL accessors
- `rippled/src/xrpld/app/consensus/RCLConsensus.cpp` - Consensus integration
- `rippled/src/xrpld/consensus/Consensus.h` - Generic consensus template
- `rippled/src/xrpld/consensus/ConsensusTypes.h` - Consensus state definitions
