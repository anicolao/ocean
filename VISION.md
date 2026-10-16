# Ocean
### A Vision for the Future of Personal Computing

## 1. The Computing Experience

Ocean is a computing environment organized around **people, their information, and their intentions**, rather than applications, files, devices, or services.

A person interacts with Ocean through conversation, direct manipulation, gestures, or any other suitable interface. They work with documents, drawings, conversations, simulations, and other meaningful objects without needing to choose particular applications to manipulate them.

An intelligent agent understands the user's goals, discovers available capabilities, and composes them to accomplish tasks. The agent is not required for ordinary interaction: users can directly manipulate objects with immediate, predictable responses.

Information belongs to its owner, persists independently of the devices used to access it, and can be shared securely with others. A person's computing environment follows them across devices, adapting its presentation and interaction methods to the capabilities of each.

**The fundamental promise of Ocean is that people express what they want to accomplish, while the system determines how to accomplish it.**

## 2. Persistent Objects

The fundamental unit of information in Ocean is a **persistent, identifiable object**.

An object has an enduring identity independent of its current contents, representation, location, or implementation.

Its state is defined by a history of meaningful operations. Rather than merely storing the latest result, Ocean preserves sufficient information to reconstruct how the object came to be.

Objects can be examined, modified, combined, shared, and presented in multiple ways. Different interfaces can operate on the same underlying object without converting it into application-specific formats.

Objects and their histories are immutable at the storage level. Changes create new states, while enduring identities reference the appropriate current state.

This provides intrinsic versioning, historical inspection, reproducibility, and the ability to reinterpret past operations when their definitions improve.

## 3. Composable Interaction and Presentation

Ocean has no fundamental concept of an application.

Instead, an interactive environment is assembled from **composable operators** that transform information and events.

### Input

Input devices produce streams of events that pass through successive transformations:

**Physical input → normalized events → gestures → semantic intentions → object operations**

Operators provide capabilities such as coordinate transformation, snapping, gesture recognition, selection, text editing, and geometric constraint solving.

Each operator has a defined interface and can be independently selected, replaced, or composed with others.

The same semantic intention may originate from a mouse, stylus, keyboard, voice command, or autonomous agent.

### Output

Object states are transformed through complementary presentation operators:

**Object state → semantic view → layout → rendering → sensory output**

Views expose structured semantic information, not merely pixels, enabling interaction operators to understand and manipulate the objects being presented.

Input and output form a continuous feedback loop. Operators may maintain explicit state, observe context, and participate in reactive computation graphs.

Capabilities such as spell checking, geometric constraints, accessibility transformations, and alternative visualizations are independently composable rather than embedded in particular applications.

The result is an environment in which **interfaces are assembled around information and tasks**, not predefined software products.

## 4. The Intelligent Intermediary

Ocean includes a personal intelligent agent that translates human intentions into structured, executable operations.

The agent understands the available objects, capabilities, specifications, permissions, and execution resources. It can construct or modify interaction environments, orchestrate complex work, and explain the consequences of proposed actions.

The agent operates at the highest meaningful semantic level. It manipulates objects and invokes capabilities directly rather than imitating physical interactions when structured operations are available.

Ocean separates interpretation from execution:

- **Intelligence** resolves ambiguous intentions, plans work, and selects capabilities.
- **Deterministic operators** perform well-defined transformations.
- **The persistent object model** records the resulting changes and their provenance.

The agent is a participant in the same computational environment as the human, governed by explicit capabilities and authority.

Ocean is fearless by design: changes are inspectable, consequences can be evaluated, and recoverable state transitions can be reversed. Irreversible external actions are distinguished explicitly from reproducible computation.

## 5. Operations and Their Meaning

All Ocean computation is expressed through **operations with explicit behavioral specifications**.

An operation is identified by what it is intended to accomplish, independently of any particular implementation.

Specifications consist of executable tests, formal constraints, proofs, or other verifiable descriptions of expected behavior. Immutable specification versions have content-derived identities.

Multiple implementations may satisfy the same specification. Ocean selects implementations according to correctness evidence, trust, compatibility, performance, resource requirements, and user preferences.

Specifications evolve as human understanding improves.

When a specification is refined, Ocean can identify implementations that satisfy the new definition and determine which prior computations depend on the affected behavior.

Those computations can be reevaluated, producing corrected interpretations and results without destroying their original histories.

This creates a **correctness ratchet**: a system in which the formal definitions of intended behavior become progressively more precise and implementations can be improved without changing the identity of the capabilities they provide.

## 6. Operator Graphs and Computational Storage

Ocean represents computation through immutable, content-addressed **operator graphs**.

An operator graph declaratively describes how an output is constructed from inputs through a composition of operations.

Evaluation may produce data, transformed event streams, object states, or structured views.

Graphs explicitly represent computational dependencies, permitting:

- Incremental reevaluation and caching.
- Parallel and distributed execution.
- Reproducible results and historical reconstruction.
- Dependency-aware invalidation when specifications change.
- Alternative implementations and execution strategies.

Ocean distinguishes three identities:

**Specification identity** describes intended behavior.

**Recipe identity** describes a particular computation.

**Content identity** describes the resulting information.

Multiple recipes may produce identical content. Ocean can discover alternative recipes and select among them according to availability, execution time, bandwidth, cost, and trust.

The execution system is declarative: it determines what must be produced, resolves dependencies, selects compatible implementations, and obtains results through computation or retrieval.

Programs, configurations, and execution environments are themselves represented as reproducible computational objects.

## 7. Sovereign Identity, Trust, and Authority

Every participant in Ocean possesses a persistent cryptographic identity, independent of any provider or device.

Identity is established through locally controlled cryptographic credentials, with explicit mechanisms for key rotation, recovery, and revocation.

Anyone can create an identity and begin using their own or a willing provider's storage without an invitation or endorsement. Participants choose which identity claims and evidence to disclose; others may sign attestations vouching for specific claims. These attestations form a decentralized trust graph that observers interpret for themselves, separately from grants of authority over information and operations.

Access to information and authority to modify it are distinct.

Information is encrypted, with cryptographic access granted through recipient-specific key distribution.

Authority over objects and operations is established through signed capabilities and verifiable histories.

Capabilities can grant narrowly scoped rights to read information, propose modifications, commit operations, execute computations, or delegate authority.

Humans, agents, and services participate under the same authority model.

**An object's identity, ownership, and history are independent of the machines and organizations that store or process it.**

## 8. Distributed Persistence and Execution

Ocean operates across a decentralized network of storage and computation providers.

Encrypted information is stored in content-addressable structures and retrieved on demand.

Immutable content, computational recipes, and mutable references to current object states are distinct mechanisms.

Storage providers need not understand or possess the keys required to interpret the information they hold.

Content can be replicated, cached, transferred, and reconstructed through alternative recipes. Availability is maintained through explicit replication and retention mechanisms.

Computations may execute locally or remotely according to available resources, latency, privacy, cost, and authority.

Shared objects support concurrent, offline operation through causally ordered histories and object-appropriate conflict resolution.

The same objects and capabilities remain available as users move between devices or execution environments.

No individual server, operating-system installation, storage provider, or computing service is essential to the continuity of a user's digital life.

## 9. A Unified Computational World

Ocean's architecture can be summarized as a hierarchy of responsibilities:

| Layer | Responsibility |
|---|---|
| **Human experience** | Direct manipulation, conversation, and expression of intent |
| **Intelligent intermediary** | Interpretation, planning, capability discovery, and composition |
| **Interaction and presentation** | Transformation of events and object states into interactive experiences |
| **Persistent object model** | Enduring identities, operation histories, and shared state |
| **Semantic operations** | Behavioral specifications, implementation equivalence, and refinement |
| **Computational engine** | Operator graphs, dependency resolution, evaluation, and caching |
| **Distributed substrate** | Encrypted storage, execution resources, replication, and synchronization |
| **Identity and authority** | Cryptographic ownership, trust, capabilities, and delegation |

Identity and authority govern every layer. The persistent object model connects human intentions to reproducible computation.

The architecture does not privilege a particular machine, programming language, interface, storage provider, or implementation.

## 10. The End State

In Ocean, a person does not need to think in terms of installing applications, managing files, configuring operating systems, or choosing service providers.

They work with information directly.

When they need a capability, Ocean discovers or constructs an appropriate composition of operations.

When they encounter a new device, their environment adapts to it.

When they collaborate, they share objects and delegate authority without surrendering ownership.

When software improves, their environment can benefit without losing the meaning or history of their work.

When errors are discovered, affected computations can be identified, corrected, and reproduced.

When infrastructure changes, their information and identity endure.

**Ocean is a persistent, sovereign, self-improving computational world in which human intention is the organizing principle and software is a replaceable means of realizing it.**

---

# References and Prior Art

The Ocean vision synthesizes several architectural traditions and earlier concepts. These are influences and analogues, not claims that existing systems implement Ocean in full.

| Concept or system | Relationship to Ocean |
|---|---|
| **Concord** | Self-created cryptographic identities, chosen disclosures, signed identity attestations, encrypted decentralized storage, and distributed access control. |
| **Operator Trees** | Composable computation, content-addressed representations, alternative reconstruction recipes, and cost-aware evaluation. |
| **vOS** | Persistent functional state, operation histories, specification-defined behavior, correctness refinement, and dependency-aware reevaluation. |
| **Composable GUI concept** | Independently replaceable input transformations, semantic intentions, reducers, and presentation operators. |
| **Nixie** | The intelligent intermediary between human intention and a declarative computing environment. |
| **[Nix / NixOS](https://nixos.org/)** | Derivations, reproducible execution environments, immutable dependency graphs, build substitution, and rollback. |
| **[Plan 9](https://9p.io/plan9/)** | Uniform access to distributed resources through composable namespaces. |
| **[Symbolics Genera](https://en.wikipedia.org/wiki/Symbolics)** and Lisp machines | Integrated object environments, introspection, live modification, and deeply integrated user interfaces. |
| **[TUNES](https://tunes.org/)** | Reflective, persistent, distributed computing; implementation-independent abstractions and capability security. |
| **[Secure Scuttlebutt](https://ssbc.github.io/scuttlebutt-protocol-guide/)** | Cryptographic identities, decentralized social relationships, and replicated information histories. |
| **[IPFS](https://docs.ipfs.tech/)** | Content addressing and decentralized content retrieval. |
| **[age](https://age-encryption.org/) / [SOPS](https://getsops.io/)** | Recipient-based encryption and structured management of encrypted information. |
| **[Event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)** | Object state reconstructed from persistent histories of operations. |
| **[CRDTs](https://crdt.tech/)** | Convergent shared state and offline collaboration without universal serialization. |
| **[Functional reactive programming](https://en.wikipedia.org/wiki/Functional_reactive_programming)** | Composable transformations of events and time-varying information. |
| **[Bidirectional transformations](https://doi.org/10.1145/1244651.1244671)** | Maintaining coherent relationships between editable views and underlying data. |
| **[Sisyphus](https://ilyasergey.net/assets/pdf/papers/sisyphus-pldi23.pdf)** | Reusing and repairing correctness proofs as software implementations evolve. |

### The distinguishing synthesis

Individual parts of Ocean have precedents. The proposed contribution is their integration into a common computational model:

**Persistent objects provide continuity. Cryptographic identities provide sovereignty. Operator graphs provide reproducible computation. Behavioral specifications provide meaning. Composable interfaces provide interaction. Intelligence connects human intention to the entire system.**

Together, these replace the application-centered computer with an **intention-centered computing environment**.
