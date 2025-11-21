# Homework 1: Transaction Consensus Journey

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

### Objective

This homework deepens your understanding of how a **transaction travels through the XRPL consensus process**. You will trace a transaction from network entry through consensus rounds to final validation, identifying key implementation files and understanding the role of each component.

**Format**: Written report (PDF or Markdown) including diagrams, code snippets, and explanations.

***

### Task

Perform a hands-on exploration of Rippled's consensus and transaction processing code. Trace how a transaction moves from submission through the consensus phases to inclusion in a validated ledger.

#### Requirements

1. **Setup**
   * Have access to the Rippled source code (clone from GitHub if needed)
   * Optionally run Rippled in standalone mode for testing
   * Select a transaction type to analyze (e.g., Payment)

2. **Phase 1: Network Entry**
   * Locate the entry point in `PeerImp.cpp`
   * Document the `onMessage()` and `handleTransaction()` functions
   * Identify:
     * How transactions are deserialized
     * What initial validation occurs
     * How invalid transactions are rejected early
   * Provide file location and relevant line numbers

3. **Phase 2: Open Ledger Application**
   * Trace the flow through `OpenLedger.cpp`
   * Document the `apply_one()` function
   * Explore the Transaction Queue (`TxQ.cpp`):
     * How fees are evaluated
     * How transactions are queued or applied directly
   * Identify what checks occur before a transaction enters the open ledger

4. **Phase 3: Consensus Proposal**
   * Explore `RCLConsensus.cpp` and `Consensus.ipp`
   * Document how transactions become part of a proposal
   * Identify:
     * The `timerEntry()` function's role
     * How `startRound()` initializes consensus
     * How transaction sets are created

5. **Phase 4: Consensus Rounds**
   * Document the iterative voting process
   * Explore `updateOurPositions()` and `haveConsensus()`
   * Identify:
     * How validators exchange positions
     * How disputes are detected and resolved
     * What determines when consensus is reached

6. **Phase 5: Final Validation**
   * Trace ledger building in `BuildLedger.cpp`
   * Document the `buildLedger()` function
   * Identify:
     * How the final transaction set is applied
     * How the ledger hash is calculated
     * How validations are created and broadcast

***

### Deliverable

A written report containing:

* **Flow Diagram**: Visual representation of a transaction's journey through all five phases
* **File Reference Table**: For each phase, include:

  | Phase | File | Key Functions | Line Numbers |
  |-------|------|---------------|--------------|
  | Network Entry | PeerImp.cpp | onMessage(), handleTransaction() | ~1200-1300 |
  | ... | ... | ... | ... |

* **Code Snippets**: 5-10 lines for each phase showing key logic
* **Error Analysis**: For each phase, describe:
  * What could cause a transaction to fail
  * How failures are handled
  * What error codes are returned
* **Status Tracking**: Document how a transaction's status changes:
  * Tentative → Proposed → Accepted → Validated

***

### Tips

* Use `grep` or your IDE to trace function calls across files
* Start from `NetworkOPs::processTransaction()` and follow the call chain
* Pay attention to the `ApplyFlags` and how they change through phases
* The `TER` (Transaction Engine Result) codes indicate success or failure
* Use the Rippled documentation in `README.md` files within each directory

#### Learning Goals

By completing this homework, you should be able to:

* Navigate Rippled's consensus and transaction processing code
* Understand the complete lifecycle of a transaction
* Identify where validation occurs at each phase
* Explain how consensus transforms individual proposals into network agreement
* Trace the path from submission to finality
