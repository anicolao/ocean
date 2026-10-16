# Concord protocol and security requirements

Requirements for the [Concord v1 design](concord-v1.md). This companion records validation rules, limitations, and deferred work.

## Participation and cryptographic authority

Identity creation works locally without invitations, endorsements, or legal-identity evidence. Self-service providers and self-hosting supply storage; provider quotas and billing govern resource consumption.

Concord v1 uses possession of a decryption key as read permission. It has no separate access-grant object, read ACL, recipient authorization lookup, or evaluate-only permission. Fetching ciphertext need not require the owner's approval or recipient authentication. A provider may apply transport limits without making those limits the confidentiality boundary.

Stream authorship is established by owner signatures. Provider requests that append events, change locators, spend quota, or remove the caller's pins authenticate the responsible principal. Possession of a stream key does not enable forging its owner's signature or spending someone else's quota.

## Identity and record encoding

Genesis contains version, random 256-bit nonce, initial signing public key, initial encryption public key, and separate recovery encryption public key. The principal ID is lowercase SHA-256 over the UTF-8 prefix `concord-genesis-v1`, one LF byte, and canonical genesis bytes.

Use Ed25519 signatures and JSON Canonicalization Scheme through established libraries. Signed record types have distinct domain prefixes followed by LF and their canonical unsigned payloads. Public keys and signatures use unpadded base64url. Reject duplicate JSON keys, unsupported versions, invalid encodings, and integers outside the supported exact range. [Ed25519](https://www.rfc-editor.org/rfc/rfc8032.html) · [JCS](https://www.rfc-editor.org/rfc/rfc8785.html)

Operational signing, operational encryption, and recovery encryption use separate key material. The v1 encryption profile must select and document a standard authenticated-encryption suite and standard public-key wrapping scheme, including nonce rules, authenticated context, and interoperable test vectors, before implementation. This design does not prescribe a custom cryptographic construction.

## Encrypted streams and keyboxes

An owner-signed stream header contains version, random stream ID, owner principal, and encryption-key version. Each stream starts with an independently random key.

Every event binds the stream ID, sequence, previous-event hash, key version, and encrypted payload. Bind these fields through authenticated encryption and the owner's signature. Use fresh nonces according to the selected suite. Clients verify signatures, hashes, sequence, and decryption before applying events. Append uses an expected-head precondition; conflicting successors require explicit resolution.

A keybox contains the stream ID, key version, recipient public-key fingerprint, creator identity, wrapped stream key, and creator signature. The authenticated wrapping context binds these fields. Verify the recipient key against their genesis and trusted contact exchange before sharing. A key received from an unverified locator can be substituted.

Create keyboxes for the owner's operational encryption key and recovery public key at stream creation. Sharing wraps the same stream key for the recipient. Bind large encrypted attachments to their encrypted event references; keep attachment keys and private plaintext digests inside the encrypted payload.

A recipient can retain plaintext, copy a stream key, or wrap it for someone else. A creator signature provides provenance for a keybox, not a non-delegation guarantee. Deleting a keybox does not retract a key already obtained. Keybox expiry cannot revoke decryption.

The initial key covers retained history and future events using that key. Use separate streams for different audiences or selected results. Prospective exclusion requires a fresh key and new ciphertext for a new stream; it cannot revoke old ciphertext or copied plaintext. Automated membership changes and key-rotation protocols are future work.

## Presentations and attestations

Verify the subject signature and exact presentation digest before offering to attest. Show the claims, evidence, and proposed statement. A signature establishes signing-key control, not truth.

A static QR code can be copied. Claims about current key control require a fresh challenge tied to the presentation. Record other verification methods as described by the attestor.

Changed presentations receive new digests. Earlier attestations stay bound to their original claims and digest. Public disclosure requires an explicit preview; private evidence remains in separately shared streams. Optional evidence commitments use random salts kept with the evidence rather than public hashes of guessable fields.

Public statements and recipient keybox metadata can reveal relationships. Presentations under one principal are linkable. Neither signatures nor encryption imply universal trust scores or unlinkable personas.

## Attestation status and discovery

Expiry, replacement, and withdrawal affect a statement's standing, not the existence of old copies. Signed changes reference the original statement. Observers retrieve the public or decryptable status history and see its source and last-check time.

Providers cannot guarantee discovery of every statement or withdrawal. An unavailable or stale status must remain visible; absence of a known withdrawal is not proof of absence.

Public service-location updates form a signed chain with sequence and previous-record hash. Clients bootstrap from an exchanged identity record or local configuration, verify to genesis, and retain the greatest verified head. A valid old prefix can fool a fresh client lacking a newer checkpoint; signatures alone do not establish freshness.

Migration copies retained history and publishes a signed location update during handoff. If the old locator disappears first, a verified contact exchange supplies the new one.

## Provider operations and naming

For state-changing provider requests, issue a random single-use challenge valid for 60 seconds. Sign the service principal, canonical HTTPS origin, method, exact request target, body SHA-256, caller principal, and challenge. Verify against the caller's genesis signing key and consume the challenge atomically. Require TLS and reject replay, changed bodies, wrong audiences, or origin-changing redirects.

The provider verifies stream-owner signatures and expected heads on append. It tracks caller-owned retention commitments and job quotas. Removing one caller's pin must preserve objects retained by other pins or active jobs.

Name bindings contain owner principal, path, immutable target, sequence, and previous-binding hash. Updates require the owner's signature and expected version. Private bindings are encrypted in streams. URLs and object hashes are locators and integrity identifiers; readable access comes from keys.

## Evaluation and confidentiality

An evaluator receives a keybox for a dedicated stream containing the recipe and all input keys it needs. It can therefore decrypt those inputs. There is no promise that it will learn only the output, forget plaintext, or lose access when a job ends.

The client prepares an output stream key with keyboxes for itself, its recovery key, and the evaluator. The evaluator encrypts its signed result artifact under that key with job and stream context. The owner verifies and appends the result to its own signed history. The evaluator's signature makes the result attributable; a content digest alone does not prove correct execution.

Persist private recipes, inputs, outputs, and evaluation records as encrypted objects. Keep plaintext hashes inside encrypted records. Any plaintext cache or temporary execution state belongs to the explicitly trusted evaluator, outside the opaque storage contract. Persisted output caches must be encrypted for their output stream; reusing bytes for another audience requires that audience's appropriate encryption.

## Recovery preparation and future work

V1 creates and exports a separate recovery encryption key, verifies the saved material using a fixture keybox, and includes recovery keyboxes on every owned stream. Remove the recovery private key from everyday client storage after confirmed export.

This reserves a path to recovering content. A recovery encryption key does not by itself establish authority to replace a signing key. Recovery transitions, identity takeover rules, lost-device replacement, operational key rotation, and multiple-device management need a later protocol. V1 does not offer a working recovery workflow.

Losing the operational key can make the current identity unusable until that future work exists. Loss of both operational and recovery keys makes their encrypted content unrecoverable. Theft of the recovery private key exposes streams whose recovery keyboxes the thief obtains.

## Persistence

Preserve genesis records, signed heads and histories, keyboxes, encrypted objects, disclosed withdrawals, and retention commitments across service restart. Cache rebuilding must preserve the greatest trusted history checkpoint. Missing dependencies and corrupt objects fail verification.

## Required failure tests

1. Create an unattested identity and store its first encrypted stream without an invitation.
2. Share with Bob; verify historical and subsequent events decrypt. Carol cannot decrypt with Bob's keybox.
3. Reject altered events, swapped keyboxes, wrong stream contexts, substituted recipient keys, and invalid owner signatures.
4. Demonstrate that knowing a stream key cannot forge owner-authored events; demonstrate that a recipient can retain and redistribute readable content.
5. Reject stale appends, request replay, cross-service requests, and rollback below a client's trusted checkpoint.
6. Verify exact presentation binding and keep private evidence out of public attestations.
7. Open a fixture keybox using exported recovery material and verify newly created streams have recovery keyboxes.
8. Share evaluator inputs explicitly, verify encrypted result delivery, and ensure stored private artifacts contain no plaintext payloads or plaintext digests.
9. Restart storage, verify history and decryption, and test retention races and missing dependencies.

Wire schemas, the encryption suite, interoperable vectors, and security review remain prerequisites for deployment with valuable private data.

[Concord design](concord-v1.md) · [Remote Operator Trees](remote-operator-trees.md)
