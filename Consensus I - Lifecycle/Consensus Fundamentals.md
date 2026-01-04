# Consensus Fundamentals

[← Back to Consensus and Ledger Architecture](https://docs.xrpl-commons.org/core-dev-bootcamp/module08)

***

### Introduction

The XRP Ledger consensus mechanism is the heart of how distributed nodes agree on a single, canonical view of the network state. Unlike proof-of-work systems that rely on computational puzzles, or proof-of-stake systems that rely on economic incentives, XRPL uses a unique consensus protocol based on trusted validators reaching agreement through iterative voting.

This chapter introduces the fundamental concepts that underpin the consensus process, providing the foundation for understanding how thousands of independent nodes create identical ledgers without central coordination.

### The Consensus Problem

**The Challenge:**

In a distributed system with no central authority, how do nodes agree on:
- Which transactions are valid?
- What order should transactions be processed?
- What is the resulting state of all accounts?

**Traditional Solutions and Their Trade-offs:**

| Approach | Mechanism | Trade-off |
| --- | --- | --- |
| Proof of Work | Computational puzzles | High energy cost, slow finality |
| Proof of Stake | Economic staking | Wealth concentration, nothing-at-stake |
| PBFT | Voting rounds | Limited scalability |
| XRPL Consensus | Trusted validators + iterative agreement | Requires UNL overlap |

### XRPL's Approach: Federated Consensus

The XRPL consensus protocol achieves agreement through a federated model:

```
                    Validator Network

    [Validator A] ←→ [Validator B] ←→ [Validator C]
          ↓               ↓               ↓
       Proposal       Proposal        Proposal
          ↓               ↓               ↓
    ┌─────────────────────────────────────────────────────┐
    │     Iterative Voting Process (Establish Phase)      │
    │                                                     │
    │      Iteration 1: Share initial positions           │
    │      Iteration 2: Adjust based on peers             │
    │      Iteration 3: Continue until supermajority      │
    └─────────────────────────────────────────────────────┘
                         ↓
              80% agree on same transaction set
                         ↓
                  Consensus reached
                         ↓
                   New Ledger created
```

**Key Principles:**

1. **Trust is configurable**: Each node chooses which validators to trust (Unique Node List - UNL)
2. **Iterative convergence**: Validators adjust positions based on peer input
3. **Avalanche voting**: Dynamic thresholds (50% → 65% → 70% → 95%) help validators converge on which transactions to include
4. **Final consensus**: 80% of validators must agree on the complete transaction set (same hash) to declare consensus
5. **Fast finality**: Ledgers close in 3-5 seconds on average

### The Consensus State Machine

The consensus process is implemented as a template-based state machine in the codebase:

```cpp
// Core consensus engine (Consensus.h)
template <typename Adaptor>
class Consensus {
    // Current phase of consensus
    ConsensusPhase phase_;

    // Operating mode of this node
    ConsensusMode mode_;

    // Timing for this round
    ConsensusTimer timer_;

    // Peer proposals and disputes
    std::map<NodeID, ConsensusProposal> currPeerPositions_;
    std::map<TxID, DisputedTx> disputes_;
};
```

**Design Properties:**

- **Generic Architecture**: Template-based design allows flexibility
- **Adaptor Pattern**: Integrates with different ledger and transaction types
- **State Isolation**: Each round maintains independent state
- **Event-Driven**: Timer events drive phase transitions

### Unique Node List (UNL)

The UNL is the set of validators a node trusts for consensus:

```
Node's Perspective:

    ┌─────────────────────────────────┐
    │         My Trusted UNL          │
    │                                 │
    │  [Validator 1] - Ripple         │
    │  [Validator 2] - Exchange A     │
    │  [Validator 3] - University B   │
    │  [Validator 4] - Company C      │
    │  ...                            │
    └─────────────────────────────────┘
              ↓
    Only consider proposals from these validators
    when determining consensus
```

**UNL Requirements:**

- **Overlap**: For network-wide agreement, UNLs must overlap sufficiently
- **Diversity**: Mix of organizations prevents single points of failure
- **Threshold**: Minimum ~80% overlap between any two UNLs recommended

### Consensus vs. Validation

**Important Distinction:**

| Consensus | Validation |
| --- | --- |
| Agreement on transaction set | Cryptographic endorsement of ledger |
| Happens during ledger close | Happens after ledger is built |
| Determines what's in the ledger | Confirms ledger is correct |
| Internal process | Published to network |

**Flow:**

```
Transactions → [Consensus] → Agreed Set → [Build Ledger] → [Validation] → Validated Ledger
```

### Byzantine Fault Tolerance

XRPL consensus tolerates Byzantine (malicious or faulty) validators:

**Tolerance Threshold:**

```
For UNL of size n:
  - Can tolerate up to ⌊n/5⌋ Byzantine validators (less than 20%)
  - Requires 80% honest agreement
  - Safety guaranteed with <20% Byzantine
```

**Attack Resistance:**

- **Sybil Attacks**: UNL selection limits attacker influence
- **Denial of Service**: Multiple validators ensure availability
- **Equivocation**: Proposals are signed and tracked
- **Censorship**: Requires controlling >20% of UNL

### Consensus Parameters

Key parameters that control consensus behavior:

```cpp
struct ConsensusParms {
      // Minimum consensus percentage required
      static constexpr int minCONSENSUS_PCT = 80;

      // Close time consensus threshold (added)
      static constexpr int avCT_CONSENSUS_PCT = 75;

      // Minimum time before consensus can be reached
      std::chrono::milliseconds ledgerMIN_CONSENSUS{1950};

      // Maximum time before consensus times out
      std::chrono::seconds ledgerMAX_CONSENSUS{15};

      // Avalanche state machine thresholds
      std::map<AvalancheState, int> avalancheCutoffs;
  };
```

**Parameter Impact:**

| Parameter | Low Value | High Value |
| --- | --- | --- |
| minCONSENSUS_PCT | Faster but less secure | Slower but more secure |
| ledgerMIN_CONSENSUS | Faster finality | More time for propagation |
| ledgerMAX_CONSENSUS | Potential stalls | Eventual termination |

### The Avalanche Mechanism

  XRPL's avalanche mechanism uses **increasing thresholds** to force convergence on disputed transactions:

  **Threshold Progression:**
```
  Time:     0% ────→ 50% ────→ 85% ────→ 200%
  State:    init     mid       late      stuck
  Threshold: 50%     65%       70%       95%
```
  **How It Works:**

  - **Early (50% threshold)**: Easy to include transactions → rapid exploration
  - **Mid (65% threshold)**: Harder to change votes → stabilization begins
  - **Late (70% threshold)**: Very hard to change → forced convergence
  - **Stuck (95% threshold)**: Near-impossible to change → lock-in

  **Key Point:** Thresholds **rise** over time, making it progressively harder to dissent from the majority. This creates an "avalanche effect" - once a majority position emerges, validators are forced to converge to it.

  **Purpose:**
  1. Prevent endless flip-flopping of votes
  2. Force commitment to majority view
  3. Resist Byzantine manipulation
  4. Guarantee convergence in finite time

  **Analogy:** Like a real avalanche, once momentum builds (majority emerges), the increasing thresholds make it impossible to stop the convergence toward that position.


### Integration with the Ledger

Consensus doesn't operate in isolation—it's tightly integrated with ledger management:

```
                    Application Layer
                          ↓
    ┌─────────────────────────────────────────┐
    │              NetworkOPs                 │
    │   (Coordinates consensus + ledger)      │
    └─────────────────────────────────────────┘
                          ↓
    ┌─────────────────────────────────────────┐
    │             RCLConsensus                │
    │      (XRPL-specific consensus)          │
    └─────────────────────────────────────────┘
                          ↓
    ┌─────────────────────────────────────────┐
    │          Generic Consensus<T>           │
    │     (Template-based state machine)      │
    └─────────────────────────────────────────┘
                          ↓
    ┌─────────────────────────────────────────┐
    │            LedgerMaster                 │
    │   (Manages ledger state and history)    │
    └─────────────────────────────────────────┘
```

### Why This Design Matters

Understanding consensus fundamentals is essential because:

1. **Transaction Finality**: Consensus determines when transactions are final
2. **Network Security**: The mechanism protects against attacks
3. **Performance**: Parameters affect throughput and latency
4. **Debugging**: Many issues trace back to consensus behavior
5. **Protocol Evolution**: Changes require deep understanding

### Summary

**Key Concepts:**

1. **Federated Consensus**: Trust-based agreement without central authority
2. **UNL**: Configurable set of trusted validators
3. **Iterative Voting**: Validators adjust positions based on peer input
4. **Avalanche Voting**: Rising thresholds (50%→95%) force convergence on disputed transactions
5. **Supermajority**: 80%+ validators must agree on the complete transaction set (same hash) for finalization
6. **Byzantine Tolerance**: Resists up to 20% malicious or faulty validators

**Design Properties:**

- Fast finality (3-5 seconds)
- Energy efficient (no mining)
- Configurable trust model
- Provable safety guarantees

In the next chapter, we'll explore the specific modes and phases that define the consensus lifecycle.
