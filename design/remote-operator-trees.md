# Remote Operator Trees

A proposed first storage substrate for Ocean, with Concord identity

Revised after the Kopia, Upspin, and URL naming comparisons. The initial implementation includes the [Concord identity and encrypted-stream service](concord-v1.md) and follows the [naming policy](urls-vs-upspin-naming.md).

## The idea

An Operator Trees server stores both bytes and descriptions of how to produce bytes. A client can upload data, compose it into a recipe, and ask the server to return the result. The result can be cached, retrieved later, or rebuilt from its inputs.

The server is the durable home for the data. A client needs its Concord operational keys, stream keyboxes, verified history, service locations, and object addresses, but no local object database or permanent local data copy. Two clients can work with the same stored objects. Local caching is optional.

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

Inputs must already exist and be decryptable by the chosen evaluator when a recipe is accepted. The caller supplies a dedicated input stream containing the recipe and the required input keys. The server checks the dependency graph and rejects cycles, unknown operators, and excessive depth or size. Operators cannot read a clock, use randomness, fetch a URL, access arbitrary files, or invoke uploaded code.

## The client and server contract

All requests use HTTPS. Concord signatures authenticate writes, retention changes, and resource-consuming jobs. Providers assign quotas to self-created principals and support self-service or self-hosted storage. Encrypted object retrieval does not depend on a separate read grant: the ability to decrypt establishes readable access.

The [Concord design](concord-v1.md) defines independent identities, optional attestations, encrypted event streams, and recipient keyboxes. Each principal creates a separate recovery key at setup; recovery workflows are future work.

### Encrypted storage and the byte-operator interface

The blob store addresses the exact bytes it retains. For a private object these are ciphertext, and the outer SHA-256 is a ciphertext digest. The recipe examples above describe logical bytes after decryption; their plaintext digests and recipe encodings stay inside encrypted stream events.

The recipe and evaluation endpoints below belong to the trusted evaluator. Private recipe submissions and returned records are encrypted artifacts, delivered through job streams and keyboxes as described in Concord. The evaluator decrypts the logical recipe, checks its canonical encoding and dependency graph, and runs the byte operators on plaintext. The opaque storage service retains only the encrypted artifacts. Logical recipe/output addresses and encrypted storage addresses are recorded separately inside the encrypted result.

Plaintext example hashes remain useful for testing deterministic computation. Randomized encryption means equal plaintext results need not have equal storage addresses or deduplicate across streams.

The following paths define the proposed interface; an implementation should turn these rules into a versioned API schema. For private recipes, path digests identify encrypted artifacts; logical recipe digests stay inside those artifacts. Opaque job IDs identify status requests.

| Request | Behavior |
|---|---|
| `PUT /v1/blobs/{digest}` | Upload bytes; verify their SHA-256 before accepting them |
| `GET /v1/blobs/{digest}` | Return retained or currently cached bytes, or report that they are unavailable |
| `POST /v1/recipes` | Open a shared recipe artifact, validate its logical recipe, and return an encrypted receipt |
| `GET /v1/recipes/{digest}` | Return the encrypted artifact containing the canonical recipe |
| `POST /v1/evaluations` | Submit input/output stream locators and evaluator keyboxes; return a job identifier |
| `GET /v1/evaluations/{id}` | Return queued, running, succeeded, failed, or canceled status |
| `DELETE /v1/evaluations/{id}` | Request cancellation of a job |
| `POST /v1/pins` | Retain an address and, for a recipe, its dependency closure |
| `DELETE /v1/pins/{id}` | Release that retention commitment |
| `GET /v1/refs/{name}` | Read a named target and its version |
| `PUT /v1/refs/{name}` | Create or change a name with a matching version precondition |

An evaluation request identifies a recipe in the shared input stream, an owner-prepared output stream, and ceilings for output bytes and elapsed time. The encrypted input includes an optional expected logical output blob address. The server applies whichever is stricter: the requested limits or its configured limits. A successful job returns an encrypted, evaluator-signed artifact containing the logical recipe address, output address, output length, and expiry. Its outer ciphertext address locates stored bytes. Clients verify the ciphertext hash, decrypt, and verify the logical output hash before appending the artifact to their own stream.

Blob reads do not silently launch computation. To rebuild an evicted result, submit its recipe again. Automatic selection among alternative recipes is a later extension; the first version records the relationships but lets the client choose the recipe explicitly.

Creating a name requires an `If-None-Match: *` precondition. Replacing one requires the version returned as its ETag in `If-Match`. A stale update fails with `412 Precondition Failed`, allowing the client to read and decide again. Names are owner-principal-qualified signed bindings, not provider-account paths. The naming service verifies the owner's binding signature and version precondition. Private names resolve through encrypted bindings in a shared stream. Recipes always use immutable addresses. HTTPS gateway links are locators, not ownership credentials.

## What happens during evaluation

1. Open the evaluator's input keybox, decrypt the recipe and required inputs, and verify their logical hashes and dependency graph. Temporarily retain encrypted inputs for the job's lifetime.
2. Reuse a verified result within the trusted evaluator's execution context if available; otherwise evaluate the required nodes, sharing repeated dependencies.
3. Enforce limits on graph size, depth, active jobs, execution time, temporary storage, and output size. Stop work when a limit is reached.
4. Write the result to temporary storage while hashing it. If an expected output address was supplied, compare it with the computed address.
5. Encrypt the complete output and signed evaluation record for the caller-prepared output stream before durable publication. Return the ciphertext locator. The caller verifies the result and appends it to their stream. Canceled jobs stop publication where cancellation wins the completion race; cancellation cannot recall data already disclosed to the evaluator.

A partial upload or interrupted evaluation never becomes a readable blob. Upload hash mismatches and expected-result mismatches fail without publishing a successful record. Repeating the same upload is idempotent. Repeating an evaluation may create another job, but produces the same bytes and can reuse existing work.

The client can verify that downloaded bytes match their address. When it has not independently evaluated a recipe, it still trusts the server's claim that those bytes are the correct result of that recipe. An output hash is integrity evidence, not a proof of execution.

## Retention is part of the interface

Every successful upload, recipe creation, and evaluation result has a server-reported staging expiry at least one hour in the future. This gives clients time to assemble recipes and create pins. Active jobs hold temporary leases on their dependencies and outputs until completion, after which the output receives its staging window.

A successful pin means the server has durably recorded both the object and the commitment to retain it. For encrypted recipes, the client or trusted evaluator supplies the ciphertext dependency manifest needed for retention; the opaque store cannot infer it from encrypted contents. Pinning atomically retains this declared closure. These retention manifests expose ciphertext linkage to the provider. If anything is unavailable or the principal's quota would be exceeded, pin creation fails without creating a partial commitment. Clients can instead pin a result blob when preserving the bytes alone is sufficient.

A named reference owns a pin on its target's closure. Changing the name atomically retains the new closure before releasing the old reference-owned pin. Explicit pins remain independent, so clients can preserve older versions before moving a name. Mutable names alone do not provide an enduring history.

Garbage collection may remove objects only when they have no pins, active job leases, or unexpired staging leases. Cached intermediate and final outputs may be discarded under those same rules; a recipe pin preserves the inputs needed to rebuild them, not every materialized result. Removing one pin cannot remove data still retained by another.

For the first deployment, durable means surviving a service restart on the same persistent storage. Replication, backup, and protection against disk or provider loss require additional commitments and are outside this PoC. Content addressing alone does not provide them.

## Access and encryption

Each principal owns signed event streams and shares their encryption keys through recipient keyboxes. Readable access follows key possession. Storage can serve ciphertext without deciding which recipients may decrypt it.

A stream key opens its retained history and future events using that key. Share a separate stream for a selected document, input set, or result. Removing a keybox cannot retract a copied key; future exclusion requires fresh keys and ciphertext.

Remote computation deliberately shares input keys with an evaluator principal. That evaluator can read the disclosed inputs. Alice can instead evaluate locally, then upload encrypted outputs. A result-only recipient receives the output stream key, while the evaluator receives the input keys needed for computation.

Self-created identities, presentations, attestations, encrypted streams, recipient keyboxes, and recovery-key preparation are in scope. Recovery workflows, operational key rotation, and automated multi-device management remain future work. The [protocol requirements](concord-protocol-requirements.md) spell out these limits.

## A small implementation boundary

Build one deployment with separate identity, encrypted-stream, naming, blob storage, and bounded-evaluation interfaces. They may share a process and durable database. Identity records, verified stream checkpoints, and keyboxes must survive crashes and must not be garbage-collected as cached content. Supply a thin command-line client that uploads, creates recipes, evaluates, downloads, pins, and changes references. Start with complete-object transfers; resumable uploads, range requests, and streaming results to clients can follow later.

This version needs no desktop UI, AI agent, peer discovery, arbitrary execution, or recipe optimizer. Its useful product is a remote service with observable storage, reconstruction, and retention behavior.

The first demonstration should prove:

- Alice encrypts `ABCD`, `EFGH`, and the concatenation/slice recipes into a dedicated input stream. She pins its ciphertext dependency closure and creates a keybox for the evaluator.
- The evaluator returns an encrypted result containing `CDEF`. Alice verifies and appends it to her output stream, then makes a keybox for Bob. Bob decrypts and verifies the logical output hash with no local object cache. A third identity can fetch ciphertext but cannot decrypt it.
- A second recipe, `concat/v1` over blobs `CD` and `EF`, produces the same logical output blob address, while keeping a different logical recipe address. Separately encrypted outputs may have different ciphertext storage addresses.
- After output staging leases expire, evict unpinned materialized results through a test-only maintenance hook. Evaluating the pinned slice recipe still reconstructs `CDEF`.
- A service restart preserves pinned data and named references. Competing name updates cannot silently overwrite one another.
- Corrupt uploads, missing dependencies, wrong-key decryption, invalid slices, and jobs exceeding their limits fail without exposing partial results.
- Releasing a pin leaves shared dependencies intact while another pin needs them; after all leases expire and pins are removed, those objects become eligible for collection.

## Lessons incorporated and validation added

The [Kopia comparison](operator-trees-vs-kopia.md) motivates measuring materialization rather than assuming recipes save space. Before accepting a pin, expose logical dependency size and marginal retained bytes under the principal's quota. Benchmark a small slice of a large concatenation, including temporary storage and bytes read. No performance advantage is assumed.

The [Upspin comparison](operator-trees-vs-upspin.md) motivates separate identity, naming, and storage contracts. Named bindings use conditional signed updates; a reconnecting client can initially re-read the current binding. A subscription API is deferred rather than implying watch support.

In addition to the storage scenarios above, require the [Concord acceptance scenarios](concord-v1.md#acceptance-scenarios) and [protocol failure tests](concord-protocol-requirements.md#required-failure-tests). Test corruption, missing dependencies, restart recovery, and reclamation racing with evaluation. A reachable hash is not proof that bytes remain recoverable.

## Decisions to revisit after the PoC

The first version deliberately makes recipe selection explicit and runs only fixed byte operators. The next design questions are whether retrieval should discover alternative recipes automatically, how providers advertise retention and execution costs, and how encrypted object graphs and recipient keyboxes travel between providers.

Those extensions should preserve the central distinction: **a recipe says how information can be produced; a content address says exactly which information was produced.**

[Back to Ocean](../README.md) · [Ocean vision](../VISION.md)
