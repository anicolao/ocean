# Remote Operator Trees compared with Kopia

> Historical comparison: this evaluates the earlier storage proposal. The current [Concord v1 design](concord-v1.md) uses encrypted streams and recipient keyboxes for readable access, and prepares recovery keys while deferring recovery workflows. Earlier tenant checks and access-grant recommendations below are superseded by that design and the [updated storage contract](remote-operator-trees.md).

Comparison baseline: this document evaluates the [original token-based proposal](https://github.com/anicolao/ocean/blob/bcfd49b13c1d9bd6e57d32736da3cc288379a2b9/design/remote-operator-trees.md). Its findings informed the [revised design](remote-operator-trees.md), [first Concord implementation](concord-v1.md), and [URL naming comparison](urls-vs-upspin-naming.md); statements about deferred identity describe that earlier baseline.

Design comparison, 16 October 2026

The [Remote Operator Trees proposal](remote-operator-trees.md) is an unimplemented storage and evaluation service. Kopia is an existing backup system with a reusable repository architecture. This comparison evaluates the proposed first version, not every capability imagined for Ocean, against Kopia's documented design. Recommendations and performance expectations below are design judgments, not benchmark results.

**Kopia is a strong baseline for storing and reconstructing data remotely. Operator Trees is worth pursuing if explicit, composable computations are useful objects in their own right.** With only `cat`, `concat`, and `slice`, the initial difference is narrower than “storage versus computation” suggests.

## The closest architectural match

Kopia layers content-addressed blocks over blob storage, groups encrypted blocks into packs, and indexes their locations. Its object layer represents larger objects through indirect records referencing constituent blocks. Label-addressed manifests describe snapshots and policies. The indirect object is already a reconstruction description: reading it resolves its parts. It is therefore a close analogue to a concatenation recipe, although the two formats and interfaces differ. [Kopia architecture](https://kopia.io/docs/advanced/architecture/)

| Question | Remote Operator Trees proposal | Kopia |
|---|---|---|
| What is the main abstraction? | Blobs, recipes, evaluation jobs, pins, and named references | Repository objects and snapshot manifests |
| How is reconstruction expressed? | Explicit, versioned byte operations composed by clients | Indirect objects linking stored pieces |
| What identifies a result? | SHA-256 of its complete output bytes, separate from its recipe | An object ID that may identify an indirect representation |
| What work is exposed remotely? | Submit a recipe and obtain a materialized result | Repository access and backup operations |

The Kopia object-ID distinctions in this table come from its [architecture documentation](https://kopia.io/docs/advanced/architecture/). In particular, a Kopia indirect object ID should not be treated as our proposed SHA-256 of the fully reconstructed byte stream.

### Remote does not distinguish the designs

Kopia can store repositories in remote services including S3-compatible storage, Azure, and Google Cloud Storage, as well as local storage. A remote Operator Trees server is not an advance merely because its data lives elsewhere. [Kopia repositories](https://kopia.io/docs/repositories/)

There are also two Kopia deployment models to distinguish. A client can access repository storage directly, or use a Repository Server that holds the underlying storage connection and mediates access using user credentials. The latter already addresses a substantial part of the thin-client motivation. Its documented interface is not the proposed recipe-job protocol. [Kopia Repository Server](https://kopia.io/docs/repository-server/)

## Pros of the Operator Trees design

1. **Recipes are explicit application objects.** A client can construct a slice of a concatenation, retain it, share its address within the tenant, and inspect exactly how it was defined. This gives Ocean an interface for composition without first translating the task into a filesystem snapshot.

2. **Computation and result have separate identities.** Different recipes yielding `CDEF` converge on one output address. The service can preserve the derivation while allowing clients to compare results independently of that derivation. This is a useful contract for future transformations, even though the first version does not select recipes automatically.

3. **Work can happen beside the stored inputs.** A phone can request a small derived result without fetching all its sources. This could reduce client bandwidth and CPU use; the benefit depends on server evaluation efficiency and whether the comparison client could already read just the needed data.

4. **Retention choices are exposed directly.** A client can pin a result alone or retain a recipe's complete dependency closure. That makes “keep the answer” and “keep enough to reproduce the answer” different, deliberate choices.

5. **The proposed authorization boundary is stricter.** Every dependency and cache hit must be authorized for the tenant; possession of an address is insufficient. Kopia's documented server defaults allow authenticated users to read objects whose IDs they know, while restricting snapshot and policy manifests. Our proposed isolation is an advantage if implemented and tested, not an achieved security property. [Kopia server access controls](https://kopia.io/docs/repository-server/)

## Cons and gaps in the Operator Trees design

1. **The first operations offer limited new storage capability.** The demonstration proves composition and remote execution, but assembling stored pieces is already familiar territory. A compelling PoC must show why naming and retaining the computation helps an application, rather than claim reconstruction itself as the innovation.

2. **It omits automatic splitting and compression.** Kopia splits files, checks chunk hashes for duplicates, then compresses and encrypts new chunks before packing them. Our proposal leaves upload boundaries to clients. Editing a large monolithic blob may therefore require uploading the whole replacement unless the client deliberately builds it from smaller parts. [Kopia compression pipeline](https://kopia.io/docs/advanced/compression/)

3. **The trusted server sees plaintext unless clients encrypt first.** With encrypted inputs, the current operators only manipulate ciphertext. By contrast, Kopia's direct-to-storage workflow encrypts file data and names before upload. That is a stronger starting point for confidential remote persistence. This comparison does not assume the same trust boundary for a separately operated Kopia Repository Server. [Kopia encryption behavior](https://kopia.io/docs/features/#user-controlled-end-to-end-encryption)

4. **Materialization can consume more storage and delay reads.** The draft publishes complete verified outputs before clients can download them. A concatenation may coexist with its source blobs, and nested recipes may create additional intermediates. A four-byte slice of a huge recipe is cheap only if the evaluator can avoid materializing unnecessary input; the draft does not require that optimization.

5. **A small output may retain a large dependency closure.** Pinning a slice currently pins its entire source graph. A four-byte answer could keep gigabytes alive. Pinning the answer alone avoids that cost but drops its reconstruction guarantee. The service should expose this tradeoff before accepting a pin.

6. **Execution adds operational obligations.** Cancellation, quotas, temporary files, job recovery, deterministic behavior, and publication races become storage-service concerns. Three bounded byte operators keep this manageable, but extending the operator set will expand both the attack surface and the compatibility burden.

7. **Its durability promise is deliberately weak.** Surviving a service restart does not address disk loss, missing dependencies, corruption, or provider failure. Kopia offers explicit structural verification and optional file-content verification; our acceptance criteria cover only a small part of that territory. [Kopia consistency checks](https://kopia.io/docs/advanced/consistency/)

8. **Portable raw hashes have a privacy cost.** SHA-256 of plaintext bytes is easy to verify across services, but a disclosed digest lets someone test guesses about the content. Access checks stop unauthorized retrieval, not inference from leaked identifiers. Encrypting before hashing changes the identity to that of the ciphertext; choosing a private address scheme would change the proposed interoperability contract.

## Pros of using Kopia as the starting point

1. **Useful storage behavior exists already.** Its chunking pipeline provides deduplication without requiring users to design their own section boundaries. That makes it a more practical baseline for real documents and large changing files. [Kopia compression pipeline](https://kopia.io/docs/advanced/compression/)

2. **It separates repository encryption from password handling.** Envelope encryption separates the repository passphrase from the keys used for stored content. Ocean's proposal has no comparable key-management design yet. [Kopia encryption design](https://kopia.io/docs/advanced/encryption/)

3. **Retention and reclamation have an operational model.** Kopia traces live snapshots, reclaims unreachable content, compacts packs, and coordinates maintenance ownership. Its safeguards account for concurrent clients and delayed visibility. These are concrete lessons for implementing our pins and staging leases. [Kopia maintenance](https://kopia.io/docs/advanced/maintenance/)

4. **There is a way to test whether data remains recoverable.** Kopia distinguishes checking repository structure from downloading, decrypting, and decompressing file contents for verification. Ocean should adopt that distinction: a reachable address does not prove that its bytes are readable. [Kopia consistency checks](https://kopia.io/docs/advanced/consistency/)

## Cons of treating Kopia as the Ocean substrate without another layer

1. **The recipe protocol would still need to be built.** Storing recipe JSON as data does not give Kopia the proposed operation semantics, evaluation records, job limits, or result publication rules. Reusing storage could save work, but it would not implement the central experiment.

2. **The retention model needs an adapter.** Ocean would need to connect its live recipe roots to whatever mechanism protects repository content from collection. A reference hidden inside an ordinary JSON blob must not be assumed to retain its target. An integration must prove that recipe dependencies survive maintenance and become reclaimable when released.

3. **Tenant isolation cannot be inferred from repository-server login.** The documented object access behavior differs from the proposal's per-tenant rules. Reuse would need an authorization layer, isolation through separate repositories, or another explicitly reviewed arrangement; changing a UI or hiding IDs would not establish that boundary.

4. **The two identifier schemes should remain separate.** An adapter should map Ocean addresses to storage handles, rather than expose repository IDs as if they were the proposed whole-output hashes. That mapping becomes additional durable metadata to back up and repair.

These are integration costs, not claims that Kopia cannot be extended. Its documented storage layers are useful prior art; whether to reuse its implementation requires a separate API, licensing, and dependency review.

## What the comparison suggests for our PoC

Keep the experiment centered on **explicit recipes, remote evaluation, and separate recipe/result identities**. Treat efficient physical storage as a separable concern. Neither a new storage format nor a custom server is itself the desired result.

Before implementation, I would add four requirements to the proposal:

1. **Expose retention cost.** Report both output size and retained dependency size, with shared data accounted for explicitly.
2. **Measure materialization cost.** Exercise a small slice of a large concatenation and record bytes read, temporary storage, latency, and client transfer. Compare with retrieving the same logical data through Kopia; use equivalent encryption and storage conditions.
3. **Specify the encryption boundary.** Decide whether the first deployment is intentionally a trusted plaintext execution service or whether private data must be client-encrypted. Do not imply both unrestricted server computation and provider-blind plaintext storage.
4. **Add recovery checks.** Test interrupted publication, missing inputs, corrupted stored bytes, and garbage collection racing with evaluation. Successful hashing at upload time is only the beginning of persistence.

**Recommendation:** use Kopia as the storage baseline and a possible implementation source, while testing the recipe service as the new architectural contribution. Do not decide to replace or embed Kopia until the PoC demonstrates a useful behavior beyond reconstruction and measures the cost of providing it.

This document proposes those follow-ups; it does not change the original design or claim that either system has been benchmarked here.

[Remote Operator Trees proposal](remote-operator-trees.md) · [Ocean vision](../VISION.md)
