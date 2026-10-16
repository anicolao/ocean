# First Concord identity service

Revised initial scope for Ocean, 16 October 2026

Concord identity is part of the first remote Operator Trees prototype. Alice and Bob should be able to establish distinct identities, introduce themselves, authorize a specific shared computation, and change devices without replacing their identity. Provider accounts and bearer tokens alone do not meet that goal.

This is a proposed minimal identity and authority implementation, not the complete Concord system or a production-ready cryptographic protocol. Wire schemas, protocol test vectors, and security review are required before deployment with valuable private data.

## What the comparisons change

- From [Upspin](operator-trees-vs-upspin.md): separate identity discovery, names, storage, and evaluation. Provide a concrete sharing model immediately.
- From the [URL comparison](urls-vs-upspin-naming.md): use HTTPS endpoints without equating an endpoint or friendly name with a principal. Bind names and service locations with signatures.
- From [Kopia](operator-trees-vs-kopia.md): retain a clear boundary between logical identity and physical storage, make dependencies visible to reclamation, and verify recoverability rather than merely the existence of hashes.

One process may host all services initially, but their records and authorization checks must remain distinct. The identity service stores public records and current authority state; private keys remain on user-controlled devices or in offline recovery material.

## Identity and keys

A client generates a genesis record containing a version, a random 256-bit nonce, an initial controller public key, and an offline recovery public key. Its principal ID is the lowercase SHA-256 digest of a domain prefix `concord-genesis-v1` followed by one LF byte and the canonical record bytes. No host, email address, or display name participates in this identity.

Use Ed25519 signatures and JSON Canonicalization Scheme for these new signed records, implemented through established libraries. Each record signs a distinct domain prefix, one LF byte, and its canonical unsigned payload. Reject duplicate JSON keys, unknown schema versions, invalid encodings, and integers outside the supported exact range before verification. Public keys and signatures use unpadded base64url. This does not change the existing recipe encoding. [Ed25519 specification](https://www.rfc-editor.org/rfc/rfc8032.html) · [JCS specification](https://www.rfc-editor.org/rfc/rfc8785.html)

The controller authorizes devices and identity updates. Each device has its own signing key and a controller-signed certificate specifying principal, key, roles, recovery epoch, expiry, and certificate ID. Roles initially cover data operations and naming; they never imply controller or recovery authority. A service is also a principal with an authenticated device or service key.

Friendly names are local address-book entries attached to verified principal IDs. The first version makes no claims about legal identity or globally unique human names.

## Introductions and enrollment

Alice exchanges an invitation with Bob through an existing trusted channel, a QR code in person, or a separately checked fingerprint. It contains her genesis record, signed current profile, a random invitation ID, intended recipient principal when known, expiry, and an optional proposed grant. Bob replies with his identity and a signed acceptance bound to the invitation digest.

An invitation without a pre-bound recipient is only an introduction: Alice must confirm the returned fingerprint before a grant is issued. Relays cannot safely select a new trusted recipient for her. Invitations are single-use at the accepting service and expire after 24 hours. Replaying an accepted invitation cannot create another grant.

An introduction establishes a local relationship, not transitive trust and not permission to access any data. Storage providers separately enroll principals and assign quotas. In the PoC, an administrator allowlists Alice and Bob; an invitation cannot promise free storage or computation on someone else's service.

## Authentication without shared bearer credentials

For each protected operation, the server issues a random, single-use challenge valid for 60 seconds. The client signs a request envelope containing the challenge, service principal ID, canonical HTTPS origin, HTTP method, exact request target, body SHA-256, caller principal, device certificate ID, and requested authority IDs. The server verifies the envelope, certificate, current identity state, and operation authorization before atomically consuming the challenge.

TLS remains required. Reject reuse, expired challenges, changed request bodies, incorrect service audiences, and revoked devices. This intentionally chatty PoC protocol avoids defining reusable sessions; later session optimization must preserve proof of possession and revocation checks. Requests must not follow redirects while carrying a signed authorization envelope to another origin.

## Signed profiles and discovery

A controller-signed profile identifies the principal, recovery epoch, sequence, previous-record hash, expiry, active device certificates, and service descriptors. A descriptor binds a role, service principal, and HTTPS URL. The initial roles are identity publication, naming, storage, and evaluation.

The initial service keeps one current, conditionally updated profile chain per principal. A client bootstraps its identity-publication locator from an introduction or local configuration and verifies records back to genesis. Profiles are public and must not contain contact graphs, private object names, or grants. Applications fetch private authority records only after authorization.

Clients remember the greatest verified epoch and sequence. Older records are rejected; conflicting successors stop automatic resolution and require explicit repair. Signatures detect alteration but do not prove that a relay returned the newest record. The first deployment trusts its configured identity publisher for fresh state and availability; it is not a globally consistent decentralized directory.

Moving publication to a new provider requires a signed transition available at the old location during handoff, copying the record history, and confirming the new service before switching. If the old locator disappears first, contacts need a new locator through a trusted introduction; no automatic global discovery is claimed. Identity continuity is possible without promising that every old URL stays reachable.

## Authority and sharing

A grant is controller-signed and includes grant ID, issuer and recipient principal IDs, service audience, issuer recovery epoch, exact resource scope, allowed actions, expiry, and `delegable: false`. The first version permits only direct grants from a resource owner. No groups, transitive delegation, wildcard paths, or inferred social authority.

Actions distinguish reading bytes, reading a recipe, evaluating it, and updating a named binding. Creating storage, pins, or jobs also requires the caller's provider quota. Grant possession alone is insufficient: the recipient must authenticate with a currently authorized device.

For a recipe grant, scope records an immutable recipe and its exact dependency closure. The owner must control every member; mixed-owner compositions are deferred. A read grant on one result does not expose its sources. An evaluate grant authorizes the service to use the listed inputs for that operation and publish its result to the recipient; it does not grant direct downloads of the sources. The owner sees that evaluation releases derived information before approving it.

The server stores grants and revocations durably and checks current authority on every protected read, cache hit, job admission, and result publication. A canceled or newly unauthorized job publishes no new result. Previously downloaded bytes cannot be recalled. Internal deduplication must not expose another principal's data or grant access merely because hashes match.

Sharing does not transfer ownership, retention, or storage bills. The owner pins the inputs needed for the promised computation; the recipient may separately pin an authorized result under their own quota. Grant expiry removes future access but does not delete data retained by another valid pin.

## Rotation, recovery, and revocation

Normal controller rotation requires a transition signed by both old and new controllers at the next profile sequence. Device revocation is a controller-signed update. The service retains a durable current-state checkpoint and rejects transitions that branch from an obsolete head.

The offline recovery key may publish a recovery record with a strictly higher recovery epoch and a replacement controller. Recovery invalidates all old device certificates and grants issued by that identity; explicit reissuance is required. Inbound grants addressed to the stable principal survive, subject to their issuers' state. The recovery key is fixed in genesis for this PoC; recovery-key rotation and threshold recovery are deferred.

A stolen recovery key can take over the identity. Losing both controller access and recovery material makes recovery impossible. Neither an email reset nor a provider administrator can silently replace identity keys. Export and restoration of recovery material must be tested before using the prototype for anything important.

The single service applies revocations before acknowledging them and rechecks authorization before publishing results. A future separately hosted worker must obtain current authorization from that authority service; if it cannot, it fails closed. Offline authorized writes and availability during an authority outage are outside the initial guarantee.

## Naming and encrypted data

Names are principal-qualified path segments following the [naming comparison](urls-vs-upspin-naming.md). A signed binding contains owner principal, path, immutable target, sequence, previous binding hash, and writer device certificate. A valid device role or exact-name grant must authorize the writer. Conditional updates prevent stale replacement; signature verification alone does not establish freshness.

Serving a name requires its own read authorization. Knowing a principal, path, or blob hash grants no access. Profile discovery is public, but bindings, grants, and dependency graphs are private to authorized callers and visible to the trusted provider. A name owns a pin as specified in the storage design.

The first evaluator is explicitly trusted with submitted plaintext. Client-encrypted blobs can be stored as opaque bytes, but this identity implementation does not yet supply recipient key wrapping or private plaintext computation. A signing key is not reused as an encryption key. Concord's complete encrypted sharing layer remains separate work; invitations and signed grants here establish service authority, not end-to-end content secrecy.

## Minimal API responsibilities

| Interface | Initial responsibility |
|---|---|
| Identity records | Publish genesis, fetch verified history, conditionally append controller/recovery transitions |
| Introductions | Issue, accept, and consume invitations without automatically granting access |
| Authentication | Issue one-use challenges and validate signed requests |
| Authority | Register direct grants, record revocations, check live permissions |
| Naming | Read authorized bindings and apply signed conditional updates |
| Storage and evaluation | Enforce identity, permissions, leases, quotas, and publication checks |

Identity records, revocations, grant state, and pin ownership are durable control data, not ordinary evictable evaluation results. Back them up with their consistency checkpoints. Restoration must not silently resurrect a revoked device or reset an identity epoch.

## Acceptance scenarios

1. Alice and Bob create distinct identities on separate clients. A verified introduction connects their local contacts without creating data access.
2. Alice uploads the two sample blobs, creates and pins the recipe, and grants Bob evaluation. Bob obtains `CDEF`; an uninvited third principal cannot read the result, sources, recipe, or private name by guessing an address.
3. Bob's compute-only grant permits the result but not source downloads. An altered scope, wrong recipient, expired grant, or cross-service replay fails.
4. Alice adds a second device, revokes the first, and recovers with the offline key. Old certificates and issued grants stop working, including against cached results; her principal ID is unchanged.
5. A replayed challenge, forked profile, stale name update, or rolled-back authority checkpoint is rejected or explicitly stops service for repair.
6. Alice moves the naming endpoint using a signed profile update. Bob follows the verified update without replacing Alice's identity; an unsigned gateway substitution fails.
7. Revoke access during evaluation and verify that no new result is published. Stop and restart the service and verify that revocation and retention survive.
8. Corrupt stored bytes and remove an input in a controlled test. Verification detects the damage and the job fails; it must not return a successful but unreconstructable result record.

This scope adds enough Concord to test two independent people using the remote substrate. It deliberately leaves full social discovery, distributed consensus, private metadata, encrypted recipient sharing, and autonomous agents for subsequent work.

[Remote Operator Trees design](remote-operator-trees.md) · [Naming comparison](urls-vs-upspin-naming.md)
