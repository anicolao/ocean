# First Concord identity service

Concord lets people create an identity, choose how to present themselves, and vouch for one another. Those identities also let people sign their work, use storage services, and share access to information.

The first implementation follows Alice as she creates an identity, stores a document, meets Bob, and receives an endorsement. Carol later uses that endorsement to help interpret Alice's work.

## Terminology

| Term | Meaning in this design |
|---|---|
| **Identity or principal** | An actor recognized by Concord through its stable identifier and authorized keys. A principal can represent a person, agent, or service. We usually say “identity” in the user flow and “principal” in protocol definitions. |
| **Owner** | A principal with control over a particular resource, including authority to grant access. Ownership is a role relative to that resource. |
| **Subject** | The principal whose identity claims a presentation or attestation concerns. |
| **Attestor** | The principal signing a statement that vouches for specified claims about a subject. |
| **Recipient** | The principal receiving a disclosure or access grant. |
| **Presentation** | A subject-signed selection of identity claims, an optional avatar, and chosen evidence references. |
| **Attestation** | An attestor's signed statement about specific claims in an exact presentation. |
| **Access grant** | An owner's signed authorization for a recipient to perform specified actions on a resource. |

Alice and Bob are both principals. Alice owns a document and grants Bob permission to read it. When Bob vouches for Alice, Alice is the subject and Bob is the attestor. Bob controls his attestation and can replace or withdraw it. The same principal can hold several roles, depending on the resource or interaction.

## The user flow

### 1. Create an identity and save something

Alice selects **Create identity**. Concord creates her keys locally, prepares recovery material, and helps her save a recovery copy. Her identity is ready to use.

She chooses a storage provider or connects her own server. The client shows the provider's capacity and terms, configures access, and lets her save her first document. Alice can return to it from another device she authorizes.

### 2. Choose how to present yourself

Alice creates a presentation with the display name **Mickey Mouse** and a drawing for an avatar. She previews a card showing exactly what someone else will see, then selects **Show QR code**.

She can also create a more detailed presentation containing her legal name, address, and selected supporting documents. Each presentation has its own audience and disclosure choices. Alice can use different presentations for different situations; they remain associated with the same underlying identity.

The QR code carries a small presentation or a reference to it. Large attachments are shared separately with the recipients Alice chooses.

### 3. Ask someone to vouch for you

Bob scans Alice's code. Concord shows her presentation and offers **Vouch for this identity**. Bob chooses the claims he wants to affirm and reviews the statement he will sign.

For the Mickey Mouse presentation, he might write:

> I know this person as Mickey Mouse and vouch for them as the holder of this identity.

For the detailed presentation, he might write:

> I inspected the presented passport and driver's licence, compared the holder to their photos, and checked that the utility bill matches the claimed address.

Bob records how he checked the claims. When confirming control of the identity, his client can ask Alice's device to sign a fresh challenge. He then signs the statement, creating an **attestation** attached to that exact presentation.

### 4. Choose what to share

Alice and Bob review the attestation and its intended audience. Alice might show Bob the supporting documents privately and share the resulting endorsement with a wider audience. Bob chooses the precise statement he is willing to stand behind.

The sharing preview lists the claims, evidence, attestation, and recipients separately. Alice can publish the agreed presentation and endorsement alongside her work, or share them with particular people.

### 5. Interpret someone else's identity

Carol opens something Alice has published and follows its identity attribution. She sees Alice's chosen presentation, Bob's statement, what Bob says he checked, when he signed it, and the source and freshness of its status.

Carol can inspect Bob's identity and disclosed attestations as well. She decides how much weight to place on his endorsement, based on her own relationships, the evidence available, and his record. The same presentation can attract several attestations, each making its own claim.

A pseudonymous identity with a long history of thoughtful endorsements and a documented legal identity give Carol different information. Concord presents the statements and their provenance so she can interpret both Alice and the people vouching for her.

### 6. Keep presentations and endorsements current

When Alice changes a presentation, Concord creates a new version. Bob can review it and issue a new attestation. His earlier statement remains attached to the version he originally saw.

Bob can set an expiry, replace a statement, or publish a signed withdrawal. Carol's view shows these changes and when their status was last checked. The history keeps the statements attributable to their authors.

### 7. Share access and manage devices

To let Bob read a private document or run a particular computation, Alice selects the resource, recipient, and permitted action, then signs an access grant. The sharing interface handles these permissions separately from the identity card and its endorsements.

Alice can add a phone, retire an old computer, rotate her controller key, or restore control using her recovery material. Her underlying identity remains the same. She can also move her records and names to another provider through a signed service update.

## What the first implementation contains

One deployment can host the following components, each with a distinct interface:

| Component | Responsibility |
|---|---|
| Identity client | Creates keys, manages devices and recovery, signs records |
| Identity record service | Publishes the identity's key history and current service locations |
| Presentation and attestation service | Stores disclosed identity cards, endorsements, replacements, and withdrawals |
| Naming service | Resolves owner-qualified names to signed bindings |
| Authority service | Checks device permissions and resource grants |
| Storage and evaluation service | Retains data and carries out authorized operations |

A person can create an identity locally and connect to a self-service provider or operate their own deployment. Providers manage their own capacity, billing, and resource policies. The client's signing keys stay under the person's control.

## Identity records and presentations

The stable principal ID is a fingerprint of a **genesis record** containing the initial controller public key, an offline recovery public key, a random nonce, and a format version. Operational updates form a signed sequence beneath that identity.

A public **control profile** contains active device certificates and service descriptors. Each descriptor identifies a service's role, principal, and HTTPS endpoint. Clients retain verified history and follow signed updates when service locations change.

A **presentation** is an immutable, subject-signed record containing the principal ID, presentation ID, selected claims, optional avatar, creation time, and selected evidence references. Display names belong to presentations; local address-book labels are another way people recognize their contacts.

## Attestations

An attestation records an attributable statement about selected claims in one presentation. It contains:

- The attestor and subject identities, attestation ID, and signing-key reference.
- The presentation digest and the specific claims being affirmed.
- The statement, verification method, and any disclosed evidence commitments.
- Issue time, optional expiry, and an optional reference to a statement it replaces.

Controllers or devices with an attestation-signing role can issue these records. Clients verify the signer and show the statement in context. Signed withdrawals and replacements refer back to the original record.

Presentations, evidence, and attestations have explicit disclosure choices. Optional evidence commitments allow a later recipient to check that disclosed evidence matches what was committed earlier. Private exchanges in the first prototype use an explicitly chosen channel; the service stores the records the participants choose to disclose.

## Names, permissions, and persistence

A logical name combines an owner identity with path segments. A signed binding records its immutable target and update history. The [naming design](urls-vs-upspin-naming.md) describes how URLs can carry that logical name and how clients resolve it to service endpoints.

An access grant identifies its issuer, recipient, service, exact resource, permitted actions, and expiry. The first grants are direct owner-to-recipient grants. Recipe grants identify the recipe and the dependencies the service may use. The owner retains the required inputs, and the recipient may retain an authorized result under their own storage quota.

The service checks current authority when accepting work and publishing its result. It preserves identity checkpoints, revocations, attestation withdrawals, grant state, and pin ownership as durable control records.

## Acceptance scenarios

1. A fresh client creates an identity, saves recovery material, configures storage, and retains a first document. Demonstrate both a self-service provider and a self-hosted deployment.
2. Alice presents a Mickey Mouse name and drawing by QR code. Bob chooses a statement, signs it, and Carol inspects both the statement and Bob's identity.
3. Using synthetic document fixtures, Alice shows Bob a legal-name and address presentation with supporting evidence. The sharing preview publishes their chosen summary and audience.
4. Alice updates her presentation. Bob signs the new version, then replaces or withdraws an earlier endorsement. Carol can inspect the history and status freshness.
5. Alice gives Bob an exact recipe-evaluation grant. He receives the expected `CDEF` result, with the permitted actions and retained inputs visible in the record.
6. Alice adds and retires devices, rotates keys, and restores control from recovery material. Her principal ID stays constant through each operation.
7. Alice moves her naming service. Bob follows the signed service update and resolves the same logical name.
8. Restart the service and verify that identity history, attestation status, permissions, and retained data remain available and consistent.

The [protocol and security requirements](concord-protocol-requirements.md) specify validation rules, failure behavior, trust boundaries, and tests for the implementation.

[Remote Operator Trees design](remote-operator-trees.md) · [Naming design](urls-vs-upspin-naming.md)
