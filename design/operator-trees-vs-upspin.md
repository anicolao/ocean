# Remote Operator Trees compared with Upspin

Comparison baseline: this document evaluates the [original token-based proposal](https://github.com/anicolao/ocean/blob/bcfd49b13c1d9bd6e57d32736da3cc288379a2b9/design/remote-operator-trees.md). Its findings informed the [revised design](remote-operator-trees.md), [first Concord implementation](concord-v1.md), and [URL naming comparison](urls-vs-upspin-naming.md); statements about deferred identity describe that earlier baseline.

Design comparison, 16 October 2026

The [Remote Operator Trees proposal](remote-operator-trees.md) describes a small, unimplemented remote storage and evaluation service. Upspin describes a network for naming, storing, and sharing information across independently operated services. This comparison uses its official design documents and published interfaces; it does not assess current deployment scale, service availability, or implementation performance.

**Upspin is particularly useful prior art for Ocean's identity, naming, and sharing layers. Operator Trees explores a different boundary: making reproducible computations explicit objects that a remote service can evaluate.** Neither design, by itself, supplies the whole Ocean vision.

## How the architectures line up

Upspin separates three services: a KeyServer associates users with public keys and directory locations, a DirServer resolves hierarchical names, and a StoreServer holds data blocks. To read a name, the client discovers the directory, obtains block locations, fetches the blocks, and reconstructs the content. This already separates a person's namespace from the machines storing their data. [Upspin architecture](https://upspin.io/doc/arch.html)

| Concern | Remote Operator Trees proposal | Upspin |
|---|---|---|
| User-facing address | Tenant-local named reference | User-qualified hierarchical path |
| Stored data | Immutable blobs and recipes | Blocks referenced by directory entries |
| Identity | Tenant bearer token; Concord deferred | User name and public/private keys |
| Computation | Explicit recipe evaluation jobs | Client reconstruction and service-defined behavior |
| Service layout | One trusted provider in the PoC | Separate key, directory, and storage roles |

The Upspin side of this mapping follows its [architecture](https://upspin.io/doc/arch.html). These are correspondences, not interchangeable protocols or equivalent guarantees.

### A common example

Suppose Alice keeps a document assembled from two sections and wants Bob to read it. In Operator Trees, Alice can create a concatenation recipe, pin its dependencies, and ask the server for a materialized result. The current proposal supports access within one tenant; sharing with Bob as an independent identity remains undefined.

In Upspin, Alice can publish the document under a name such as `alice@example.com/notes/design` and grant Bob read access. The access model defines rights through `Access` files and groups, including separate read, write, create, list, and delete rights. Encryption adds another requirement: Bob must have a usable wrapped content key. [Upspin access control](https://upspin.io/doc/access_control.html)

The practical distinction is what the caller can express. Our recipe records “join these specific immutable inputs.” An Upspin name tells the caller where to obtain an item under a user's namespace. Publishing recipe JSON at that name would preserve its bytes, but would not by itself define how any server should execute it.

## Pros of the Operator Trees design

1. **Reproducibility is explicit.** A recipe names immutable inputs and versioned operations. It can be retained and evaluated again without resolving a mutable pathname. This is useful when an application needs the exact earlier computation, rather than the latest item at a name.

2. **The derivation and result have distinct identities.** Two recipes that yield identical bytes share an output address while preserving different recipe addresses. That supports inspecting provenance and comparing results independently of the steps used to obtain them.

3. **Evaluation can run near the data.** A small client can submit a slice of a composition and retrieve the answer. Whether this saves bandwidth or time depends on evaluation strategy and the comparison workload; it is not automatically better than fetching selected source blocks.

4. **Retention follows computational dependencies.** Pinning a recipe explicitly preserves its source closure. This gives an application a way to request future reconstruction without making every intermediate result permanent.

5. **The experiment is narrower.** A single service and three byte operators can test remote composition before adopting a global namespace or identity infrastructure. That reduces initial scope, although it also postpones much of what would make the system useful between people.

## Cons and gaps in the Operator Trees design

1. **It has no real person-to-person sharing model yet.** Tenant tokens do not define discovery, attestations, key rotation, or authority delegated between independently identified users. Upspin's user-qualified namespace and key lookup provide an existing model to study. [Upspin overview](https://upspin.io/doc/overview.html)

2. **It combines roles that may need different trust boundaries.** The first server stores data, resolves local names, authorizes access, and evaluates recipes. Separating these roles would make it easier to place private naming, bulk storage, and authorized computation with different operators. That is a design inference from Upspin's service decomposition, not a claim that decomposition alone makes a system secure.

3. **Confidential plaintext evaluation remains unresolved.** Upspin's default encrypted packing performs encryption and decryption on clients and signs files; servers do not need the plaintext contents. Our proposal either exposes plaintext to its evaluator or operates on ciphertext, where a byte slice generally has no useful relationship to a plaintext slice. [Upspin encryption FAQ](https://upspin.io/doc/faq.html#security)

4. **It lacks namespace change notifications.** A client can read and conditionally replace a named reference, but the proposed API has no subscription mechanism. Upspin's directory interface includes sequence numbers and `Watch`; these are useful models for making multiple devices react to changes. Its conditional `Put` behavior also shows that detecting stale writes is not unique to our proposal. [Upspin interfaces](https://pkg.go.dev/upspin.io/upspin#DirServer)

5. **It is less general about where information comes from.** Our recipes require already stored inputs and prohibit external reads. Upspin's overview explicitly discusses services such as a camera whose named image changes on each read. Those services offer useful live data, though their outputs would need to be captured as immutable inputs before entering a reproducible recipe. [Upspin service examples](https://upspin.io/doc/overview.html#examples)

6. **Its operational promises remain to be proven.** Pinning, leases, crash recovery, tenant checks, and quotas are specified behavior, not tested implementation. Keeping a dependency graph reachable does not repair corrupted bytes, preserve a failed disk, or prove that a remote evaluator executed an operation correctly.

## Pros of Upspin as an architectural foundation

1. **It gives people a shared naming model.** Users can refer to information owned by someone else without sharing a local filesystem or choosing that person's storage provider. This is directly relevant to Ocean objects following people across machines. [Upspin overview](https://upspin.io/doc/overview.html)

2. **It connects sharing to cryptographic readers.** The security design uses a randomly chosen content key, wraps it for authorized readers, and signs the content. This is more concrete than simply saying that Ocean will encrypt its storage. It also identifies which work belongs on a trusted client. [Upspin security](https://upspin.io/doc/security.html)

3. **It separates naming policy from storage mechanics.** The StoreServer interface provides retrieval, storage, and deletion without interpreting the data. That is a useful boundary beneath an optional evaluator. A storage reference is an implementation-defined string; it is not universally our proposed whole-output SHA-256. [Upspin StoreServer interface](https://pkg.go.dev/upspin.io/upspin#StoreServer)

4. **It distinguishes deleting names from reclaiming shared storage.** Its audit tooling compares stored blocks with directory references, and warns that deletion must account for other trees and snapshots. That is a practical lesson for recipe dependency retention: every live root matters. [Upspin audit and deletion commands](https://pkg.go.dev/upspin.io/cmd/upspin)

## Cons and limits of using Upspin directly for Ocean

1. **Its global key service is a dependency, not Concord's self-created identities and attestation-based trust model.** The documented ecosystem uses a central key server and describes a tamper-evident log. That offers a concrete discovery mechanism, but adopting it would be a distinct identity decision rather than implementing decentralized trust automatically. [Upspin key management](https://upspin.io/doc/security.html#key-management)

2. **Content privacy does not hide all metadata.** The directory service knows plaintext paths and sharing information. Choosing where to run it is therefore a privacy decision. Ocean should not equate encrypted payloads with an opaque object graph. [Upspin security](https://upspin.io/doc/security.html)

3. **Encryption affects deduplication.** Independent writes of identical plaintext generally use different encryption keys in the documented design and can occupy separate storage. This is a privacy tradeoff, not a flaw to remove casually; globally comparable plaintext identities can leak equality information. [Upspin security](https://upspin.io/doc/security.html)

4. **Access changes and cryptographic access are different operations.** Updating an `Access` file can require updating reader key wraps, with client cooperation. Revoking a directory permission cannot make a recipient forget previously obtained plaintext or keys. The latter is a general limit on sharing, and the former is an explicit part of Upspin's access design. [Upspin access control](https://upspin.io/doc/access_control.html)

5. **It does not supply the proposed recipe contract.** A custom Upspin service could expose computed data, but Ocean would still need to specify operation versions, immutable input bindings, job limits, cancellation, output verification, and recipe retention. Service extensibility is useful infrastructure, not an existing implementation of that contract.

6. **A path is not an enduring Ocean object identity.** If an object is renamed, moved, or exposed through several views, Ocean still needs to say what preserves its identity and history. A namespace adapter should map names to that identity rather than silently treat the current path as the object itself. This is an integration requirement, not a claim that Upspin cannot host such a layer.

## What to borrow and what to keep separate

The clearest architectural lesson is to distinguish four responsibilities: **identity and authority, naming, byte storage, and evaluation**. They can share one process in the PoC while retaining separate contracts. Upspin provides particularly relevant ideas for the first three; Operator Trees makes the fourth explicit.

I would carry these questions into the next revision of our proposal:

1. **How does a name bind to an immutable object?** Keep mutable names outside recipes. Resolve and record an exact input before evaluation, including when the source is a live service.
2. **Who can decrypt for an evaluation?** Begin with an explicitly trusted evaluator or encrypted storage with client-side computation. Giving a worker plaintext access must be a deliberate grant, not an accidental consequence of granting storage access.
3. **How do changes reach other clients?** Add a versioned notification mechanism for named references, with a way to recover after a disconnected client misses events.
4. **What happens when data crosses tenants or providers?** Define identity, authorization, and retention commitments before offering sharing. Hash equality alone establishes none of them.
5. **Which roots protect recipe inputs?** If an Upspin-backed store is explored, make recipes visible to retention accounting. A hash written inside an opaque file is not automatically a live storage reference.

**Recommendation:** treat Upspin as a stronger reference for Ocean's distributed namespace and secure sharing than for its computation model. Preserve the small Operator Trees experiment, but structure it so naming and identity can evolve independently of the evaluator. An integration spike should prove one encrypted shared input, an explicitly authorized computation, and correct retention before attempting a general adapter.

These are proposed follow-ups, not changes to the original design. No benchmark or deployment test is included here.

[Remote Operator Trees proposal](remote-operator-trees.md) · [Kopia comparison](operator-trees-vs-kopia.md) · [Ocean vision](../VISION.md)
