# Homework 2: Consensus Dispute Engineering

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

### Objective

This homework provides hands-on experience with **XRPL's dispute resolution mechanisms**. You will create a consensus test scenario where validators are evenly split on transaction inclusion, triggering and observing how the consensus algorithm resolves disagreements.

**Format**: Written report (PDF or Markdown) with code, test output, and analysis.

***

### Background

Disputes occur when validators disagree on whether to include a transaction. The `DisputedTx` class tracks votes and adjusts thresholds over time:

* **Initial threshold**: 50% (simple majority)
* **Later rounds**: 65%, then 70%, finally 95%
* **Vote switching**: Validators can change positions based on peer consensus

Understanding how disputes are resolved is crucial for understanding network resilience and the guarantees provided by XRPL consensus.

***

### Task

Create and analyze a consensus test that generates a 50% validator dispute, observing how the consensus algorithm resolves the disagreement.

#### Requirements

1. **Setup**
   * Navigate to `rippled/src/test/consensus/`
   * Open `Consensus_test.cpp`
   * Locate the existing `testDisputes()` function (~line 1000+) to understand existing patterns

2. **Create Test Function**
   * Add a new test function `testCustomDispute()` after `testDisputes()`
   * Create two equal groups of validators (e.g., 3 + 3 = 6 total)
   * Set up trust relationships where all validators trust each other
   * Configure realistic network delays between validators

3. **Engineer the Dispute**
   * Have Group A propose transaction `Tx{100}`
   * Have Group B propose transaction `Tx{200}`
   * Have both groups propose a common transaction `Tx{999}`
   * This creates a scenario where validators disagree on 100 and 200 but agree on 999

4. **Monitor Resolution**
   * Implement a `DisputeMonitor` struct to track:
     * Position changes by each validator
     * When validators accept ledgers
     * Final transaction set composition
   * Run consensus rounds and observe the resolution process

5. **Analyze Results**
   * Document which transactions made it into the final ledger
   * Record how many consensus rounds were needed
   * Identify which validators changed positions first
   * Explain what determined the final outcome

***

### Implementation Guide

**Test Function Skeleton:**

```cpp
void testCustomDispute()
{
    using namespace csf;
    using namespace std::chrono;
    testcase("custom 50% dispute");

    ConsensusParms const parms{};
    Sim sim;

    // Create two equal groups
    PeerGroup groupA = sim.createGroup(3);
    PeerGroup groupB = sim.createGroup(3);
    PeerGroup network = groupA + groupB;

    // Set up trust and connections
    network.trust(network);
    SimDuration delay = round<milliseconds>(0.2 * parms.ledgerGRANULARITY);
    network.connect(network, delay);

    // Initial synchronization
    sim.run(1);
    BEAST_EXPECT(sim.synchronized());

    // Create dispute scenario
    for (Peer* peer : groupA)
        peer->openTxs.insert(Tx{100});
    for (Peer* peer : groupB)
        peer->openTxs.insert(Tx{200});
    for (Peer* peer : network)
        peer->openTxs.insert(Tx{999});

    // Run and analyze...
}
```

**Register Your Test:**

In the `run()` function, add:
```cpp
testCustomDispute();
```

***

### Deliverable

A written report containing:

* **Implementation**: Complete test code with comments explaining each section
* **Test Output**: Console output showing:
  * Initial setup (validator groups)
  * Position changes during consensus
  * Final ledger acceptance
* **Analysis Table**:

  | Metric | Value |
  |--------|-------|
  | Total validators | 6 |
  | Consensus rounds needed | ? |
  | Position changes observed | ? |
  | Final transaction count | ? |
  | Tx{100} included? | YES/NO |
  | Tx{200} included? | YES/NO |
  | Tx{999} included? | YES/NO |

* **Questions Answered**:
  1. Why use 6 validators total? What happens with an odd number?
  2. Why add Tx{999} to everyone? What role does it play?
  3. How did the 50/50 split resolve?
  4. What determines which disputed transaction wins?

***

### Advanced Challenges (Optional)

1. **Uneven Splits**: Modify to create a 60/40 split and compare results
   ```cpp
   PeerGroup groupA = sim.createGroup(6);
   PeerGroup groupB = sim.createGroup(4);
   ```

2. **Network Partitions**: Add significant delays between groups
   ```cpp
   groupA.connect(groupB, round<milliseconds>(2.0 * parms.ledgerGRANULARITY));
   ```

3. **Priority Analysis**: Create scenarios where transaction fees affect dispute resolution

***

### Tips

* The consensus simulation framework (`csf`) provides a controlled environment for testing
* `sim.synchronized()` returns false if validators haven't agreed
* `BEAST_EXPECT()` is the test assertion macro
* Position changes often cascade—one validator switching can trigger others
* The avalanche mechanism (threshold changes over time) affects resolution speed

#### Learning Goals

By completing this homework, you should be able to:

* Write consensus tests using the csf framework
* Understand how disputes are detected and tracked
* Observe the iterative convergence process
* Explain how threshold adjustments help resolve deadlocks
* Predict consensus outcomes based on validator distribution
