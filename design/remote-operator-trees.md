# Remote Operator Trees

A proposed first storage substrate for Ocean, with Concord identity

Revised after the Kopia, Upspin, and URL naming comparisons. The initial implementation includes the [Concord identity and authority service](concord-v1.md) and follows the [naming policy](urls-vs-upspin-naming.md).

## The idea

An Operator Trees server stores both bytes and descriptions of how to produce bytes. A client can upload data, compose it into a recipe, and ask the server to return the result. The result can be cached, retrieved later, or rebuilt from its inputs.

The server is the durable home for the data. A client needs its Concord device key, verified identity state, service locations, and object addresses, but no local object database or permanent local data copy. Two clients can work with the same stored objects. Local caching is optional.

This first version tests one proposition: **a remote store can preserve information as either materialized bytes or a reproducible recipe, while presenting the same verified result to its clients.** It is a proposed design, not an implemented service.

## A concrete example

Imagine a document assembled from independently stored sections. Upload the sections once, then store a recipe that concatenates them in order. Reading the recipe produces the document. Editing one section means uploading that section and making a new recipe; the other sections remain shared.

For a tiny example, upload `ABCD` and `EFGH`. Create a concatenation recipe that produces `ABCDEFGH`, then a slice recipe that takes four bytes starting at offset two. Evaluating the second recipe returns `CDEF`.

Another computer can request that same recipe and receive `CDEF` without downloading the source sections itself. If the server discards its cached result, it can reconstruct it from the retained recipe and inputs.

This is the basic relationship we want to exercise before adding richer operators, distributed providers, or an intelligent agent.

## What the service stores

| Entity | Meaning | Changes over time |
|---|---|---|
| Blob | An exact sequence of bytes | Never |
| Recipe | An operation and its ordered input references | Never |
| Evaluation record | A verified relationship between a recipe and its output blob | New records may be added |
| Pin | A principal's retention commitment for a blob or recipe | Explicitly created or removed |
| Named reference | A convenient name pointing to a pinned blob or recipe | Updated with a version check |

Recipes form trees conceptually, but shared inputs make the stored structure a directed acyclic graph. References name already stored blobs or recipes. There are no embedded mutable names or URLs in a recipe.

### Two different addresses

A blob address is `blob:sha256:<hex>`, where the digest covers exactly the stored bytes. A recipe address is `recipe:sha256:<hex>`, where the digest covers the UTF-8 bytes of `ocean-recipe-v1\n` followed by the canonical recipe encoding below. Hex digests are lowercase and contain 64 characters.

The recipe address identifies a computation; the blob address identifies its result. Two different recipes can produce the same blob. An evaluation record connects the two addresses after successful execution and hashing. The recipe address alone does not tell a client what the output hash will be.

For this version, operation names such as `concat/v1` fix their behavior. A later specification layer can distinguish behavioral specifications from their implementations, as described in the [Ocean vision](../VISION.md#5-operations-and-their-meaning).

## The first operators

Keep the initial operator set small, deterministic, and about bytes.

| Operation | Inputs | Result |
|---|---|---|
| `cat/v1` | One blob or recipe reference | Its bytes, unchanged |
| `concat/v1` | An ordered list of references | Their bytes joined in that order |
| `slice/v1` | A reference, byte offset, byte length | Exactly that range of bytes |

Offsets and lengths are nonnegative integers at most `2^53 - 1`. Out-of-range slices fail rather than silently truncate. An empty concatenation returns an empty blob. Zero-length slices are valid at any offset up to and including the input length. Text encoding has no special meaning to these operators.

Recipes use a deliberately restricted JSON array format:

```json
["cat/v1","blob:sha256:<digest>"]
["concat/v1",["blob:sha256:<first>","blob:sha256:<second>"]]
["slice/v1","recipe:sha256:<concat-digest>",2,4]
```

The angle-bracket values above are explanatory placeholders, not valid addresses. Actual recipes contain only these array shapes, the exact operation names, valid addresses, and integers. Canonical encoding is compact JSON with no whitespace or trailing newline, unescaped ASCII strings, and integers in decimal without leading zeros or exponent notation. The server rejects duplicate or extra fields by rejecting anything outside these shapes, canonicalizes valid input, and returns the resulting address. Clients use the same encoding to verify it.

Inputs must already exist and be authorized for recipe construction when a recipe is accepted. First-version recipes have one owner who controls every input; mixed-owner composition is deferred. The server checks the dependency graph and rejects cycles, unknown operators, and excessive depth or size. Operators cannot read a clock, use randomness, fetch a URL, access arbitrary files, or invoke uploaded code.

## The client and server contract

All requests use HTTPS and Concord proof-of-possession authentication. The first deployment supports self-service storage for newly created principals, including Alice and Bob, each with authorized devices and provider-assigned quotas. Provider resource limits are separate from participation in Concord; self-hosting remains possible. Resource ownership, pins, names, and job authority are principal-scoped. Addresses are identifiers, not credentials. The [Concord specification](concord-v1.md) defines self-created identities, optional attestations, direct grants, revocation, and recovery. Internal tenant partitions are an implementation detail, not the identity model.

The following paths define the proposed interface; an implementation should turn these rules into a versioned API schema.

| Request | Behavior |
|---|---|
| `PUT /v1/blobs/{digest}` | Upload bytes; verify their SHA-256 before accepting them |
| `GET /v1/blobs/{digest}` | Return retained or currently cached bytes, or report that they are unavailable |
| `POST /v1/recipes` | Validate and store a recipe; return its address and canonical encoding |
| `GET /v1/recipes/{digest}` | Return the canonical recipe |
| `POST /v1/evaluations` | Request evaluation of a recipe; return a job identifier |
| `GET /v1/evaluations/{id}` | Return queued, running, succeeded, failed, or canceled status |
| `DELETE /v1/evaluations/{id}` | Request cancellation of a job |
| `POST /v1/pins` | Retain an address and, for a recipe, its dependency closure |
| `DELETE /v1/pins/{id}` | Release that retention commitment |
| `GET /v1/refs/{name}` | Read a named target and its version |
| `PUT /v1/refs/{name}` | Create or change a name with a matching version precondition |

An evaluation request supplies a recipe address, an optional expected output blob address, and ceilings for output bytes and elapsed time. The server applies whichever is stricter: the requested limits or its configured limits. A successful job returns the recipe address, output blob address, output length, and result availability expiry. Clients fetch the result through the blob endpoint and verify its hash locally.

Blob reads do not silently launch computation. To rebuild an evicted result, submit its recipe again. Automatic selection among alternative recipes is a later extension; the first version records the relationships but lets the client choose the recipe explicitly.

Creating a name requires an `If-None-Match: *` precondition. Replacing one requires the version returned as its ETag in `If-Match`. A stale update fails with `412 Precondition Failed`, allowing the client to read and decide again. Names are owner-principal-qualified signed bindings, not provider-account paths. The naming service verifies the writer's current authority and binding signature as well as the version precondition. Recipes always use immutable addresses. HTTPS gateway links are locators, not ownership credentials.

## What happens during evaluation

1. Resolve the complete dependency graph and check current Concord authority. A scoped evaluate grant may allow the service to use inputs without allowing the recipient to download them. Temporarily retain inputs for the job's lifetime.
2. Reuse a verified cached result if it is still present; otherwise evaluate the required nodes, sharing repeated dependencies.
3. Enforce limits on graph size, depth, active jobs, execution time, temporary storage, and output size. Stop work when a limit is reached.
4. Write the result to temporary storage while hashing it. If an expected output address was supplied, compare it with the computed address.
5. Recheck live grants and device authority before atomically publishing the complete blob and evaluation record with recipient access. Revoked jobs publish no new result. Return the output address and length.

A partial upload or interrupted evaluation never becomes a readable blob. Upload hash mismatches and expected-result mismatches fail without publishing a successful record. Repeating the same upload is idempotent. Repeating an evaluation may create another job, but produces the same bytes and can reuse existing work.

The client can verify that downloaded bytes match their address. When it has not independently evaluated a recipe, it still trusts the server's claim that those bytes are the correct result of that recipe. An output hash is integrity evidence, not a proof of execution.

## Retention is part of the interface

Every successful upload, recipe creation, and evaluation result has a server-reported staging expiry at least one hour in the future. This gives clients time to assemble recipes and create pins. Active jobs hold temporary leases on their dependencies and outputs until completion, after which the output receives its staging window.

A successful pin means the server has durably recorded both the object and the commitment to retain it. Pinning a recipe atomically retains its recipe dependencies and source blobs. If anything is unavailable or the principal's quota would be exceeded, pin creation fails without creating a partial commitment. Clients can instead pin a result blob when preserving the bytes alone is sufficient.

A named reference owns a pin on its target's closure. Changing the name atomically retains the new closure before releasing the old reference-owned pin. Explicit pins remain independent, so clients can preserve older versions before moving a name. Mutable names alone do not provide an enduring history.

Garbage collection may remove objects only when they have no pins, active job leases, or unexpired staging leases. Cached intermediate and final outputs may be discarded under those same rules; a recipe pin preserves the inputs needed to rebuild them, not every materialized result. Removing one pin cannot remove data still retained by another.

For the first deployment, durable means surviving a service restart on the same persistent storage. Replication, backup, and protection against disk or provider loss require additional commitments and are outside this PoC. Content addressing alone does not provide them.

## Access and encryption

This PoC trusts one remote provider to store data and execute the fixed operators. Concord access checks apply to every operation and dependency, including cache hits. Knowing an address grants no access. Direct, recipient-bound grants permit sharing within this provider; transitive delegation and cross-provider execution are deferred. Internal deduplication never establishes ownership or permission. Shared computation retains owner-pinned inputs, while recipients may pin authorized results against their own quota.

Clients can upload encrypted bytes as ordinary blobs. The server can retain and retrieve them without a decryption key. Operators see ciphertext in that case; a slice of ciphertext is not a slice of the original document. Structured operations on confidential plaintext would require client-side evaluation or a separately authorized execution environment.

Support for self-created Concord identities, chosen presentations, optional attestations, direct grants, device revocation, and recovery is in scope for this revised PoC. No endorsement or introduction is required to create an identity or store one's own data. End-to-end recipient encryption, transitive delegation, and a distributed trust graph are later work. The first evaluator is explicitly trusted with plaintext, and identity signatures do not change that confidentiality boundary.

## A small implementation boundary

Build one deployment with separate identity/authority, naming, blob storage, and bounded-evaluation interfaces. They may share a process and durable database. Identity checkpoints and revocations must survive crashes and must not be garbage-collected as cached content. Supply a thin command-line client that uploads, creates recipes, evaluates, downloads, pins, and changes references. Start with complete-object transfers; resumable uploads, range requests, and streaming results to clients can follow later.

This version needs no desktop UI, AI agent, peer discovery, arbitrary execution, or recipe optimizer. Its useful product is a remote service with observable storage, reconstruction, and retention behavior.

The first demonstration should prove:

- Alice uploads `ABCD` and `EFGH`, creates the concatenation and slice recipes, and pins the slice recipe.
- Bob, using his independently created identity and an exact evaluate grant from Alice, evaluates the recipe with no local object cache and downloads `CDEF`; its locally computed hash matches the returned blob address. A third identity cannot obtain the result or inputs by knowing their hashes.
- A second recipe, `concat/v1` over blobs `CD` and `EF`, produces the same output blob address, while keeping a different recipe address.
- After output staging leases expire, evict unpinned materialized results through a test-only maintenance hook. Evaluating the pinned slice recipe still reconstructs `CDEF`.
- A service restart preserves pinned data and named references. Competing name updates cannot silently overwrite one another.
- Corrupt uploads, missing dependencies, unauthorized reads, invalid slices, and jobs exceeding their limits fail without exposing partial results.
- Releasing a pin leaves shared dependencies intact while another pin needs them; after all leases expire and pins are removed, those objects become eligible for collection.

## Lessons incorporated and validation added

The [Kopia comparison](operator-trees-vs-kopia.md) motivates measuring materialization rather than assuming recipes save space. Before accepting a pin, expose logical dependency size and marginal retained bytes under the principal's quota. Benchmark a small slice of a large concatenation, including temporary storage and bytes read. No performance advantage is assumed.

The [Upspin comparison](operator-trees-vs-upspin.md) motivates separate identity, naming, and storage contracts. Named bindings use conditional signed updates; a reconnecting client can initially re-read the current binding. A subscription API is deferred rather than implying watch support.

In addition to the storage scenarios above, require the [Concord acceptance scenarios](concord-v1.md#acceptance-scenarios). Test corruption, missing dependencies, restart recovery, and reclamation racing with evaluation. A reachable hash is not proof that bytes remain recoverable.

## Decisions to revisit after the PoC

The first version deliberately makes recipe selection explicit and runs only fixed byte operators. The next design questions are whether retrieval should discover alternative recipes automatically, how providers advertise retention and execution costs, and how encrypted object graphs travel between providers under Concord authority.

Those extensions should preserve the central distinction: **a recipe says how information can be produced; a content address says exactly which information was produced.**

[Back to Ocean](../README.md) · [Ocean vision](../VISION.md)
