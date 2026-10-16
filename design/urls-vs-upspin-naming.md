# URL syntax and Upspin naming

Revised comparison and proposed Ocean naming policy, 16 October 2026

**An Upspin name can be expressed as a URL without losing its naming semantics.** The useful comparison is between URL syntax, the resolution rules selected by a scheme, and the transport used after resolution. Treating “URLs” and “Upspin names” as competing categories obscures this distinction.

## An Upspin name in URL form

Consider these two spellings:

```text
alice@example.com/notes/design
upspin://alice@example.com/notes/design
```

The second can mean exactly the first, with `upspin:` telling the reader which naming and resolution rules apply. This is a proposed representation for this discussion, not a claim that existing Upspin clients implement this spelling or its query extensions.

The mapping can be lossless with a specified encoding and parser. One detail matters: a generic URI parser separates `alice` as user information and `example.com` as host. An Upspin-aware handler must reconstruct the owner name and must not interpret Alice as HTTP login information or automatically connect to that domain. Escaping and normalization must preserve Upspin's name semantics. [URI authority and scheme syntax](https://www.rfc-editor.org/rfc/rfc3986.html#section-3)

That parsing detail does not invalidate the representation. It means that generic parsing supplies components, while the scheme supplies their meaning. A different spelling could avoid user-information handling if library compatibility warrants it; that is an encoding decision, not a different namespace.

## What URL syntax adds

A scheme tag makes the naming system explicit. Query syntax also provides a place for useful selectors:

```text
upspin://alice@example.com/notes/design?version=<shortcode-hash>
upspin://alice@example.com/notes/design?sha=<sha-hash>
```

The angle-bracket values are placeholders. These could identify a particular revision or require particular bytes while retaining a recognizable owner and path. They are not inherent meanings of `version` or `sha`: the scheme must define them. URI query syntax does not itself prescribe key/value parameters or revision lookup. [URI query component](https://www.rfc-editor.org/rfc/rfc3986.html#section-3.4)

This is an extension to bare path naming, not an ability exclusive to HTTPS. The same selector convention could be carried in an Upspin-style URL, a Concord URL, or an HTTPS gateway URL.

### Proposed selector semantics

| Form | Proposed meaning | Required behavior |
|---|---|---|
| No selector | Resolve the current authorized binding at this owner and path | Verify current binding and authority |
| `version=<hash>` | Resolve an exact historical signed binding record for this owner and path | Verify its record digest, signature, and historical signing authority |
| `sha=<digest>` | Resolve the current binding, but require its resulting blob bytes to match this SHA-256 digest | Fail if the current binding produces different bytes |
| Both | Resolve that historical binding and verify its resulting bytes | Both constraints must hold |

These are proposed Ocean conventions, not documented Upspin features. In particular, `sha` alone is an integrity constraint here, not a promise to search every prior version. A different policy could use the hash to select historical content, but would need an explicit lookup and ownership contract.

Use full hashes in durable machine references. A short version code can be an interactive abbreviation only when it resolves uniquely within the owner's path history; expand and display the full digest before storing it. Ambiguity, missing history, and mismatches fail explicitly. Never silently fall back to the latest version.

For `sha`, define the bytes precisely: the stored blob bytes returned by the byte API, or the materialized output of a recipe, before any client rendering. For encrypted blobs this means ciphertext, not plaintext. Recipe identity remains a separate address. A digest does not prove authorship or grant permission; historical lookup still requires current read authorization.

Selectors do not create retention. A historical binding and its target must actually have been preserved. Supporting `version` therefore requires an authorized history lookup and an explicit retention policy beyond the current-binding API. Until those exist, a client must reject the selector as unsupported rather than ignore it.

## HTTPS and Upspin choose different resolution models

An HTTPS URL selects HTTP semantics over a secured connection and an origin authority. With a domain-name host, normal resolution relies on DNS; HTTPS can also address an IP literal, so DNS is not mandatory for every URL. Proxies and alternative routing can affect connection establishment without changing the origin's identity. [HTTPS scheme and authoritative access](https://www.rfc-editor.org/rfc/rfc9110.html#name-https-uri-scheme)

An Upspin name starts with a user identity. The client discovers that user's directory through the key service, asks the directory for the named item, and obtains storage locations. The name itself does not tell the client to contact the email-shaped suffix as a DNS host. [Upspin architecture](https://upspin.io/doc/arch.html)

Upspin nevertheless has protocols and infrastructure. Its interfaces contain service endpoints with transport information, and its security documentation describes HTTPS RPC. Those discovered endpoints may use DNS. The distinction is that the logical name does not fix the network protocol or server address; it is not a claim that deployed Upspin operates without either. [Upspin endpoint interface](https://pkg.go.dev/upspin.io/upspin#Endpoint) · [Upspin server management](https://upspin.io/doc/security.html#server-management)

| Layer | HTTPS URL | Proposed Upspin URL representation |
|---|---|---|
| Syntax | Scheme, authority, path, optional query | The same general URL structure |
| Resolution anchor | Web origin | Owner identity and directory discovery |
| Domain-looking component | Ordinarily identifies the origin host | Part of the Upspin user name |
| Transport selected by the scheme | HTTP over a secured connection | Upspin resolution; discovered endpoints supply transport |
| Revision or hash selectors | Application-defined | Scheme-defined extension, if adopted |
| Permission | Application authorization | Upspin authorization; URL spelling adds no permission |

## Pros and cons of each resolution model

### HTTPS origin-based resolution

1. **Pro: broad interoperability.** Browsers and web tooling already know how to open an HTTPS link. A custom naming scheme needs a handler or gateway.
2. **Pro: a directly usable access mechanism.** The identifier names an origin and specifies how to interact with it. Applications can still return signed records or immutable content.
3. **Pro: names can outlive deployments.** Keeping the same domain and routing allows the infrastructure behind a URL to change. Persistent naming is possible with deliberate stewardship. [W3C guidance on stable URIs](https://www.w3.org/Provider/Style/URI)
4. **Con: continuity depends on origin control.** A provider-owned hostname is a dependency. Changing that hostname usually needs redirects or another discovery mechanism to preserve existing references.
5. **Con: personal identity is additional machinery.** DNS and HTTPS authentication do not, by themselves, establish that a resource belongs to a particular Concord principal.

### Upspin owner-based resolution

1. **Pro: storage location is indirect.** A stable owner/path can resolve to different directory or storage endpoints without rewriting the logical name.
2. **Pro: one naming contract can span providers.** The client follows a common discovery model instead of treating each provider's URL layout as the ownership model. [Upspin overview](https://upspin.io/doc/overview.html)
3. **Pro: URL representation preserves these properties.** Adding `upspin:` and well-defined query selectors need not bind the name to HTTP, a particular host, or DNS resolution of the owner's suffix.
4. **Con: discovery requires its own trust and availability model.** The documented Upspin ecosystem uses a central key service. Replacing DNS-based origin discovery does not eliminate infrastructure dependencies. [Upspin key management](https://upspin.io/doc/security.html#key-management)
5. **Con: interoperability needs integration.** A syntactically valid URL does not make a browser understand its scheme. Handler registration, escaping, normalization, and gateway behavior still need specification.
6. **Con: paths and selectors do not solve object continuity alone.** Renames change paths. History must be retained, and an immutable content hash is different from an enduring object identity.

Neither model wins merely by using URL punctuation. The important choice is what anchors resolution and who can change the binding.

## Implications for the first Concord design

Keep these layers separate:

| Thing | First-version representation |
|---|---|
| Principal | Concord genesis-record fingerprint |
| Logical name | Principal ID and structured path segments |
| Selection constraint | Optional version record digest and/or expected output digest |
| Immutable target | Blob or recipe address |
| Service locator | Signed service descriptor containing an HTTPS endpoint |

A Concord-aware URL could encode the logical name and selectors, just as the illustrative `upspin:` URL does. The exact Concord scheme spelling is undecided. HTTPS remains the initial wire transport and can provide a browser gateway; it need not be the canonical identity-bearing name format.

For example, `https://gateway.example/n/<principal-id>/notes/design?sha=<digest>` names a gateway serving the logical request. A verifying client checks the principal, signed binding, selectors, and resulting bytes. An ordinary browser relies on the gateway's presentation unless verification is provided there. The gateway host is not the owner identity.

Retain the first-version path grammar used by the Concord proposal: structured ASCII segments matching `[A-Za-z0-9_-]{1,64}`, at most 16 segments, with case-sensitive comparison. Reject empty segments, dot segments, embedded separators, and alternate percent-encoded spellings at the API boundary. Display labels may be Unicode. These restrictions apply to our PoC paths, not to Upspin names in general.

Keep selectors separate from paths and credentials. Define SHA-256 explicitly, reject duplicate or unknown selector parameters, and do not place bearer grants in URLs. Resolve the selected, verified binding to an immutable target before creating a recipe. Record that immutable target in the recipe, not a mutable name or gateway URL.

The initial API still resolves current bindings; revision selectors remain proposed until historical resolution and retention are specified. This revision changes the comparison and naming direction, not the implementation claims of the identity prototype.

**Revised direction:** URL syntax is a suitable container for owner-based names. Adopt identity-led resolution, use explicit selectors for revisions and byte integrity, and let discovered endpoints choose the transport. HTTPS is the first transport and gateway option, not a substitute for the logical namespace.

[First Concord implementation](concord-v1.md) · [Operator Trees design](remote-operator-trees.md) · [Upspin comparison](operator-trees-vs-upspin.md)
