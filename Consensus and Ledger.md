# Consensus and Ledger

### Module Overview

This module focuses on the heart of XRPL's distributed agreement: the **Consensus** mechanism that enables thousands of nodes to agree on a single ledger state, and the **Ledger** system that stores and manages the network's complete state history.

Every few seconds, validators across the globe must agree on which transactions to include in the next ledger. Without robust consensus, the network could fork. Without efficient ledger management, nodes couldn't synchronize or verify history.

Consensus and Ledger systems work together to provide:

* Distributed agreement without central authority
* Fast finality (3-5 seconds)
* Complete transaction history
* Efficient node synchronization
* Byzantine fault tolerance

***

### Explore the Topics

This module covers the complete architecture of Consensus and Ledger systems, from theoretical foundations to production implementations:

* **Consensus** – Understand the federated consensus model, UNL configuration, proposal exchange, dispute resolution, and the state machine that drives agreement.
* **Ledger** – Learn about ledger structure, data organization, immutability guarantees, and how state transitions are managed.
* **Integration** – See how consensus results flow into ledger creation and how the transaction lifecycle connects both systems.
* **Advanced Topics** – Explore acquisition mechanisms, validation quorums, error handling, and network synchronization.

Through these topics, you will gain both conceptual understanding and practical knowledge of how XRPL maintains consistent, verifiable state across a global network of independent nodes.

***

### What You Will Learn

By completing this module, you will be able to:

* Navigate Consensus and Ledger code confidently
* Trace the lifecycle of transactions from submission to finality
* Explain why federated consensus works without proof-of-work
* Understand the phases and modes of the consensus state machine
* Debug consensus and ledger synchronization issues
* Implement efficient state queries and understand validation
* Appreciate the engineering elegance enabling distributed consistency
* Apply concepts in real code exploration and practical exercises

***

#### 🤝 Consensus Fundamentals

**Understanding Distributed Agreement**

Explore the fundamental concepts behind XRPL's federated consensus model, including why it differs from proof-of-work and how trust is configured through UNLs.

**Key Topics**: Federated consensus, UNL, Byzantine tolerance, agreement thresholds\
**Codebase**: `src/xrpld/consensus/`

[Explore Consensus Fundamentals →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/consensus-fundamentals)

***

#### 🔄 Consensus Modes and Phases

**State Machine Operations**

Learn the different modes a node can operate in (proposing, observing, wrongLedger) and the phases of each consensus round (open, establish, accepted).

**Key Topics**: ConsensusMode, ConsensusPhase, state transitions, timer events\
**Codebase**: `src/xrpld/consensus/`

[Explore Consensus Modes and Phases →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/consensus-modes-phases)

***

#### ⏱️ Consensus Lifecycle

**Complete Round Progression**

Trace through the entire consensus lifecycle from round initiation through proposal exchange, dispute resolution, and ledger acceptance.

**Key Topics**: startRound, timerEntry, updateOurPositions, haveConsensus, onAccept\
**Codebase**: `src/xrpld/consensus/`, `src/xrpld/app/consensus/`

[Explore Consensus Lifecycle →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/consensus-lifecycle)

***

#### 📚 Ledger Architecture

**State Organization and Design**

Discover the architectural principles behind XRPL's ledger system, including sequential chains, Merkle trees, and multi-tier storage.

**Key Topics**: Ledger chain, hierarchical data, Merkle proofs, immutability\
**Codebase**: `src/xrpld/app/ledger/`

[Explore Ledger Architecture →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/ledger-architecture)

***

#### 🗄️ Ledger Data Structures

**Core Classes and Interfaces**

Dive deep into the Ledger class, LedgerInfo, LedgerHolder, LedgerHistory, and how they work together to manage ledger state.

**Key Topics**: Ledger class, SHAMap integration, Rules, immutability enforcement\
**Codebase**: `src/xrpld/app/ledger/`

[Explore Ledger Data Structures →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/ledger-data-structures)

***

#### 🔍 Ledger Acquisition and Validation

**Network Synchronization**

Learn how nodes acquire missing ledgers from peers, validate them through quorum checks, and maintain a consistent chain.

**Key Topics**: LedgerMaster, InboundLedgers, validation quorum, gap filling\
**Codebase**: `src/xrpld/app/ledger/`

[Explore Ledger Acquisition and Validation →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/ledger-acquisition)

***

#### 📡 RPC and Peer Transaction Lifecycle

**Transaction Processing Paths**

Follow transactions from submission (RPC or peer relay) through validation, queue management, application, and network propagation.

**Key Topics**: doSubmit, processTransaction, TxQ, preflight/preclaim/doApply, relay\
**Codebase**: `src/xrpld/rpc/`, `src/xrpld/overlay/`

[Explore RPC and Peer Transaction Lifecycle →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/transaction-lifecycle)

***

### 📚 Appendices

[Explore Codebase Navigation Guide →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/appendices/navigation)

[Explore Configuration Reference →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/appendices/configuration)

[Explore Debugging and Development Tools →](https://docs.xrpl-commons.org/core-dev-bootcamp/module08/appendices/debugging)

### Helpful Resources

* Rippled Repository: [github.com/XRPLF/rippled](https://github.com/XRPLF/rippled)
* Core Dev Bootcamp Docs: [docs.xrpl-commons.org/core-dev-bootcamp](https://docs.xrpl-commons.org/core-dev-bootcamp/)
* Consensus Code: `src/xrpld/consensus/`
* Ledger Code: `src/xrpld/app/ledger/`
* NetworkOPs: `src/xrpld/app/misc/`

## Knowledge Check

**Review and Reinforce Your Understanding**

Before moving on, take a few minutes to review key concepts from this module.\
From consensus phases and proposal handling to ledger acquisition and transaction processing, this short quiz will help you confirm your understanding of XRPL's distributed agreement and state management.

[Ready? Let's Quiz →](https://xrpl.at/cdbc-quiz08)

## Questions

If you have any questions about the homework or would like us to review your work, feel free to contact us.

[Submit Feedback →](https://docs.google.com/forms/d/e/1FAIpQLSe7qIbqepUKGVJxmWFHv0rs1zsBAzk8G6y5-bfz0Yd5rqW_5A/viewform)

***

➡️ **Next Module**: [Continue to next module →](https://docs.xrpl-commons.org/core-dev-bootcamp/module09)
