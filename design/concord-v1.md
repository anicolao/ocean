# First Concord identity service

Revised initial scope for Ocean, 16 October 2026

Concord identity is part of the first remote Operator Trees prototype. Anyone should be able to create an identity and start storing information without an invitation or endorsement. Alice and Bob can then choose what to disclose about themselves, vouch for specific claims, authorize a shared computation, and change devices without replacing their identity. Provider accounts and bearer tokens alone do not meet that goal.

This is a proposed minimal identity and authority implementation, not the complete Concord system or a production-ready cryptographic protocol. Wire schemas, protocol test vectors, and security review are required before deployment with valuable private data.

## What the comparisons change

- From [Upspin](operator-trees-vs-upspin.md): separate identity discovery, names, storage, and evaluation. Provide a concrete sharing model immediately.
- From the [URL comparison](urls-vs-upspin-naming.md): use HTTPS endpoints without equating an endpoint or friendly name with a principal. Bind names and service locations with signatures.
- From [Kopia](operator-trees-vs-kopia.md): retain a clear boundary between logical identity and physical storage, make dependencies visible to reclamation, and verify recoverability rather than merely the existence of hashes.

One process may host all services initially, but their records and authorization checks must remain distinct. The identity service stores public records and current authority state; private keys remain on user-controlled devices or in offline recovery material.

## Identity and keys

A client generates a genesis record containing a version, a random 256-bit nonce, an initial controller public key, and an offline recovery public key. Its principal ID is the lowercase SHA-256 digest of a domain prefix `concord-genesis-v1` followed by one LF byte and the canonical record bytes. No host, email address, or display name participates in this identity.

Use Ed25519 signatures and JSON Canonicalization Scheme for these new signed records, implemented through established libraries. Each record signs a distinct domain prefix, one LF byte, and its canonical unsigned payload. Reject duplicate JSON keys, unknown schema versions, invalid encodings, and integers outside the supported exact range before verification. Public keys and signatures use unpadded base64url. This does not change the existing recipe encoding. [Ed25519 specification](https://www.rfc-editor.org/rfc/rfc8032.html) · [JCS specification](https://www.rfc-editor.org/rfc/rfc8785.html)

The controller authorizes devices and identity updates. Each device has its own signing key and a controller-signed certificate specifying principal, key, roles, recovery epoch, expiry, and certificate ID. Roles initially cover data operations, naming, and optional attestation signing; they never imply controller or recovery authority. A service is also a principal with an authenticated device or service key.

Self-chosen display names and avatars belong to optional identity presentations. Local contact labels remain separate conveniences. Neither is a globally unique name or proof of legal identity. A valid signature proves control of a signing key, not the truth of its owner's claims.

## Independent creation and storage

Identity creation happens locally. It needs no invitation, endorsement, central registration, or proof of legal identity. A newly created principal with no attestations can authenticate, publish its records, and store its own information using a compatible provider or a self-hosted service.

The PoC must include a self-service path from a fresh identity to its first stored and pinned blob. Remove the administrator allowlist from the normal path. A provider can enforce capacity, payment, and abuse limits on its own resources, but this is a service arrangement, not admission to Concord. Running one's own provider must remain possible; no identity attestor is a system gatekeeper. This does not promise unlimited free storage or solve denial-of-service and mass-account abuse.

## Chosen identity presentations

A presentation is a subject-signed, immutable record containing the principal ID, presentation ID, selected claims, optional avatar, creation time, and optional evidence references. A person may make several presentations with different disclosures under the same principal. Those presentations are linkable through that principal; they are not unlinkable personas.

A QR code can contain a small presentation or its digest and retrieval locator. Larger evidence belongs in separately disclosed material. The scanner verifies the subject signature and shows the exact claims and attachments before offering to vouch. Scanning a code neither enrolls the subject nor grants access to their stored information.

The subject chooses the disclosure. One presentation might say only “Mickey Mouse” and show a drawing. Another might include a claimed legal name, address, and scans of a driver's licence, passport, and matching utility bill. Neither level is required to participate. Evidence can be shown privately to an attestor without being attached to a public record.

## Vouching through attestations

An attestation is a separate signed statement by another principal about an exact presentation and specified claims. Its payload includes:

- Attestor and subject principal IDs, an attestation ID, and the attestor's signing certificate or controller reference.
- The immutable presentation digest and the particular claims being affirmed.
- An explicit statement of what is being vouched for, the verification method, and any evidence commitments the parties choose to disclose.
- Issued time, optional expiry, and optional reference to a prior attestation being superseded.

A device may sign attestations only with a controller-issued `attest` role. Attestations use their own signature domain and never the grant or invitation format. No invitation object is part of this design.

For example, Bob could sign “I vouch for this person as the holder of this identity, presented as Mickey Mouse.” That is a meaningful endorsement of the pseudonymous presentation, not automatically a claim that Mickey Mouse is their legal name. If Bob instead signs an explicit legal-name claim without support, observers can judge Bob's care as well as the subject's claim.

For a richer presentation, Bob might sign “I inspected the presented passport and driver's licence, compared the holder to their photos, and checked that the utility bill matches the claimed address.” This records Bob's assertion and method; it does not magically establish that the documents were genuine. A signature over an ambiguous “true identity” badge would hide the distinction, so the signing interface must show a concrete statement.

If Bob claims that the person currently controls the principal, he can request a fresh signed challenge tied to the presentation. A copied static QR code alone does not establish that control. Other methods, such as an existing relationship or document inspection, are recorded as such rather than silently inferred.

An observer interprets both the claim and the attestor. They might put weight on attestations from people they know, the kinds of checks performed, the age of the statement, contradictory evidence, and the attestor's record. Identity attestations can influence how they interpret content attributed to that principal; document-backed identification is not a guarantee that everything the person publishes is reliable. The PoC exposes this evidence without a universal trust score, automatic transitive trust, or a “verified/unverified” gate to participation.

## Disclosure and attestation lifecycle

Separate the presentation shown to the attestor from the attestation shown to later observers. The subject chooses the initial evidence and intended audience; the attestor chooses the exact statement they will stand behind. Neither a QR scan nor a signature automatically publishes documents, claims, or the relationship.

A public attestation may state what the attestor checked without making the evidence public. An observer without that evidence is relying on the attestor, not independently verifying the documents. Do not publish raw hashes of guessable private fields as if hashing made them private. Optional private evidence commitments use a fresh random salt kept with the evidence; opening a commitment requires both. Signing and salted commitments do not encrypt a document. The initial prototype uses public test presentations or explicit private evidence exchange, not a new claim of end-to-end encrypted document sharing.

Changing a presentation creates a new digest; old endorsements do not transfer to changed claims. Attestors can issue signed withdrawals or replacements, which refer to the original attestation. Expiry or withdrawal changes its current standing but cannot erase copies or past observations. Attestation histories remain cryptographically verifiable across ordinary key rotation; recovery or reported compromise is shown as context for observers rather than silently rewriting earlier endorsements.

The PoC can store and retrieve disclosed attestations and withdrawals through a provider, with portable signatures and issuer references. It does not guarantee discovery of every attestation or withdrawal across all providers. Show the freshness and source of checked status; absence of a known withdrawal is not a proof that none exists. A public relationship necessarily discloses its participants, and an authorized recipient may keep or redistribute what was disclosed.

## Authentication without shared bearer credentials

For each protected operation, the server issues a random, single-use challenge valid for 60 seconds. The client signs a request envelope containing the challenge, service principal ID, canonical HTTPS origin, HTTP method, exact request target, body SHA-256, caller principal, device certificate ID, and requested authority IDs. The server verifies the envelope, certificate, current identity state, and operation authorization before atomically consuming the challenge.

TLS remains required. Reject reuse, expired challenges, changed request bodies, incorrect service audiences, and revoked devices. This intentionally chatty PoC protocol avoids defining reusable sessions; later session optimization must preserve proof of possession and revocation checks. Requests must not follow redirects while carrying a signed authorization envelope to another origin.

## Signed profiles and discovery

This control profile is separate from the optional personal presentations above. It publishes operational keys and service bindings, not identity evidence.

A controller-signed profile identifies the principal, recovery epoch, sequence, previous-record hash, expiry, active device certificates, and service descriptors. A descriptor binds a role, service principal, and HTTPS URL. The initial roles are identity publication, naming, storage, and evaluation.

The initial service keeps one current, conditionally updated profile chain per principal. A client bootstraps its identity-publication locator from a shared identity presentation or local configuration and verifies records back to genesis. Profiles are public and must not contain contact graphs, private object names, or grants. Applications fetch private authority records only after authorization.

Clients remember the greatest verified epoch and sequence. Older records are rejected; conflicting successors stop automatic resolution and require explicit repair. Signatures detect alteration but do not prove that a relay returned the newest record. The first deployment trusts its configured identity publisher for fresh state and availability; it is not a globally consistent decentralized directory.

Moving publication to a new provider requires a signed transition available at the old location during handoff, copying the record history, and confirming the new service before switching. If the old locator disappears first, contacts need a new locator through a separately verified contact exchange; no automatic global discovery is claimed. Identity continuity is possible without promising that every old URL stays reachable.

## Authority and sharing

Attestations describe claims and judgments. Grants authorize actions. Vouching for Alice never grants Alice storage access, and granting access never vouches for her identity. Neither requires the other.

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

The first evaluator is explicitly trusted with submitted plaintext. Client-encrypted blobs can be stored as opaque bytes, but this identity implementation does not yet supply recipient key wrapping or private plaintext computation. A signing key is not reused as an encryption key. Concord's complete encrypted sharing layer remains separate work; attestations establish attributable statements, and signed grants establish service authority. Neither provides end-to-end content secrecy.

## Minimal API responsibilities

| Interface | Initial responsibility |
|---|---|
| Identity records | Publish genesis, fetch verified history, conditionally append controller/recovery transitions |
| Presentations | Create and disclose subject-signed claims and selected evidence |
| Attestations | Sign, verify, store, and retrieve disclosed endorsements, replacements, and withdrawals |
| Provider setup | Self-service storage setup for a fresh principal, subject to provider resource policy |
| Authentication | Issue one-use challenges and validate signed requests |
| Authority | Register direct grants, record revocations, check live permissions |
| Naming | Read authorized bindings and apply signed conditional updates |
| Storage and evaluation | Enforce identity, permissions, leases, quotas, and publication checks |

Identity records, revocations, published attestation withdrawals, grant state, and pin ownership are durable control data, not ordinary evictable evaluation results. Back them up with their consistency checkpoints. Restoration must not silently resurrect a revoked device or reset an identity epoch.

## Acceptance scenarios

1. A fresh principal with no contacts or attestations creates an identity, configures storage without an administrator invitation, uploads a blob, and pins it. Demonstrate both the self-service provider path and a self-hosted deployment.
2. Alice uploads the two sample blobs, creates and pins the recipe, and grants Bob evaluation. Bob obtains `CDEF`; a third principal without permission cannot read the result, sources, recipe, or private name by guessing an address.
3. Bob's compute-only grant permits the result but not source downloads. An altered scope, wrong recipient, expired grant, or cross-service replay fails.
4. Alice adds a second device, revokes the first, and recovers with the offline key. Old certificates and issued grants stop working, including against cached results; her principal ID is unchanged.
5. A replayed challenge, forked profile, stale name update, or rolled-back authority checkpoint is rejected or explicitly stops service for repair.
6. Alice moves the naming endpoint using a signed profile update. Bob follows the verified update without replacing Alice's identity; an unsigned gateway substitution fails.
7. Revoke access during evaluation and verify that no new result is published. Stop and restart the service and verify that revocation and retention survive.
8. Corrupt stored bytes and remove an input in a controlled test. Verification detects the damage and the job fails; it must not return a successful but unreconstructable result record.

9. Present a “Mickey Mouse” name and drawing by QR code. Bob signs a scoped endorsement; an observer can see exactly what Bob affirmed without requiring legal documents. Scanning and attesting create no access grant.
10. With synthetic document fixtures, present a legal-name and address claim plus evidence privately. Publish only the agreed attestation summary; verify that evidence bytes, locators, and private-field hashes are not exposed by default.
11. Edit a presentation and verify that the earlier attestation no longer endorses the new digest. Verify expiry, signed withdrawal, stale status reporting, and attribution to the attestor.
12. Sign conflicting or poorly supported assertions and show both with their issuers. No endorsement count or automated score blocks identity creation or ordinary self-service storage.

This scope adds enough Concord to test independent identity creation, optional vouching, and two people using the remote substrate. It deliberately leaves full social discovery, distributed consensus, private metadata, encrypted recipient sharing, and autonomous agents for subsequent work.

[Remote Operator Trees design](remote-operator-trees.md) · [Naming comparison](urls-vs-upspin-naming.md)
