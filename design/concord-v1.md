# First Concord identity service

Concord lets people create an identity, choose how to present themselves, vouch for one another, and share encrypted event streams. **If you can decrypt a stream, you can read it.**

## Terminology

| Term | Meaning in this design |
|---|---|
| **Identity or principal** | An actor recognized by a stable identifier and its public keys. It can represent a person, agent, or service. |
| **Owner** | The principal whose signatures establish the authoritative history of a stream. Ownership is a role relative to that stream. |
| **Event stream** | An ordered, signed history with encrypted contents. A principal can create several streams for different audiences. |
| **Stream key** | The secret used to encrypt a stream's event contents. |
| **Keybox** | A stream key encrypted for a particular public encryption key and bound to the stream and key version. |
| **Recipient** | Someone able to open a stream's keybox using their private encryption key. |
| **Recovery key** | A separate offline encryption key. Its public half receives a keybox for every owned stream; its private half is saved for future recovery tooling. |
| **Presentation** | A subject-signed selection of identity claims, an optional avatar, and chosen evidence references. |
| **Subject** | The principal whose claims a presentation or attestation concerns. |
| **Attestor** | The principal signing a statement about selected claims in an exact presentation. |
| **Attestation** | That signed statement, including what was checked and how. |

Alice and Bob are principals. Alice owns a document stream and makes a keybox for Bob. When Bob vouches for Alice, Alice is the subject and Bob is the attestor. Bob controls the history of his own statements.

## The user flow

### 1. Create an identity and save something

Alice selects **Create identity**. Concord creates her operational signing and encryption keys locally, plus a separate recovery encryption key. It helps her export the recovery private key and verify the saved copy before removing it from the everyday client.

Alice starts a signed chain of events under her new identity. She chooses a storage provider or connects her own server and saves her first document. Her client encrypts the document stream and creates keyboxes for her operational encryption key and her recovery public key.

### 2. Choose how to present yourself

Alice creates a presentation with the display name **Mickey Mouse** and a drawing for an avatar. She previews exactly what someone else will see, then selects **Show QR code**.

She can also create a presentation with her legal name, address, and selected supporting documents. Each presentation has its own audience. Both refer to the same underlying identity.

A QR code carries a small presentation or a reference to it. For a private exchange, Alice shares the relevant stream with the intended recipient using their public key. Larger evidence documents can live in a separate stream with a narrower audience.

### 3. Ask someone to vouch for you

Bob scans Alice's code. Concord shows her presentation and offers **Vouch for this identity**. Bob chooses the claims to affirm and reviews the statement he will sign.

For the Mickey Mouse presentation, he might write:

> I know this person as Mickey Mouse and vouch for them as the holder of this identity.

For the detailed presentation, he might write:

> I inspected the presented passport and driver's licence, compared the holder to their photos, and checked that the utility bill matches the claimed address.

Bob records how he checked the claims. To confirm current key control, his client can ask Alice's device to sign a fresh challenge. He signs an attestation attached to the exact presentation he reviewed.

### 4. Choose what to share

Alice might show Bob her supporting documents privately and share his resulting endorsement more widely. The preview lists the presentation, evidence, statement, and intended audience.

Alice selects a stream and Bob's identity. Concord shows his public-key fingerprint and where that key came from. She checks it through their existing contact or QR exchange, then selects **Share**.

Her client encrypts the stream key to Bob's public encryption key. Bob receives the keybox and stream locator, fetches the encrypted events, and opens them locally. The same key opens retained history and future events encrypted with it.

Alice chooses the audience by choosing which stream to share. To share just one document or a computation result, she puts that material in a separate stream.

### 5. Interpret someone else's identity

Carol follows the identity attribution on Alice's published work. She sees Alice's chosen presentation, Bob's statement, what he says he checked, when he signed it, and the source and freshness of its status.

Carol can inspect Bob's disclosed identity and attestations too. She decides how much weight to place on his endorsement using her own relationships, the evidence, and his record. A pseudonymous identity with a long history and a documented legal identity give her different information.

### 6. Keep statements current

A changed presentation receives a new version. Bob can review it and issue a new attestation; his earlier statement stays attached to the version he saw.

Bob can set an expiry, replace his statement, or publish a signed withdrawal in his event history. Carol sees these changes and when their status was last checked.

### 7. Keep recovery material safely

Alice keeps her recovery private key separately from her everyday key. Each new stream includes a keybox for its public half. Concord v1 prepares and verifies this material during setup; restoring identity control and enrolling replacement devices belong to future recovery work.

## Components

| Component | Responsibility |
|---|---|
| Identity client | Creates operational keys, exports a separate recovery key, and signs records |
| Identity record service | Publishes genesis records, signed public keys, and service locations |
| Stream client | Encrypts events, verifies histories, and makes and opens keyboxes |
| Presentation and attestation tools | Build identity cards and statements for chosen audiences |
| Naming service | Resolves owner-qualified names to signed bindings |
| Storage service | Retains encrypted events, keyboxes, and referenced encrypted objects |
| Evaluation service | Runs recipes over data deliberately shared with its own principal |

People create identities locally and use self-service providers or their own deployments. Providers manage capacity, billing, and resource usage. Each principal controls its own event history and separate recovery material.

## Identity and stream records

The stable principal ID is a fingerprint of a **genesis record** containing the initial signing public key, encryption public key, recovery encryption public key, random nonce, and format version.

Public identity records supply keys and signed service locations. Private activity lives in encrypted streams. Each stream has an owner-signed header, random identifier, random encryption key, and ordered chain of signed encrypted events. Sequence numbers and previous-event hashes let clients check the history.

The owner creates an operational keybox and a recovery keybox at stream creation. Sharing adds a keybox for the recipient's verified public key. These signed records bind the wrapped key to the stream, key version, and recipient key fingerprint. They can be delivered directly or stored alongside the encrypted stream.

Large attachments use their own content keys, carried inside encrypted events. Sharing a stream therefore also shares the referenced attachment keys it contains.

## Presentations and attestations

A presentation is an immutable, subject-signed record containing the principal ID, presentation ID, selected claims, optional avatar, creation time, and evidence references.

An attestation contains the attestor and subject IDs, signing-key reference, presentation digest, exact claims affirmed, statement, verification method, disclosed evidence commitments, issue time, optional expiry, and optional replacement reference.

Each attestor signs their own statements. Private presentations, evidence, and attestations travel in streams shared through keyboxes. Explicit public disclosure publishes the selected signed records. Evidence can remain private while the statement describing its inspection is public.

## Reading, authorship, and retention

The ability to decrypt is permission to read. Storage serves encrypted bytes; recipients decrypt locally. Sharing needs the recipient's public key and a way to deliver the locator and keybox.

The owner's signing key establishes which events belong to the authoritative stream. Readers verify that history and can respond through their own signed streams. Decryption and authorship have distinct cryptographic purposes.

A logical name combines an owner identity with path segments. Its signed binding records an immutable target and update history. Private bindings travel inside encrypted streams. The [naming design](urls-vs-upspin-naming.md) describes URL syntax and endpoint resolution.

Retention is a service commitment: a principal pays for or supplies storage and pins encrypted objects. Keyboxes provide readable access; providers manage disk space and computing resources.

## Computation over shared data

An evaluator is another principal with a public encryption key. Alice makes a keybox for a dedicated input stream containing the recipe and necessary inputs. The evaluator decrypts it, runs the fixed operators, and encrypts the result into a job output stream.

Alice supplies the output stream header and keyboxes for herself, her recovery key, and the evaluator. The evaluator returns a signed result artifact; Alice verifies and appends it to her authoritative output stream. Alice can then share that stream with Bob. Selecting a dedicated input stream makes the information disclosed to the evaluator concrete.

Alice can also decrypt and evaluate on her own device. The [Operator Trees design](remote-operator-trees.md) describes the byte operators and retention contract.

## Acceptance scenarios

1. Create an identity locally, export and verify recovery material, then retain a first encrypted stream through self-service storage or a self-hosted deployment.
2. Alice presents a Mickey Mouse name and drawing. Bob signs his chosen statement; Carol inspects the statement and Bob's identity.
3. With synthetic documents, Alice shares evidence privately with Bob and publishes their chosen presentation and endorsement.
4. Update a presentation, issue a new endorsement, and withdraw an earlier statement while preserving verifiable history.
5. Alice makes a keybox for Bob. He reads retained events and a later event. Carol obtains the same ciphertext and Bob's keybox but cannot decrypt either.
6. Share a dedicated input stream with an evaluator, obtain an encrypted result containing `CDEF`, and share that result stream with Bob.
7. Verify that a saved recovery private key opens a fixture keybox and that every newly owned stream includes a recovery keybox.
8. Move a service through a signed locator update and verify the same stream history at the new provider.
9. Restart storage and verify histories, keyboxes, attestation status, encrypted objects, and retention commitments.

The [protocol requirements](concord-protocol-requirements.md) specify validation, limitations, and future work.

[Remote Operator Trees](remote-operator-trees.md) · [Naming design](urls-vs-upspin-naming.md)
