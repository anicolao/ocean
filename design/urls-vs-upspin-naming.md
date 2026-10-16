# URLs compared with Upspin naming

Design comparison and proposed Ocean naming policy, 16 October 2026

**Use HTTPS URLs to reach services and share ordinary links; use Concord identities to establish who controls names and objects.** Upspin's useful contribution is the owner-rooted namespace and discovery protocol, rather than a particular spelling of paths. We can adopt that separation without requiring a new browser URL scheme.

## Compare the protocols, not just the strings

“URL” covers many schemes. This comparison concerns ordinary HTTPS URLs, such as `https://files.example/alice/notes/design`, versus an Upspin path such as `alice@example.com/notes/design`. URI syntax permits identifiers for abstract resources; a URL need not identify a physical machine or disk file. Its semantics depend on its scheme and service. [RFC 3986](https://www.rfc-editor.org/rfc/rfc3986)

Upspin resolves the user prefix through a key service, then resolves the path through that user's directory service, which supplies storage locations. The directory and store can live separately. [Upspin overview](https://upspin.io/doc/overview.html)

For HTTPS, the origin is defined by scheme, host, and port. HTTP supplies retrieval, authentication hooks, conditional requests, and redirects, but does not prescribe a personal directory or person-to-key mapping. [HTTP semantics](https://www.rfc-editor.org/rfc/rfc9110.html)

| Question | Ordinary HTTPS URL | Upspin path |
|---|---|---|
| What anchors resolution? | A web origin and that service's routing | A user record and that user's directory |
| What does the path mean? | Defined by the application | A hierarchical item name in a user's tree |
| Where are bytes stored? | Hidden behind the service's response | Discovered through directory metadata |
| What proves a person's authority? | Additional application protocol required | User keys and Upspin authorization mechanisms |
| Does the name fix the bytes? | Only with an additional immutable-content convention | Not for an ordinary mutable path |

These mappings summarize the protocols above. Neither string alone is a cryptographic identity, a retention promise, or permission to read.

## Pros of HTTPS URLs

1. **They fit existing interaction.** A person can use a normal web link and a service can present a document, sign-in flow, or download. Ocean can offer a browser entry point while a richer client verifies signed records.
2. **Hosting can change without changing the URL.** A domain can remain stable while the deployment behind it changes. Stable URLs are a service-design commitment, not something the web architecture forbids. [W3C guidance on persistent URIs](https://www.w3.org/Provider/Style/URI)
3. **One transport can carry several models.** An HTTPS response can be bytes, a signed name binding, or a recipe. Ocean need not invent a transport to introduce its identity model.
4. **Useful concurrency mechanisms exist.** Conditional requests and entity tags can protect updates against stale writers. These complement signature validation; they do not establish who was authorized to sign. [HTTP conditional requests](https://www.rfc-editor.org/rfc/rfc9110.html#name-conditional-requests)

## Cons of relying on HTTPS URLs as the only identity

1. **Changing the origin changes the ordinary address.** Moving from a provider-owned domain to another provider can break links unless the old endpoint continues to redirect or another resolution layer exists. Control of one's own domain helps, but domain continuity is still an operational obligation.
2. **HTTPS authenticates the origin, not automatically Alice.** A service must add a binding between a human or cryptographic principal and a resource. A provider account or path component named `alice` does not supply provider-independent ownership.
3. **Naming and authority vary between services.** Read access, editing rights, public links, and account recovery are application decisions. Cross-provider use requires a common protocol in addition to URLs.
4. **Links can expose more than intended.** Names and queries may reveal sensitive context; an application that puts a bearer credential into a link turns disclosure of that link into an authority leak. Ocean should use recipient-bound grants, not secret-bearing URLs.

These are consequences for our design, not claims that HTTPS applications cannot implement stronger identity or privacy.

## Pros of Upspin naming

1. **Names start with the owner.** The user namespace is separated from the directory and storage endpoints. Moving infrastructure can preserve the logical path if discovery records and data are migrated correctly.
2. **Discovery is specified.** Clients agree how to find a user's directory and then the stored content, rather than integrating a different naming API for every provider. [Upspin architecture](https://upspin.io/doc/arch.html)
3. **Sharing has common semantics.** Upspin defines named rights and groups through `Access` files. A client has a consistent model for asking which operation is allowed. [Upspin access control](https://upspin.io/doc/access_control.html)
4. **The namespace can expose more than static files.** Upspin describes services such as a camera image available through a name. A common name interface need not limit implementations to disk storage. [Upspin overview](https://upspin.io/doc/overview.html#examples)

## Cons of adopting Upspin naming unchanged

1. **An email-shaped name still needs a trust system.** The documented ecosystem uses a central key service. That is useful coordination, but differs from Concord's goal of identities established through locally controlled keys and direct introductions. [Upspin key management](https://upspin.io/doc/security.html#key-management)
2. **A path does not survive every object change.** A rename changes a path. Ocean needs an enduring object identifier if an object should keep its identity through renames and multiple views.
3. **The directory sees naming metadata.** Upspin protects content with client-side encryption, while its directory sees paths and sharing information. Owner-rooted naming is not itself metadata confidentiality. [Upspin security](https://upspin.io/doc/security.html)
4. **A new resolution protocol has a deployment cost.** Clients must understand it, or use a gateway. A gateway's URL is a way to reach the namespace, not necessarily the namespace's authoritative identity.

## Proposed first-version naming rules

Keep four different things explicit:

| Thing | Proposed representation | Changes when |
|---|---|---|
| Principal | Concord genesis-record fingerprint | A genuinely new identity is created |
| Logical name | Structured pair of principal ID and path segments | The owner renames the binding |
| Immutable target | Existing blob or recipe address | Its bytes or recipe change |
| Locator | HTTPS service URL from a signed profile | The provider or deployment changes |

Friendly labels such as “Alice” are local contact aliases, never globally authoritative identifiers. In the first version, names bind directly to immutable targets; a separate persistent object-history layer remains future work. Do not imply that a mutable name already implements Ocean's enduring objects.

An ordinary link can look like `https://gateway.example/n/<principal-id>/notes/design`. Its host selects a gateway. A Concord client verifies the owner identity and signed name binding before accepting its target. A browser without that verifier relies on the gateway's presentation and HTTPS security; it does not gain provider-independent verification merely by opening the link. No new URI scheme is required in this PoC.

For the first path grammar, use structured ASCII segments matching `[A-Za-z0-9_-]{1,64}`, at most 16 segments, with case-sensitive comparison. Reject empty segments, dot segments, embedded separators, and alternate percent-encoded spellings at the API boundary. Display labels may be Unicode, but are not lookup keys. This intentionally small grammar avoids promising Unicode equivalence or filesystem semantics before we define them.

Resolve a logical name to a verified immutable target before creating a recipe. Record that target, not the mutable name or gateway URL, in the recipe. Provider migration changes the signed service-location record; it does not rewrite the principal or recipe addresses. Existing gateway links can still fail when that gateway disappears, so clients also retain the logical name and verified discovery information.

**Decision:** adopt Upspin's separation of owner, directory, and storage; keep HTTPS as the transport and ordinary link format; make Concord's signed identity and name records the authority. The additional verification and discovery work is the price of independence from a single host.

[First Concord implementation](concord-v1.md) · [Operator Trees design](remote-operator-trees.md) · [Upspin comparison](operator-trees-vs-upspin.md)
