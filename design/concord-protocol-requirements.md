# Concord protocol and security requirements

Implementation requirements for the [first Concord identity service](concord-v1.md).

This companion document records protocol constraints, failure behavior, and the limits of the initial design. The user flow and component model live in the main design. Wire schemas, interoperable test vectors, and security review are required before deployment with valuable private data.

## Participation and authority boundaries

- Identity creation must work locally, without invitations, endorsements, central registration, or legal-identity evidence.
- The normal first-storage flow must work through self-service provider setup or self-hosting. An administrator allowlist is not the normal enrollment path. Provider capacity, payment, and abuse policies apply to that provider's resources, not to the existence of the identity.
- Scanning a QR code, signing an attestation, and issuing an access grant are separate actions. Neither scanning nor vouching grants access or automatically publishes records.
- Attestations must identify the exact claims affirmed. A valid signature proves signing-key control, not the truth of a claim. Document-backed identification does not guarantee the reliability of all content attributed to a person.
- The first version has no universal trust score, automatic transitive trust, or endorsement threshold for participation. Presentations under one principal are linkable; they do not provide unlinkable personas.

## Record encoding and signatures

The genesis record contains a version, random 256-bit nonce, initial controller public key, and offline recovery public key. The principal ID is lowercase SHA-256 over the UTF-8 prefix `concord-genesis-v1`, one LF byte, and canonical genesis bytes.

Use Ed25519 signatures and JSON Canonicalization Scheme through established libraries. Each signed record type has a distinct domain prefix followed by one LF byte and its canonical unsigned payload. Public keys and signatures use unpadded base64url. Reject duplicate JSON keys, unsupported versions, invalid encodings, and integers outside the supported exact range before verification. Recipe encoding remains as defined in the storage design. [Ed25519](https://www.rfc-editor.org/rfc/rfc8032.html) · [JCS](https://www.rfc-editor.org/rfc/rfc8785.html)

Device certificates contain principal, public key, roles, recovery epoch, expiry, and certificate ID. Data, naming, and `attest` roles do not imply controller or recovery authority. Attestations, grants, and control transitions must have separate signature domains.

## Presentations and evidence

- Verify the subject signature and presentation digest before offering to attest. Show the claims, evidence, and proposed statement being signed.
- A static QR code can be copied. A claim of current key control requires a fresh challenge tied to the presentation. Record other verification methods as the attestor describes them.
- A changed presentation receives a new digest. Earlier attestations must remain bound to their original digest and claim set.
- Keep private evidence out of public control profiles. These profiles must also exclude contact graphs, private names, and grants.
- Publication must use explicit disclosure choices. A public attestation may describe checks while withholding the evidence. Observers then rely on the attestor for that account of the checks.
- Do not publish raw hashes of guessable private fields as a privacy measure. Optional evidence commitments include a fresh random salt kept with the evidence; verification requires both. Signatures and commitments do not encrypt documents.
- Use public test presentations or explicit private evidence exchange in the prototype. End-to-end recipient encryption is separate work. Public relationships reveal their participants; recipients can retain or redistribute disclosed material.

## Attestation status

Expiry, replacement, and withdrawal affect an attestation's standing, not the existence of old copies. Signed withdrawals and replacements must reference the original attestation. Preserve historical signature verification through normal key rotation. Display recovery or reported compromise as context for evaluating past endorsements.

The initial service retrieves disclosed attestations and withdrawals from configured providers. It cannot guarantee discovery of every statement or withdrawal across providers. Show the source and time of status checks; absence of a known withdrawal is not proof that none exists. Stale or unavailable status must remain visible to the observer.

## Request authentication

For each protected operation, issue a random, single-use challenge valid for 60 seconds. The signed request envelope contains the challenge, service principal ID, canonical HTTPS origin, HTTP method, exact request target, body SHA-256, caller principal, device certificate ID, and requested authority IDs.

Verify the signature, certificate, current identity state, and operation authorization, then atomically consume the challenge. Require TLS. Reject replay, expiry, changed bodies, incorrect audiences, and revoked devices. Do not forward a signed envelope across an origin-changing redirect. Reusable sessions are deferred; any future optimization must preserve proof of possession and revocation checks.

## Profiles, discovery, and freshness

A controller-signed control profile contains the principal, recovery epoch, sequence, previous-record hash, expiry, active device certificates, and service descriptors. A descriptor binds a role, service principal, and HTTPS URL. Initial roles cover identity publication, naming, storage, and evaluation.

Maintain one current, conditionally updated profile chain per principal. Bootstrap publication locations from a shared presentation or local configuration, then verify the chain to genesis. Clients retain their greatest verified epoch and sequence, reject rollback, and stop automatic resolution on conflicting successors.

Signatures do not prove freshness. The first deployment trusts its configured identity publisher for current state and availability; it does not implement globally consistent decentralized discovery. A provider migration requires a signed transition at the old location during handoff, copying history, and confirming the new service before switching. If the old locator disappears first, a separately verified contact exchange supplies the replacement. Existing URLs are not guaranteed to remain reachable.

## Resource grants and names

A controller-signed grant includes grant ID, issuer and recipient principals, service audience, issuer recovery epoch, exact resource scope, permitted actions, expiry, and `delegable: false`. Initially, only direct resource-owner grants are supported. Groups, transitive delegation, and wildcard paths are deferred.

The recipient must authenticate with a current device certificate. Distinguish reading bytes, reading a recipe, evaluating it, and updating a named binding. Job creation, storage, and pins also require provider quota. Knowing an address or possessing someone else's grant must not confer access.

A recipe grant specifies an immutable recipe and its exact dependency closure. Its owner controls every input; mixed-owner recipes are deferred. An evaluate-only grant permits the service to use those inputs and release the result, not to let the recipient download the sources. The owner must see the information-release scope before granting evaluation.

Check current authority on every protected read, cache hit, job admission, and result publication. Canceled or newly unauthorized jobs publish no new result. Hash equality and internal deduplication never establish permission. Grant expiry ends future access but does not delete another principal's valid pins or recall downloaded bytes.

A signed name binding contains owner principal, path, immutable target, sequence, previous-binding hash, and writer device certificate. Require the writer's current role or exact-name grant, a valid signature, and the expected version. Private names, grants, and dependency graphs require authorization and are visible to the trusted provider. Names retain targets through their pins.

## Rotation and recovery

Normal controller rotation requires old and new controller signatures at the next profile sequence. Device revocation is a controller-signed update. Retain a durable current-state checkpoint and reject transitions from obsolete heads.

The offline recovery key can publish a strictly higher recovery epoch and replacement controller. This invalidates all earlier device certificates and grants issued by the recovering principal; reissuance is explicit. Inbound grants to the stable principal survive subject to their issuers' state. The recovery key is fixed in genesis; threshold recovery and recovery-key rotation are deferred.

A stolen recovery key can take over the identity. Loss of both controller access and recovery material makes recovery impossible. Provider administrators and email resets cannot replace keys. Test recovery export and restoration before real use.

Apply revocations durably before acknowledging them. A separate evaluator must obtain current authority and fail closed if it cannot. Offline authorized writes and availability through an authority outage are outside the initial guarantee.

## Confidentiality and persistence

The first evaluator is trusted with submitted plaintext. Client-encrypted blobs are opaque bytes to it. Signing keys must not be reused as encryption keys. Identity signatures and service grants do not supply end-to-end content confidentiality.

Identity checkpoints, revocations, published attestation withdrawals, grant state, and pin ownership are durable control data, not evictable computation caches. Back them up consistently. Restoration must not reset epochs or revive revoked devices. Hash reachability alone does not establish recoverability; corruption and missing dependencies must fail verification.

Full social discovery, distributed consensus, private metadata, recipient encryption, and autonomous agents remain later scope. The initial service's persistence and authority guarantees must be stated independently of those future features.

## Required failure tests

1. Reject modified presentation payloads and attestations applied to a different presentation digest.
2. Keep evidence bytes, private locators, and private-field hashes out of an attestation-only publication.
3. Show conflicting attestations and their issuers; do not turn endorsement counts into a participation gate.
4. Reject wrong-recipient grants, altered scopes, expired grants, cross-service replay, and source downloads under an evaluate-only grant.
5. Reject reused challenges, revoked devices, stale bindings, and profile rollback; stop for repair on conflicting control histories.
6. Revoke access during evaluation and verify that no new result is published, including when a cached result exists.
7. Restart and restore the service while preserving revocations, recovery epochs, withdrawals, and retention commitments.
8. Detect corrupt bytes and missing dependencies; return a failed job rather than a successful result record.
9. Verify that self-created, unattested identities can complete normal identity setup and first-storage flows.

[Concord user flow and design](concord-v1.md) · [Remote Operator Trees design](remote-operator-trees.md)
