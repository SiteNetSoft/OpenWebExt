# Quarkus Extensions for ActivityPub, AT Protocol and Web Standards — Spec

Oct 19, 2026 · @JN · Status: Draft v0.1

## Overview

This family of Quarkus extensions lets an app federate over ActivityPub, AT Protocol, or both, or use any single underlying web standard on its own. Version 0.1 fixes the architecture for every tier and fully specifies the first two extensions: `quarkus-webfinger` and `quarkus-http-signatures`.

**Goals**

- Full support of both protocols: every role each protocol defines, not only the common subset.
- Every web standard is usable alone, with no protocol or social concept in its API.
- Every protocol is usable alone, for any domain: social, git forges, events, publishing.
- A thin protocol-neutral layer covers the common social case: publish local content to both networks and collect replies from both.
- Every module follows Quarkus extension best practices (see Quarkus extension practices).
- Easy to build on: everything an app may want to change is a CDI bean, an event or a config property.
- QuarkPress is the first consumer and the reference integration.

**What full support covers**

- ActivityPub: server-to-server federation and the client-to-server API.
- AT Protocol: the OAuth client and every service role: Personal Data Server (PDS), relay, feed generator, labeler, and the building blocks of an AppView.
- Cross-protocol bridging, as its own optional tier 3 extension, built once both protocols are complete.

**Outside this family**

- End-user interfaces. Apps build their own; extensions ship Dev UI pages only.
- A home-grown job engine. Delivery runs on a `DeliveryExecutor` SPI, with adapters for what the Quarkus ecosystem already provides.

## Architecture

Modules sit in three tiers, and dependencies only point down. A developer picks an entry point at any tier: one standard, one protocol, or both protocols behind one API.

&#91;embedded content: module tiers · 3 tiers, dependencies point down\]

Adapters are the only modules that know two protocols at once; everything in tier 1 is usable with no protocol at all.

**Dependency rules**

1. A module may depend on modules in its own tier or lower, never higher.
2. A tier 1 API never names a higher-tier concept. WebFinger knows resources and links, not actors. HTTP Signatures knows key ids, not actors.
3. A tier 2 API never names a domain concept. ActivityPub knows activities and objects, not posts or comments.
4. Higher tiers plug into lower ones only through published SPIs: CDI beans implementing an interface the lower module defines.
5. Integration with another extension (Quarkus REST, REST Client, Security, Micrometer) is a conditional dependency, active only when that extension is present.
6. Plain libraries carry no Quarkus dependency, so they also work in plain Java, Kotlin and tests.

The tier 3 adapters follow rule 5. Each lives in the `quarkus-social` repo and activates only when its protocol extension is on the classpath, so the protocols never depend on the social layer.

## Repositories and artifacts

Tier 1 gets one repo per standard with its own version; each protocol gets one multi-module repo with a BOM; the social layer gets one repo. All start under github.com/SiteNetSoft and move to Quarkiverse once stable.

| Repo | Tier | Artifacts | Versioning |
| --- | --- | --- | --- |
| `quarkus-webfinger` | 1 | `quarkus-webfinger`, `quarkus-webfinger-deployment` | Own |
| `quarkus-http-signatures` | 1 | `http-signatures-core` (plain), `quarkus-http-signatures`, `-deployment` | Own |
| `quarkus-jsonld` | 1 | `quarkus-jsonld`, `-deployment` | Own |
| `quarkus-did` | 1 | `quarkus-did`, `-deployment` | Own |
| `quarkus-safe-fetch` | 1 | `quarkus-safe-fetch`, `-deployment` | Own |
| `ipld-java` | 1 | `ipld-cid`, `ipld-dagcbor`, `ipld-car` (all plain) | Own |
| `multikey-java` | 1 | `multikey` (plain) | Own |
| `quarkus-activitypub` | 2 | `activitystreams-model`, `quarkus-nodeinfo`, `quarkus-activitypub`, `quarkus-activitypub-c2s`, `quarkus-activitypub-hibernate`, BOM | One train |
| `quarkus-atproto` | 2 | `atproto-syntax`, `quarkus-atproto-identity`, `-lexicon`, `-xrpc`, `-oauth`, `-sync`, `-repo`, `-pds`, `-relay`, `-feedgen`, `-labeler`, `-hibernate`, `quarkus-atproto`, lexicon packs, BOM | One train |
| `quarkus-social` | 3 | `quarkus-social`, adapters, `quarkus-social-bridge`, `quarkus-social-hibernate`, `quarkus-social-test` | Own |

**Naming**

- No family brand. Each extension stands on its own name.
- Quarkus extensions: `quarkus-<name>` plus `quarkus-<name>-deployment`.
- Plain libraries: no `quarkus-` prefix.
- Config prefix: `quarkus.<name>.*`, for example `quarkus.webfinger.client.timeout`.
- groupId `org.sitenetsoft.quarkus` and Java packages `org.sitenetsoft.quarkus.<name>` until Quarkiverse approves an extension.
- On approval, that extension moves to `io.quarkiverse.<name>`, with Maven relocation POMs and a migration note so existing users get a clear path.

**Versioning**

- Semantic versioning; 0.x until the API survives QuarkPress integration.
- Each repo states the minimum supported Quarkus LTS in its README and CI matrix.
- Protocol repos depend on released tier 1 versions, never snapshots.

## Conventions for every module

Every module follows the same baseline, API shape and extension-point pattern, so learning one teaches the rest.

**Baseline**

- Java 21. Kotlin users get the same API; no Kotlin-only features in 1.0.
- Build against the current Quarkus LTS, [3.40, released 2026-09-30](https://quarkus.io/releases). CI also tests the latest release and tracks the upcoming Quarkus 4.0.
- License: Apache 2.0.
- Native image is a release requirement: CI builds and tests every extension in native mode.

**API style: blocking and Mutiny**

- Every request/response operation has a blocking form, safe on virtual threads, and a Mutiny form.
- Naming: `FooClient` is blocking; `ReactiveFooClient` returns `Uni`. Both are CDI beans backed by one implementation.
- Streams (AT Protocol event streams, delivery progress) are Mutiny `Multi` only.
- SPIs a user implements come in both forms too. The extension adapts whichever one the user provides.

**JSON and HTTP**

- Jackson everywhere. Jakarta JSON-P appears only inside `quarkus-jsonld`, because its processor needs it.
- Server endpoints register as Vert.x routes, so no module forces Quarkus REST on the app.
- Outbound calls use the Vert.x HTTP client through `quarkus-safe-fetch`.
- Quarkus REST and REST Client integrations (filters, annotations) are conditional dependencies.

**Extension points**

- An SPI is a Java interface; apps implement it as a CDI bean. The full set of hook-in patterns is in Quarkus extension practices.
- Defaults are `@DefaultBean`, so any app bean replaces them.
- Where several implementations make sense, the extension injects `@All List<T>`, which ArC sorts with higher `@Priority` first.
- Decisions go through SPIs (they return a value); notifications go through CDI events the app `@Observes`.

**Configuration**

- Typed `@ConfigMapping` interfaces, with build-time and runtime properties kept separate.
- Secure defaults: HTTPS only, timeouts and response-size limits on every outbound call.
- Dev mode may relax HTTPS-only for local testing, never prod.

**Observability and security**

- Micrometer metrics and OpenTelemetry spans when those extensions are present.
- Never log secrets, private keys, signature values or bearer tokens.
- Outbound fetches to loopback, link-local and private ranges are refused by default.

**Developer experience**

- Each extension ships a Dev UI card: config in effect, registered SPI beans, and a tester (for example, a WebFinger lookup box).
- Each repo ships test utilities and published spec test vectors where they exist.
- Docs in Antora format, ready for docs.quarkiverse.io.

## Quarkus extension practices

Every extension is reviewed against the [Quarkus extension maturity matrix](https://quarkus.io/version/main/guides/extension-maturity-matrix/) before each release. The rules below are how this family meets it.

### How apps hook in

Every extension offers the same six hook-in patterns, so a developer who learns one extension knows them all.

| An app wants to | It does this | Example |
| --- | --- | --- |
| Replace a default | Provide a bean of the SPI type; the extension's `@DefaultBean` steps aside | Its own `KeyResolver` backed by a database |
| Add to the behavior | Provide another bean; all run, highest `@Priority` first | A second `WebFingerResolver` that adds an OIDC issuer link |
| React to something | `@Observes` (or `@ObservesAsync`) a CDI event | `void onFollow(@Observes Follow f)` |
| Wrap a component | A CDI `@Decorator` on the SPI interface | Audit logging around `DeliveryExecutor` |
| Configure named instances | Config keys under a name, injected with `@Identifier("name")` | Two signing keys for two webhook partners |
| Switch a feature off | Build-time `quarkus.<name>.enabled=false` removes it from the build | No WebFinger endpoint in a client-only app |

### Structure and metadata

- Every extension has a runtime and a deployment module, built with `quarkus-extension-maven-plugin`.
- The `quarkus-extension-processor` annotation processor runs in every module; it is [required for all extension modules](https://raw.githubusercontent.com/quarkusio/quarkus/d58d0eb741f12b091e7d693db885f3f17a4156c8/.agents/skills/working-with-config/SKILL.md).
- `quarkus-extension.yaml` is complete: name, description, categories, keywords, guide link and status (`experimental`, then `preview`, then `stable`).
- Each extension declares a capability, so others can detect it, and consumes others only through conditional dependencies.
- Each extension ships a `quarkus-skill.md`, which Quarkus now supports for [giving AI coding agents patterns and pitfalls](https://quarkus.io/version/3.40/guides/extension-maturity-matrix/).

### Build time first

- All discovery happens in build steps over the Jandex index: `@SignatureRequired` endpoints, resolver beans, Lexicon schema files. No runtime classpath scanning.
- Native registration (reflection, resources, runtime-initialized classes) goes through build items, never hand-written `native-image.properties`.
- SPI beans looked up programmatically are marked unremovable, so ArC's unused-bean removal never drops an app's implementation.
- Config-defined objects (static keys, named clients) become synthetic beans.
- Recorders only wire runtime objects. No network calls or heavy work in static init.

### Configuration

- `@ConfigMapping` interfaces with `@ConfigRoot(phase = …)`. Legacy config classes [cannot be built since Quarkus 3.25](https://quarkus.io/blog/legacy-config-classes-sunsetting/).
- Defaults are declared with `@WithDefault`, never in code, so generated config docs show them.
- Build-time and runtime properties live in separate interfaces.

### Threading and shared resources

- Never block the Vert.x event loop. Blocking APIs run on worker or virtual threads; Mutiny APIs are safe on the event loop.
- Background work runs on the virtual-thread executor managed by Quarkus, as the maturity matrix recommends.
- Extensions reuse the Quarkus-managed Vert.x instance, the `quarkus-jackson` `ObjectMapper` (extended through `ObjectMapperCustomizer`) and the Quarkus TLS registry. They never create their own.
- The `FooClient` / `ReactiveFooClient` naming follows existing Quarkus clients such as the Redis client.

### HTTP and security

- `/.well-known/*` routes register at the server root, not under the `/q` management path. A configured `quarkus.http.root-path` triggers a build warning, because well-known paths must sit at the domain root.
- Signature verification plugs in as a Quarkus `HttpAuthenticationMechanism` plus `IdentityProvider`, so `@Authenticated`, `@RolesAllowed` and HTTP path policies work unchanged.

### Delivery and background jobs

- Protocols hand outbound work to a `DeliveryExecutor` SPI.
- Default: a database-backed outbox processed by Quarkus Scheduler on virtual threads, suitable for a single node.
- Adapters activate when present: `quarkus-quartz` for persistent, clustered jobs, and the [JobRunr Quarkus extension](https://www.jobrunr.io/en/documentation/configuration/quarkus) for apps that already use it.

### Observability

- Micrometer metrics and OpenTelemetry spans when those extensions are present.
- SmallRye Health checks where there is an external dependency, for example the delivery backlog or a relay connection.
- One JBoss Logging category per extension, for example `org.sitenetsoft.quarkus.webfinger`.

### Developer joy

- A Dev UI card and pages for every extension. Dev UI reads `quarkus-extension.yaml` from the runtime module to build the card.
- Dev Services where a protocol needs a peer: a local PDS and PLC directory for AT Protocol, and a lightweight Fediverse server container for ActivityPub interop tests. Started only when Docker is available, never in prod.
- Codestarts, so `quarkus create app -x webfinger` produces a working example.
- Continuous testing and live reload work with every extension.

### Testing and release

- Unit tests in the deployment module use the Quarkus extension test harness; an `integration-tests` module runs the same tests in native mode.
- CI matrix: current LTS, latest release, and the Quarkus 4.0 preview once published.
- Each repo publishes a `-test` artifact with JUnit extensions and fakes for app authors.

## Tier 1 spec: quarkus-webfinger

`quarkus-webfinger` serves and queries WebFinger (RFC 7033) and host-meta (RFC 6415). The app supplies resource data through resolvers; the extension handles the endpoint, the wire format, merging, caching and safe fetching.

### Scope

- Server: `GET /.well-known/webfinger` with a required `resource` parameter and repeatable `rel` filters.
- Server, optional: `GET /.well-known/host-meta` (XRD) and `/.well-known/host-meta.json`, with an LRDD template pointing at the WebFinger endpoint.
- Client: lookups with the host-meta LRDD fallback for servers that host WebFinger elsewhere.
- Value types: `acct:` URIs, the JSON Resource Descriptor (JRD) and its links.
- Out of scope: what any link means. ActivityPub, OpenID Connect or others add meaning through resolvers.

### Model

```java
record JsonResourceDescriptor(URI subject, List<URI> aliases,
                              Map<URI, String> properties, List<Link> links) {}

record Link(String rel, String type, URI href, String template,
            Map<String, String> titles, Map<URI, String> properties) {}

record AcctUri(String user, String host) {   // acct:user@host
  static AcctUri parse(String value);         // percent-decoding, IDN to ASCII
  URI toUri();
}
```

`template` is not in RFC 7033 but appears in the wild (host-meta LRDD, remote follow). The model keeps it so clients don't lose it.

### Server SPI

```java
public interface WebFingerResolver {
  Optional<JsonResourceDescriptor> resolve(WebFingerRequest request);
}

record WebFingerRequest(URI resource, Set<String> rels, String host) {}
```

- The app registers any number of resolvers as CDI beans; a `ReactiveWebFingerResolver` variant returns `Uni`.
- All resolvers run, in `@Priority` order. Results for the same subject are merged: links and aliases are concatenated, and on a property clash the higher priority wins.
- This lets ActivityPub add an actor link and an OpenID Connect resolver add an issuer link for the same `acct:` address.
- No resolver returns data → `404`. Missing or malformed `resource` → `400`.
- The server applies `rel` filtering after the merge, so resolvers can ignore it.
- Responses use `application/jrd+json` and send `Access-Control-Allow-Origin: *` by default.
- If no resolver bean exists at build time, the endpoint is not registered and the build logs why.

A built-in OpenID Connect resolver is included and off by default. Configured with an issuer URL and a domain list, it answers the `http://openid.net/specs/connect/1.0/issuer` rel for OIDC discovery.

### Client API

```java
public interface WebFingerClient {
  JsonResourceDescriptor lookup(URI resource, Set<String> rels);
  JsonResourceDescriptor lookup(AcctUri account, Set<String> rels);
}
// ReactiveWebFingerClient: the same methods returning Uni<JsonResourceDescriptor>
```

Resolution steps:

1. Query `https://<host>/.well-known/webfinger?resource=…&rel=…`.
2. On `404`, and when fallback is enabled, fetch host-meta and follow its LRDD template once.
3. Cache the host's template, or the fact that it has none, so later misses skip host-meta.
4. Cache successful results using `Cache-Control`, capped by a configured maximum TTL.

Failures are typed: `WebFingerNotFoundException`, `WebFingerInvalidResponseException`, `WebFingerUnreachableException`. All outbound calls go through `quarkus-safe-fetch`.

### Configuration

| Property | Default | Purpose |
| --- | --- | --- |
| `quarkus.webfinger.server.enabled` | `true` | Serve the endpoint when resolvers exist |
| `quarkus.webfinger.server.cors` | `true` | Send `Access-Control-Allow-Origin: *` |
| `quarkus.webfinger.server.host-meta.enabled` | `false` | Also serve host-meta (XRD and JSON) |
| `quarkus.webfinger.oidc.enabled` | `false` | Built-in OIDC issuer resolver |
| `quarkus.webfinger.oidc.issuer` | none | Issuer URL to advertise |
| `quarkus.webfinger.oidc.domains` | none | Domains whose `acct:` addresses it answers |
| `quarkus.webfinger.client.timeout` | `10s` | Per-request timeout |
| `quarkus.webfinger.client.max-response-size` | `64K` | Larger responses are rejected |
| `quarkus.webfinger.client.host-meta-fallback` | `true` | Try LRDD on `404` |
| `quarkus.webfinger.client.cache.max-entries` | `1000` | Result cache size |
| `quarkus.webfinger.client.cache.max-ttl` | `1h` | Upper bound on cached results |

### Dev UI

The card lists registered resolvers, previews the server's answer for any resource, and runs live client lookups.

### Acceptance criteria

- [ ] The RFC 7033 examples round-trip through the model and serializer.
- [ ] Two resolvers answering the same subject produce one merged JRD.
- [ ] A Mastodon instance resolves an account served by the extension, once ActivityPub adds its resolver.
- [ ] The client resolves accounts on Mastodon and on a server that only advertises LRDD.
- [ ] Lookups to a private or loopback address are refused in prod mode.
- [ ] Native build passes all tests.

## Tier 1 spec: quarkus-http-signatures

`quarkus-http-signatures` signs and verifies HTTP messages under both RFC 9421 and the older Cavage draft 12, with body digests. Both schemes are required because the Fediverse still runs mostly on Cavage, while [Mastodon 4.5+ accepts both](https://docs.rs/actpub-httpsig). The same module serves webhooks and API-to-API authentication with no ActivityPub involved.

### Modules

- `http-signatures-core` (plain library): signature base construction, signing, verification, structured-field parsing, digests. No HTTP framework dependency.
- `quarkus-http-signatures` (extension): Vert.x, Quarkus REST, REST Client and Quarkus Security integration, config-based keys, Dev UI.

### Schemes and algorithms

| Scheme | Headers | Algorithms |
| --- | --- | --- |
| RFC 9421 | `Signature-Input`, `Signature` | `rsa-v1_5-sha256`, `rsa-pss-sha512`, `ecdsa-p256-sha256`, `ecdsa-p384-sha384`, `ed25519`, `hmac-sha256` |
| Cavage draft 12 | `Signature` | `rsa-sha256`, `hs2019` (algorithm taken from the key), `ed25519` |
| Digest, current (RFC 9530) | `Content-Digest` | `sha-256`, `sha-512` |
| Digest, legacy (RFC 3230) | `Digest` | `SHA-256` |

The verifier picks the scheme automatically: a `Signature-Input` header means RFC 9421, a lone `Signature` header means Cavage.

### Core API

```java
interface HttpMessageView {          // adapters: Vert.x, JAX-RS, java.net.http
  String method(); URI targetUri(); Headers headers();
  Optional<BodyDigest> body();
}

record SignatureSpec(Scheme scheme, String keyId, Algorithm alg,
                     List<Component> covered, Instant created,
                     Optional<Instant> expires, Optional<String> nonce,
                     Optional<String> tag) {}

interface Signer   { SignedHeaders sign(HttpMessageView msg, SignatureSpec spec, SigningKey key); }
interface Verifier { VerificationResult verify(HttpMessageView msg, VerificationPolicy policy); }

record VerificationResult(Scheme scheme, String keyId, Algorithm alg,
                          List<Component> covered, Instant created,
                          VerificationKey key) {}
```

### SPIs

- `KeyResolver`: maps a key id to a `VerificationKey`. ActivityPub's implementation fetches the actor's public key; a webhook user's reads it from config. Results are cached; on a verification failure the extension refetches once, which covers key rotation.
- `SigningKeyProvider`: chooses the `SigningKey` and key id for an outbound request. The default reads static keys (PEM or JWK files) from config.
- `NonceStore`: optional replay protection. The default is in-memory with a TTL; apps running several instances provide a shared one.
- Each SPI has a Mutiny variant.

### Verifying incoming requests

- Annotate a Quarkus REST endpoint with `@SignatureRequired`, or attach the Vert.x route handler to a path pattern.
- Optional attributes tighten the policy: `@SignatureRequired(components = {"@method", "@target-uri", "content-digest"})`.
- A verified request exposes a request-scoped `SignatureContext`. It also becomes a Quarkus `SecurityIdentity` whose principal is the key id, so `@RolesAllowed` and permission checks apply.
- Failure returns `401` with no detail in prod; Dev mode logs the signature base for debugging.

### Signing outgoing requests

- REST Client: register `HttpSignatureClientFilter`, or annotate the client interface with `@SignRequests`.
- Plain Vert.x and `java.net.http`: a small helper signs a request builder.
- Bodies get a `Content-Digest` (and `Digest` for Cavage) before signing.
- Per-client settings live under `quarkus.http-signatures.clients."<name>".*`.

**Scheme negotiation (double knocking).** Outbound requests try the preferred scheme first. On `401` or `403` the client retries once with the other scheme and remembers the answer per host. This keeps old and new servers working during the transition.

### Security requirements

- RFC 9421 requests must cover `@method`, `@target-uri` (or `@authority` plus `@path`) and, with a body, `content-digest`.
- Cavage requests must sign `(request-target)`, `host` and `date`, plus `digest` with a body.
- The digest is checked against the actual body before the signature is accepted.
- `created` (or `Date`) must fall within the clock-skew window; `expires`, when present, is enforced.
- The declared algorithm must match the key type; any mismatch is rejected to prevent algorithm confusion.
- HMAC comparisons are constant-time. Signature values and keys never reach logs.

### Configuration

| Property | Default | Purpose |
| --- | --- | --- |
| `quarkus.http-signatures.verify.clock-skew` | `5m` | Allowed drift on `created` or `Date` |
| `quarkus.http-signatures.verify.max-age` | `1h` | Oldest signature accepted |
| `quarkus.http-signatures.verify.require-digest` | `true` | Reject bodies without a digest |
| `quarkus.http-signatures.verify.schemes` | `rfc9421,cavage` | Schemes accepted |
| `quarkus.http-signatures.verify.paths` | none | Path patterns for the Vert.x handler |
| `quarkus.http-signatures.sign.preferred-scheme` | `rfc9421` | First scheme tried outbound |
| `quarkus.http-signatures.sign.fallback` | `true` | Retry with the other scheme on 401/403 |
| `quarkus.http-signatures.keys."<id>".location` | none | PEM or JWK file for a static key |
| `quarkus.http-signatures.key-cache.ttl` | `1h` | Resolved-key cache lifetime |
| `quarkus.http-signatures.nonce.enabled` | `false` | Enable replay protection |

### Dev UI

The card shows the accepted schemes and policy, registered resolvers, and a tool that verifies a pasted request and prints its signature base.

### Acceptance criteria

- [ ] The RFC 9421 Appendix B test vectors sign and verify byte for byte.
- [ ] Signed requests from Mastodon verify under Cavage, and under RFC 9421 where Mastodon sends it.
- [ ] Requests signed by the extension are accepted by a Mastodon inbox.
- [ ] Double knocking falls back once and remembers the result per host.
- [ ] A tampered body, wrong algorithm, stale `created` or missing covered component each fail.
- [ ] A webhook example works with HMAC and no ActivityPub dependency.
- [ ] Native build passes all tests.

## Tier 1 outline: the other standards

The remaining tier 1 modules are scoped here and get their own detailed specs before implementation starts.

| Module | Purpose | What Quarkus adds | Uses outside social |
| --- | --- | --- | --- |
| `quarkus-safe-fetch` | Outbound HTTP with SSRF guard, timeouts, size limits, redirect limits | Config, metrics, one shared Vert.x client | Any app fetching user-supplied URLs |
| `quarkus-jsonld` | JSON-LD 1.1 via an existing processor (Titanium), plus a fast path for known contexts | Contexts downloaded and cached at build time, native registration | schema.org markup, Verifiable Credentials, Solid |
| `quarkus-did` | Resolve `did:web`, `did:key`, `did:plc` to DID documents; `DidMethod` SPI for more | Caching, config per method, Dev UI resolver | Verifiable Credentials, decentralized identity |
| `multikey-java` | Key types and encodings: PEM, JWK, Multikey, `did:key`; RSA, Ed25519, P-256, secp256k1 | None (plain library) | Any signing or key exchange |
| `ipld-java` | CID, DAG-CBOR (canonical), CAR read/write | None (plain library) | IPFS, content-addressed storage |

Two boundaries matter here:

- `quarkus-jsonld` makes full processing opt-in. Most federated JSON is handled as plain JSON with a known context; full expansion is for Linked Data signatures and unusual input.
- `quarkus-did` resolves documents only. Handle verification, which is specific to AT Protocol, stays in `quarkus-atproto-identity`.

## Tier 2 outline: the protocols

Each protocol extension gives full native access to its protocol for any domain. 1.0 covers every role: both halves of ActivityPub, and every AT Protocol service role.

### quarkus-activitypub

| Module | Responsibility |
| --- | --- |
| `activitystreams-model` | Activity Streams 2.0 vocabulary as Jackson types; tolerant parsing (single value or array, `as:Public` or full URI) |
| `quarkus-nodeinfo` | NodeInfo discovery and document, fed by an SPI |
| `quarkus-activitypub` | Server-to-server: actors, inbox and outbox, shared inbox, collections, delivery |
| `quarkus-activitypub-c2s` | Client-to-server: authenticated outbox posting, inbox reading, media upload, side effects of client activities |

- Plugs into tier 1: a `WebFingerResolver` maps `acct:` to actors; a `KeyResolver` fetches actor keys.
- Storage SPIs: `ActorStore`, `ObjectStore`, `FollowerStore`, `DeliveryStore`. Hibernate defaults ship in `quarkus-activitypub-hibernate`, so apps using only ActivityPub need nothing else.
- Incoming activities arrive as CDI events, for example `void onFollow(@Observes Follow f)`. Accepting or rejecting one is an SPI decision, not an event.
- Outbound delivery goes through the `DeliveryExecutor` SPI, with retries, backoff and per-host failure tracking.
- Remote objects are fetched through `quarkus-safe-fetch` and rejected when the object's id domain doesn't match the fetch URL.
- Client-to-server authentication reuses Quarkus OIDC and Security, so any Keycloak-compatible identity provider works.

### quarkus-atproto

| Module | Responsibility |
| --- | --- |
| `atproto-syntax` | DID, Handle, NSID, TID, Record Key, AT-URI as validated value types |
| `quarkus-atproto-identity` | Handle resolution (DNS TXT and well-known), bidirectional handle/DID check, built on `quarkus-did` |
| `quarkus-atproto-lexicon` | Build-time code generation from Lexicon schemas; runtime record validation |
| `quarkus-atproto-xrpc` | XRPC client and server, service-auth tokens |
| `quarkus-atproto-oauth` | AT Protocol OAuth: client profile (PAR, PKCE, DPoP) and the authorization-server side a PDS needs |
| `quarkus-atproto-sync` | Event stream and Jetstream consumers as Mutiny streams with stored cursors |
| `quarkus-atproto-repo` | Repository tree, commits, signing, CAR export, on `ipld-java` |
| `quarkus-atproto-pds` | Personal Data Server: accounts, repositories, blob storage SPI, PLC operations, event stream output |
| `quarkus-atproto-relay` | Relay: crawls PDSs, verifies commits, re-emits the network event stream |
| `quarkus-atproto-feedgen` | Feed generator: XRPC endpoints and a ranking SPI |
| `quarkus-atproto-labeler` | Labeler: label storage, signing, query and subscription endpoints |
| Lexicon packs | `atproto-lexicons-bsky`, `atproto-lexicons-standard-site`; added as dependencies, picked up by codegen |

- An AppView is app-specific, so there is no AppView module. Apps build one from `-sync`, `-lexicon` and `-xrpc`.
- Before writing `quarkus-atproto-oauth`, measure how much of the profile `quarkus-oidc` and `quarkus-oidc-client` already cover.
- Event-stream decoding sits behind its own interface, because the wire format may change for sharding.
- The PDS and relay keep all storage behind SPIs (repositories, blobs, accounts), with Hibernate defaults in `quarkus-atproto-hibernate`.

## Tier 3 outline: quarkus-social

`quarkus-social` is a thin, protocol-neutral API for publishing local content and collecting interactions. Its final shape is extracted after milestones 2 and 3, from what both protocols actually share.

```java
@Inject Social social;

PublishResult r = social.publish(Article.from(post));  // every active protocol
// r: Published(activitypub), Skipped(atproto, "visibility not supported")

void onReply(@Observes SocialReply reply) { … }        // from either network
```

- **Adapters.** One per protocol, in this repo, each a conditional dependency that activates when its protocol extension is present.
- **Per-protocol results.** `publish()` reports each protocol separately. It never silently drops content or widens visibility.
- **Capabilities.** Each adapter declares what it supports (followers-only visibility, edits, long-form content), and callers can query them.
- **Identity linking.** A local account links to an ActivityPub actor, an AT Protocol DID, both, or neither.
- **Native access stays open.** Apps inject `ActivityPub` or `AtprotoRepo` directly for anything the neutral API doesn't cover.
- **Storage.** SPIs in this repo; `quarkus-social-hibernate` covers the social layer's own stores; each protocol ships its own Hibernate module.
- **Bridge.** `quarkus-social-bridge` is optional and built last. It gives remote Mastodon users a presence on AT Protocol and remote Bluesky users one on ActivityPub, so people can follow and reply across networks. It needs identity mapping, content translation and moderation hooks on top of both complete protocols.

## Milestones

Seven milestones, each ending in something a developer outside this project can use. No dates are set yet; each milestone gates the next, and 1.0 ships after milestone 6.

1. **Tier 1 foundations.** `quarkus-safe-fetch`, `quarkus-webfinger`, `http-signatures-core` and `quarkus-http-signatures`.
   - Done when both detailed specs' acceptance criteria pass, including native builds and the HMAC webhook example.
2. **ActivityPub server-to-server.** `activitystreams-model`, `quarkus-nodeinfo`, `quarkus-activitypub`, `quarkus-activitypub-hibernate`.
   - Done when Mastodon can find an actor through WebFinger, follow it, and receive a signed `Create(Note)`.
3. **AT Protocol writer.** `multikey-java`, `quarkus-did`, `atproto-syntax`, identity, lexicon, xrpc, oauth client, and the Standard.site lexicon pack.
   - Done when an app signs a user in with AT Protocol OAuth and writes a `site.standard.document` record to their PDS.
4. **Social layer.** `quarkus-social` and both adapters, extracted from milestones 2 and 3.
   - Done when one `publish()` call reaches Mastodon and Bluesky, with per-protocol results.
5. **Streams and services.** `ipld-java`, `quarkus-atproto-sync`, `-feedgen`, `-labeler`; QuarkPress integration.
   - Done when a QuarkPress blog collects replies from both networks and a sample feed generator serves Bluesky.
6. **Full protocols (1.0).** `quarkus-activitypub-c2s`; `quarkus-atproto-repo`, `-pds`, `-relay`, `-hibernate`, and the OAuth authorization-server side.
   - Done when the official Bluesky app signs in to a Quarkus PDS, the network relay picks up its repository, and an ActivityPub client posts through the C2S API.
7. **Bridge.** `quarkus-social-bridge`.
   - Done when a Mastodon user follows and replies to a Bluesky user through a Quarkus bridge, and the reverse.

## Decisions

Six decisions are settled; three remain open and use the defaults shown until decided.

| Decision | Outcome | Status |
| --- | --- | --- |
| Scope | Full support of both protocols; the bridge follows as its own extension | Decided |
| Family name | None; each extension stands on its own name | Decided |
| groupId and packages | `org.sitenetsoft.quarkus` until Quarkiverse approval, then `io.quarkiverse.<name>` with relocation POMs | Decided |
| `quarkus-safe-fetch` | Its own extension, usable alone | Decided |
| API style | Blocking and Mutiny for every operation; streams Mutiny only | Decided |
| Quarkus baseline | 3.40 LTS; CI tracks latest and 4.0 | Decided |
| Tier 3 name | `quarkus-social` | Open |
| Verification max-age | `1h`; some servers sign hours-old requests on retries, while longer windows weaken replay protection | Open |
| Default delivery backend | Database outbox on Quarkus Scheduler; Quartz and JobRunr adapters optional | Open |

## Sources

- [Quarkus releases and LTS schedule](https://quarkus.io/releases)
- [Quarkus extension maturity matrix](https://quarkus.io/version/main/guides/extension-maturity-matrix/)
- [Sunsetting legacy config classes](https://quarkus.io/blog/legacy-config-classes-sunsetting/)
- [Quarkus config conventions for extensions](https://raw.githubusercontent.com/quarkusio/quarkus/d58d0eb741f12b091e7d693db885f3f17a4156c8/.agents/skills/working-with-config/SKILL.md)
- [ArC `@All` qualifier and priority ordering](https://javadoc.io/static/io.quarkus.arc/arc/3.8.0.CR1/io/quarkus/arc/All.html)
- [Quarkus conditional extension dependencies](https://quarkus.io/guides/conditional-extension-dependencies)
- [JobRunr Quarkus extension](https://www.jobrunr.io/en/documentation/configuration/quarkus)
- [actpub-httpsig: Cavage and RFC 9421 support in the Fediverse](https://docs.rs/actpub-httpsig)
