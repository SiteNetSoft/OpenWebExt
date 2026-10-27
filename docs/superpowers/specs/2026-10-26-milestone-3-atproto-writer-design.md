# Milestone 3: AT Protocol writer — Design

Date: 2026-10-26 · Status: Draft for approval (builder check done; sign-in mode decided by the user) · Builds on: `Quarkus Open Web Extensions — Architecture Spec.md`
(Tier 1 outline rows `quarkus-did` and `multikey-java`; Tier 2 outline "quarkus-atproto"; Milestone 3)

## Intent

Milestone 3 gives a Quarkus app the AT Protocol *writer* role: sign a user in with their Bluesky / AT Protocol
account through AT Protocol OAuth, then write records to that user's own PDS. QuarkPress is the first consumer: an
author publishes a blog post and it appears as a `site.standard.document` in their repository, visible to every
Standard.site reader.

The spec defines the milestone as:

> `multikey-java`, `quarkus-did`, `atproto-syntax`, identity, lexicon, xrpc, oauth client, and the Standard.site
> lexicon pack. Done when an app signs a user in with AT Protocol OAuth and writes a `site.standard.document` record
> to their PDS.

This document fills in what the spec leaves as an outline. **(my call)** marks a decision made here, not one the
spec dictates; the user can overturn any of them. Facts about AT Protocol and Quarkus come from research done on
2026-10-26 against primary sources (atproto.com specs, the live bsky.social authorization-server metadata, the
Standard.site lexicon records published under `did:plc:re3ebnp5v7ffagz6rb6xfei4`, and the Quarkus 3.40.1.1 source).

## Goals

1. An app with `quarkus-atproto-oauth` and `atproto-lexicons-standard-site` lets a user type their handle, approve
   access at their own authorization server, return as an authenticated `SecurityIdentity` whose principal is their
   DID, and publish a `site.standard.document` with one call.
2. Every building block is usable alone: key encodings without AT Protocol, DID resolution without AT Protocol,
   syntax types without Quarkus, the XRPC client with any auth, Lexicon codegen for any lexicon pack.
3. Records are validated against their Lexicon before they leave the app.
4. The spec's conventions hold: blocking + Mutiny APIs, `@ConfigMapping`, `@DefaultBean` SPIs, safe-fetch for every
   outbound call, Dev UI card, Antora docs, `quarkus-skill.md`, CI with a native job.

## Non-goals

- The authorization-server side of OAuth, the XRPC server, the PDS, sync/firehose, repo signing: later milestones.
- `atproto-lexicons-bsky` (posting to Bluesky timelines): not needed for the acceptance criterion; Milestone 4 or 5.
- App-password (`com.atproto.server.createSession`) authentication: Bluesky discourages it for apps **(my call: out)**.
- A persistent (Hibernate) OAuth session store: spec places `quarkus-atproto-hibernate` in Milestone 6; this
  milestone ships the SPI and an in-memory default **(my call)**.
- Runtime Lexicon resolution over the network (`_lexicon` DNS + `com.atproto.lexicon.schema` records): packs are
  pinned snapshots.
- PLC operations (creating or updating `did:plc` identities), handle changes.
- Docker Dev Services (no Docker on the build host; same deferral as milestones 1–2).

## Anti-goals

- Logging access or refresh tokens, DPoP private keys, client assertions or authorization codes.
- Accepting a session whose issuer, `sub` DID or PDS does not survive the mandatory post-token checks.
- Assuming OpenID Connect: there is no ID token and no UserInfo in AT Protocol OAuth.
- Blocking the Vert.x event loop in the login, callback or XRPC paths.

## Constraints

- Java 21, Quarkus 3.40.1, groupId `org.sitenetsoft.quarkus`, version `0.1.0-SNAPSHOT`, Apache 2.0.
- Host rules: Maven `-T 8`, one heavy build at a time (check for another Maven Java process first), no local
  native-image; CI's native job is the native check and needs Mandrel 25.
- Tier 2 depends on Tier 1 snapshots from the local repository until the first Tier 1 release (same as Milestone 2).
- Packages: `org.sitenetsoft.multikey`, `org.sitenetsoft.quarkus.did`, `org.sitenetsoft.atproto.syntax`,
  `org.sitenetsoft.atproto.lexicon.codegen`, `org.sitenetsoft.quarkus.atproto.{identity,lexicon,xrpc,oauth}`.

## What `quarkus-oidc` covers (the spec asked to measure this first)

Measured against the Quarkus 3.40.1.1 source:

| AT Protocol OAuth needs | `quarkus-oidc` 3.40 |
| --- | --- |
| PKCE S256 | Supported |
| PAR (required) | Supported |
| `private_key_jwt`, ES256 | Supported |
| Per-user issuer discovered at login (handle → DID → PDS → protected resource → AS) | Awkward: tenants are pre-configured per issuer |
| Callback `iss` check (RFC 9207, required) | Not supported |
| **Client-side DPoP** on PAR, token, refresh and every PDS call, per-session key, `ath`, per-server nonces with retry | **Not supported** (only resource-server verification) |
| No ID token; DID arrives as `sub` in the token response | Not supported without custom work (the DID never reaches the identity) |

**Decision:** `quarkus-atproto-oauth` implements the client flow itself and integrates with Quarkus Security only
(its own `HttpAuthenticationMechanism` and `IdentityProvider`). The missing DPoP support alone rules out building on
`quarkus-oidc`, and what would be reused (PKCE, PAR, a JWT assertion) is small.

## Repository layout

Three new git repos under `OpenWebExt/`, ignored by the root repo like the existing ones, pushed to
`github.com/SiteNetSoft` by the user:

```
multikey-java/                    tier 1, plain library; artifact multikey
quarkus-did/                      tier 1; quarkus-did, quarkus-did-deployment
quarkus-atproto/                  tier 2 train; parent quarkus-atproto-parent
  atproto-syntax/                 plain
  atproto-lexicon-codegen/        plain (generator library, used by the Quarkus code generator)
  identity/runtime|deployment     quarkus-atproto-identity
  lexicon/runtime|deployment      quarkus-atproto-lexicon (runtime validation + CodeGenProvider)
  xrpc/runtime|deployment         quarkus-atproto-xrpc (client only this milestone)
  oauth/runtime|deployment        quarkus-atproto-oauth (client only this milestone)
  lexicons/core                   atproto-lexicons-core (com.atproto.* subset)   (my call)
  lexicons/standard-site          atproto-lexicons-standard-site
  test/                           quarkus-atproto-test (fake PDS, fake authorization server, fake PLC)
  bom/, integration-tests/, docs/, .github/workflows/build.yml
```

`atproto-lexicons-core` is not in the spec's list **(my call)**: every writer needs `com.atproto.repo.*` and
`com.atproto.label.defs`, and Standard.site references `com.atproto.repo.strongRef`, so the core lexicons live in
their own pack instead of being copied into each pack.

## Module 1: `multikey-java` (tier 1, plain)

Key types and encodings: PEM, JWK, Multikey, `did:key`; RSA, Ed25519, P-256, secp256k1.

- `Multikey`: encode/decode `publicKeyMultibase` (multicodec varint + compressed point, base58btc `z…`); codecs
  `0xE7` secp256k1-pub (`E7 01`), `0x1200` p256-pub (`80 24`), `0xED` ed25519-pub. `DidKey`: `did:key:<multikey>`.
- `Multibase` (base58btc, base64url), `Varint`, `EcPoints` (compress/decompress for P-256 and secp256k1 in pure Java:
  square root mod p; the JDK cannot do this).
- `PublicKeys`/`PrivateKeys`: to/from PEM and JWK (`kty` RSA, EC P-256, EC secp256k1, OKP Ed25519); JWK thumbprint
  (RFC 7638), needed for DPoP `jkt`.
- `Signatures`: ES256 and ES256K compact (r‖s) sign/verify, **low-S** normalisation and enforcement (AT Protocol
  requires it for repo commits and labels); Ed25519; RSA. P-256 and Ed25519 use the JDK.
- **secp256k1 crypto (my call):** the JDK removed secp256k1 in JDK 16, so ES256K sign/verify and secp256k1
  `java.security` key objects use BouncyCastle's lightweight API (no JCA provider registration), declared as an
  `<optional>` dependency. Without BouncyCastle on the classpath, encoding/decoding and point maths still work and
  ES256K operations throw a clear error naming the dependency. Nothing in this milestone's writer path signs or
  verifies with secp256k1, so native-image support for BouncyCastle is deferred to the milestone that verifies repos.
- Test vectors from the AT Protocol cryptography spec and the W3C did:key test suite.

Overlap: `http-signatures-core` has its own PEM/JWK parsing. Moving it onto `multikey-java` is a later refactor, not
part of this milestone **(my call)**.

## Module 2: `quarkus-did` (tier 1)

Resolves DIDs to DID documents. Resolution only: handle verification is AT Protocol's job (spec).

- `DidDocument` (Jackson): `id`, `alsoKnownAs`, `verificationMethod` (`id`, `type`, `controller`,
  `publicKeyMultibase`, `publicKeyJwk`), `service` (`id`, `type`, `serviceEndpoint`), unknown properties kept. Both
  relative (`#atproto`) and absolute ids are accepted; helpers `verificationMethod(fragment)`, `service(fragment)`.
- `DidResolver` / `ReactiveDidResolver`: `resolve(String did)` → `DidDocument`; errors `DidNotFound`,
  `DidDeactivated`, `InvalidDidDocument`, `DidResolutionFailed`. The document `id` must equal the requested DID.
- `DidMethod` SPI (`String method()`, `Uni<DidDocument> resolve(String did)`); built-ins:
  - `plc`: `GET {quarkus.did.plc.url}/{did}` (default `https://plc.directory`); 404 → not found; 410 → deactivated.
  - `web`: `https://{host}/.well-known/did.json` for host-only DIDs, `https://{host}/{path}/did.json` for path DIDs
    (W3C did:web); percent-decoded port allowed.
  - `key`: built locally from the multikey, no network.
- Fetches go through `quarkus-safe-fetch`. Caffeine cache (`quarkus.did.cache-ttl`, default `5m`, below the 10-minute
  ceiling AT Protocol OAuth sets). Dev UI: a resolver box.
- Config `quarkus.did.*`: `plc.url`, `cache-ttl`, `methods` (enabled set, default `plc,web,key`), `max-document-size`
  (default `256K`).

## Module 3: `atproto-syntax` (plain)

Validated value types following the AT Protocol syntax specs exactly: `Did` (any method syntactically; `isBlessed()`
for plc/web), `Handle` (lowercased; disallowed TLDs rejected; `handle.invalid` constant), `Nsid` (authority +
name), `Tid` (parse/generate; monotonic clock + random clock id), `RecordKey` (+ `RecordKeyType` tid/nsid/literal/any),
`AtUri` (authority DID or handle, optional collection and rkey; no query/fragment/port), `AtIdentifier` (DID or
handle), `Datetime` (validates the AT Protocol profile of RFC 3339 and **keeps the original string** for round
trips), `Cid` (string form, syntax only), `Language`. Each has `parse`, `isValid`, `toString`, Jackson
(de)serialisers in an `AtprotoSyntaxModule`. Test vectors: the `interop-test-files` syntax fixtures from the
bluesky-social repo.

## Module 4: `quarkus-atproto-identity`

- `IdentityResolver` / `ReactiveIdentityResolver`:
  - `resolveHandle(Handle) → Did`: DNS TXT `_atproto.<handle>` (`did=…`, several different values = failure) and
    HTTPS `https://<handle>/.well-known/atproto-did` in parallel; first success wins; on disagreement DNS wins.
  - `resolve(AtIdentifier) → AtprotoIdentity(did, handle-or-invalid, pds URL, signing key multikey)`, with the
    **bidirectional check**: the DID document's first valid `alsoKnownAs` `at://` handle must resolve back to the
    same DID, else the handle is `handle.invalid`.
  - `pds` from service `#atproto_pds` of type `AtprotoPersonalDataServer` (https scheme + host + optional port only).
- DNS goes through a `TxtResolver` SPI; the default uses the Quarkus-managed Vert.x DNS client (servers from
  `quarkus.atproto.identity.dns.servers`, default system). Tests and apps can replace it **(my call)**.
- Cache 5 minutes for both directions (OAuth requires under 10). Dev UI: identity lookup.

## Module 5: Lexicons — `atproto-lexicon-codegen`, `quarkus-atproto-lexicon`, packs

**Packs** are jars holding Lexicon JSON under `META-INF/atproto/lexicons/<nsid path>.json` **and the classes
generated from it** **(my call)**. The pack's own build runs the generator (the plain `atproto-lexicon-codegen`
library via its CLI), so each NSID's Java types exist exactly once, in the pack that owns the NSID. If every consuming
module generated pack types itself, two modules depending on the same pack (an app plus a plugin jar, or
`quarkus-atproto-xrpc` plus the app) would emit the same `com.atproto.repo.CreateRecord` and collide. This keeps the
spec's intent ("added as dependencies, picked up by codegen": the app's own Lexicons that reference pack NSIDs are
generated against the pack's classes) without duplicates. `quarkus-atproto-xrpc` uses the generated core types from
`atproto-lexicons-core` internally.

- `atproto-lexicons-core`: `com.atproto.repo.{createRecord,putRecord,getRecord,deleteRecord,uploadBlob,describeRepo,
  listRecords,strongRef,defs}`, `com.atproto.identity.resolveHandle`, `com.atproto.label.defs`,
  `com.atproto.server.describeServer`, copied from `bluesky-social/atproto` at a pinned commit recorded in the pack
  README.
- `atproto-lexicons-standard-site`: `site.standard.{publication,document,graph.subscription,graph.recommend,
  theme.basic,theme.color,authFull,authSocial}` as published in `com.atproto.lexicon.schema` records under
  `did:plc:re3ebnp5v7ffagz6rb6xfei4` on 2026-10-26; the README records the fetch commands so the snapshot can be
  refreshed. `authFull`/`authSocial` are permission sets: shipped for `include:` scopes, not generated into classes.

**Codegen** (`atproto-lexicon-codegen`, plain, used by a Quarkus `CodeGenProvider` in
`quarkus-atproto-lexicon-deployment` so generation runs in the app's `generate-code` phase like gRPC):

- Inputs: the app's own Lexicon files in `src/main/lexicons/`. Dependency packs are read for **resolving refs only**
  (their types already exist in the pack jar, located through a `META-INF/atproto/lexicons/index.json` listing NSID →
  generated class); an app Lexicon that refers to `site.standard.document` gets the pack's class. Config
  `quarkus.atproto.lexicon.codegen.{include,exclude}` (NSID globs) and `package-prefix`. A dependency that ships
  Lexicon JSON **without** generated classes (a third-party pack) is generated into the app only when listed in
  `quarkus.atproto.lexicon.codegen.generate-from-dependencies` (explicit, so duplicates cannot happen silently).
- Mapping **(my call)**: package = the NSID authority segments in order (`site.standard.document` →
  package `site.standard`, `com.atproto.repo.createRecord` → `com.atproto.repo`), optional prefix
  `quarkus.atproto.lexicon.codegen.package-prefix`; class = PascalCase of the name segment (`Document`,
  `CreateRecord`); non-main defs become nested types (`Document.Contributor`).
  - `record`/`object` → Java `record` with Jackson annotations; optional properties nullable; `$type` written for
    records and union members, omitted for plain object refs (spec rule).
  - `union` → sealed interface with one record per ref plus `Unknown(JsonNode)` for open unions.
  - `token` → `String` constants; `knownValues` → constants on the owning type.
  - `string` formats → the `atproto-syntax` types; `datetime` → `Datetime`; `blob` → `BlobRef`; `cid-link` → `CidLink`;
    `bytes` → `byte[]` (base64 `$bytes` wire form); `unknown` → `JsonNode`.
  - `query`/`procedure` → a typed client per NSID namespace (`ComAtprotoRepo.createRecord(CreateRecord.Input)`)
    built on the generic XRPC client (Module 6).
  - `subscription` and `permission-set` defs are skipped with an INFO line.
- Constraints (lengths, graphemes, enums, required) are **not** enforced in constructors; the validator does that.

**Runtime validation** (`quarkus-atproto-lexicon`):

- `LexiconRegistry`: every Lexicon file on the classpath, discovered at build time (resource list recorded, native
  resources registered; no runtime classpath scanning).
- `LexiconValidator.validate(Nsid collection, JsonNode record)` → `List<ValidationProblem>`: types, required,
  `minLength`/`maxLength` (UTF-8 bytes), `minGraphemes`/`maxGraphemes` (JDK `BreakIterator`, extended grapheme
  clusters since JDK 20), enum/const, string formats via `atproto-syntax`, `blob` `accept`/`maxSize`, closed vs open
  unions (unknown `$type` in an open union passes), nested refs.
- Modes as in the spec: explicit true, explicit false, optimistic (validate if the Lexicon is known; default).

## Module 6: `quarkus-atproto-xrpc` (client)

- `XrpcClient` / `ReactiveXrpcClient`: `query(URI service, Nsid nsid, Map<String,?> params, Class<T> out)`,
  `procedure(URI service, Nsid nsid, Object input, Class<T> out)`, `procedure(..., byte[] body, String contentType, …)`
  for blobs. Path `/xrpc/<nsid>`; JSON in and out; non-JSON outputs as bytes.
- `XrpcAuth` SPI: `Uni<XrpcRequest> authorize(XrpcRequest)` and `boolean retry(XrpcResponse)`; the OAuth module
  supplies the DPoP implementation; without auth calls are anonymous.
- Errors: `XrpcException(status, error, message)` from the standard `{error, message}` body; rate-limit headers
  (`ratelimit-*`) exposed on the exception.
- Transport: `quarkus-safe-fetch` (`send`), so SSRF guards apply to user-supplied PDS hosts. XRPC calls do not follow
  redirects for `POST` (safe-fetch rule); a `GET` redirect to another host is refused (Milestone 2 rule).
- `AtprotoRecords` / `ReactiveAtprotoRecords` (the writer API, bound to a session; obtained either from the
  request-scoped `AtprotoUser` or, outside any HTTP request (scheduled or queued publishing), from
  `AtprotoSessions.records(Did)` (Module 7)): `create(collection, record)`
  (server-generated TID, or a client `Tid`), `put(collection, rkey, record, swapRecord)`, `get`, `delete`,
  `uploadBlob(bytes, mimeType)` → `BlobRef`. Records are validated (optimistic mode) before sending; the result
  carries `AtUri`, CID and `validationStatus`.

## Module 7: `quarkus-atproto-oauth` (client)

### Client identity

- **Confidential client (production default):** `client_id` = `{client-base-url}{metadata-path}`, served by the
  extension with `client_id`, `client_name`, `client_uri`, `redirect_uris`, `grant_types`
  (`authorization_code`, `refresh_token`), `response_types` (`code`), `scope`, `dpop_bound_access_tokens: true`,
  `application_type: web`, `token_endpoint_auth_method: private_key_jwt`, `token_endpoint_auth_signing_alg: ES256`,
  `jwks_uri` (also served). Signing key: an ES256 key from `quarkus.atproto.oauth.signing-key.location` (PEM or JWK);
  several keys supported for rotation (removing a key revokes its sessions, per spec).
- **Localhost development client (automatic in dev mode when `client-base-url` is unset or localhost):**
  `client_id=http://localhost?redirect_uri=http://127.0.0.1:<port>/…&scope=…`, public client, no key. This lets a
  developer sign in with a real Bluesky account from their laptop with no hosting **(my call)**.

### Flow

1. `POST {login-path}` with form field `handle=<handle-or-did>` (amended 2026-10-27, see "Amendments" below: same-origin
   POST only, GET → 405): resolve via Module 4 → PDS → GET
   `/.well-known/oauth-protected-resource` (exactly one `authorization_servers` entry) → GET
   `/.well-known/oauth-authorization-server`; check `issuer` equals the fetched origin and the required flags
   (`require_pushed_authorization_requests`, `authorization_response_iss_parameter_supported`,
   `client_id_metadata_document_supported`, ES256 in `dpop_signing_alg_values_supported`). Also accepts a bare PDS or
   entryway host instead of a handle.
2. New per-session DPoP key (P-256), PKCE verifier (S256), random `state`; store a `PendingAuthorization` (state →
   verifier, DPoP key, expected issuer, expected DID, post-login target) with a 10-minute TTL.
3. PAR to the AS with DPoP (nonce handshake: on `use_dpop_nonce` retry once with the `DPoP-Nonce` value), plus the
   client assertion for confidential clients, `login_hint` = handle; redirect the browser with `client_id` and
   `request_uri` only.
4. `GET {callback-path}?code&state&iss`: look up and delete the pending entry by `state`; `iss` must equal the expected
   issuer; token request with `code_verifier`, DPoP proof and client assertion.
5. **Mandatory post-token checks:** `scope` present and containing `atproto`; `sub` (a DID) equals the expected DID
   when the flow started from an account; otherwise resolve `sub`, take its PDS, fetch its protected-resource metadata
   and require the same issuer. Any failure aborts the login (no session) and logs the reason without tokens.
6. Store an `AtprotoSession` (DID, handle, PDS, issuer, granted scope from the response, tokens, expiry, DPoP key,
   AS and PDS nonces), set the session cookie, redirect to the target (default `/`).

### Sign-in modes (decided by the user on 2026-10-26: both)

`quarkus.atproto.oauth.mode` (runtime, default `login`):

- **`login`**: AT Protocol OAuth *is* the app's sign-in. After the callback the extension sets its session cookie and
  `AtprotoOAuthAuthenticationMechanism` produces the `SecurityIdentity` (principal = DID). Fits apps whose only
  accounts are Bluesky accounts.
- **`link`**: the user is already signed in through the app's own mechanism (form, OIDC, anything producing a
  `SecurityIdentity`). The login endpoint requires an authenticated identity (401 otherwise); after the callback the
  extension stores a **link** from the app principal's name to the DID and the session, sets no identity cookie of its
  own, and registers no authentication mechanism. `AtprotoAccountLinks` SPI (+ reactive; in-memory `@DefaultBean`):
  `link(appPrincipal, did, sessionId)`, `findByPrincipal(appPrincipal)` → list of linked DIDs, `unlink`. The request-
  scoped `AtprotoUser` resolves through the current principal's link (one linked account by default;
  `AtprotoUser.forDid(did)` when several are linked). `AtprotoLoggedIn` fires in both modes, carrying the app principal
  in link mode. QuarkPress uses link mode.

Both modes share everything after the callback: the session store, refresh, DPoP, `AtprotoSessions` for background
work, revocation (`POST {logout-path}` in login mode; `POST {unlink-path}`, default `/atproto/unlink`, in link mode).
The sample app and its integration test cover both modes (two test profiles: login mode alone; link mode behind a
test form login).

### Using the session

- `AtprotoOAuthAuthenticationMechanism` reads the session cookie → `SecurityIdentity` with principal = DID and
  attributes `handle`, `pds`, `scope`; `@Authenticated`, `@RolesAllowed` (role mapping SPI, default role `user`) and
  HTTP permissions work unchanged. A request-scoped `AtprotoUser` gives the app `did()`, `handle()`, `records()`
  (`AtprotoRecords` bound to the PDS with DPoP auth) and `xrpc()`.
- DPoP on PDS calls: `Authorization: DPoP <token>`, proof with `htm`, `htu`, `iat`, `jti`, `ath`, the **PDS's own**
  nonce; `use_dpop_nonce` → store and retry once; expired token → refresh, then retry once.
- **Refresh:** refresh tokens are single-use, so refreshes are serialised per session (in-process lock) and the store
  SPI updates with compare-and-set on the old refresh token, which decides which node's tokens are stored; the loser
  re-reads the stored session. Compare-and-set alone runs after the spend, so to stop two nodes from both spending
  one token a shared store implements the optional refresh lease (amended 2026-10-27, see "Amendments").
- `POST {logout-path}`: revoke at the AS `revocation_endpoint` when advertised (best effort), delete the session,
  clear the cookie.

### SPIs and storage

- `AtprotoSessionStore` (+ reactive): `save`, `find(sessionId)`, `findByDid`, `replaceIfRefreshToken(session,
  expectedRefreshToken)`, `delete`. `PendingAuthorizationStore`: `put(state, entry, ttl)`, `take(state)`.
  In-memory `@DefaultBean`s that WARN once in production (same pattern as Milestone 2). Implementations must encrypt
  DPoP private keys and tokens at rest (documented).
- `AtprotoSessions` (application-scoped, works without a request context): `find(Did)`, `records(Did)`,
  `xrpc(Did)`, `revoke(Did)`. Background jobs use it to publish later (a blog engine's scheduled post). Token refresh
  there follows the same per-session lock and compare-and-set as in requests; if the session is gone or its refresh
  token was revoked, the call fails with `AtprotoSessionExpired` so the job can ask the author to sign in again
  **(my call)**.
- `AtprotoRoleMapper` (DID/handle → roles), `AtprotoLoginListener` CDI events (`AtprotoLoggedIn`, `AtprotoLoggedOut`,
  `AtprotoSessionRevoked`) so an app can link the DID to its own account.
- Session cookie: encrypted (AES-GCM) value holding the session id only, key from
  `quarkus.atproto.oauth.session.encryption-key` (generated per run in dev, required in prod), `HttpOnly`, `Secure`
  outside dev, `SameSite=Lax`.

### JOSE

DPoP proofs and client assertions are compact JWS ES256, built in-house on the JDK (`SHA256withECDSAinP1363Format`)
with `multikey-java` for JWK and thumbprints **(my call)**: about a hundred lines, native-friendly, no
`smallrye-jwt-build` configuration pulled into apps. Tested against RFC 9449 and RFC 7638 vectors.

### Scopes

`quarkus.atproto.oauth.scope` default `atproto` (least privilege; apps add what they need). The acceptance app
requests `atproto repo:site.standard.publication?action=create&action=update
repo:site.standard.document?action=create&action=update blob:image/*` (it writes both record types).

Granular scopes are implemented in the reference PDS but their deployment on bsky.social could not be confirmed from
primary sources on 2026-10-26. **(my call)** Optional `quarkus.atproto.oauth.fallback-scope` (e.g.
`atproto transition:generic`): when the authorization server answers PAR with `invalid_scope`, the extension retries
PAR once with the fallback and logs at INFO; without it the login fails with that reason. The **granted** scope from
the token response is what the session records and what `AtprotoUser.scope()` reports. Permission sets (`include:`)
are passed through unchanged; the client does not expand them (the authorization server does).

`quarkus-atproto-oauth` offers `ScopeCheck.allows(grantedScope, "repo", collection, action)` and `AtprotoRecords`
refuses a write the granted scope does not cover with `AtprotoScopeException` before calling the PDS, so apps get a
clear error instead of a PDS 403 **(my call)**. The fakes parse and enforce `repo:` and `blob:` scopes (collection,
action, MIME) and treat `transition:generic` as full repo access; they reject `include:` with `invalid_scope` (enough
to exercise the fallback).

### Configuration (`quarkus.atproto.oauth.*`)

Build time: `enabled` (true). Runtime: `mode` (`login` | `link`, default `login`), `unlink-path` (`/atproto/unlink`),
`fallback-scope` (none), `client-base-url` (required in prod), `client-name`, `client-uri`,
`logo-uri`, `tos-uri`, `policy-uri`, `scope` (`atproto`), `login-path` (`/atproto/login`), `callback-path`
(`/atproto/oauth/callback`), `logout-path` (`/atproto/logout`), `metadata-path` (`/oauth-client-metadata.json`),
`jwks-path` (`/oauth/jwks.json`), `signing-key.location` (repeatable), `session.cookie-name` (`atproto-session`),
`session.encryption-key`, `session.max-age` (`14d`), `pending.ttl` (`10m`), `landing-page` (`/`),
`error-page` (none: 401 with a reason).

### Dev UI

Card: client type and `client_id`, the served client metadata, active sessions (DID, handle, PDS, scope, expiry; no
tokens), a "sign in" link.

## Tests

- **Fakes in `quarkus-atproto-test`** (plain Vert.x, like Milestone 2's `FakeFediverseServer`):
  - `FakePlcDirectory` (DID documents), handle well-known endpoints on fake hosts.
  - `FakeAuthorizationServer`: metadata, PAR (requires DPoP, issues and rotates nonces, validates PKCE and the client
    metadata document it fetches from the app, verifies `private_key_jwt`), `authorize` (auto-approves and redirects
    with `code`, `state`, `iss`), token and refresh (single-use refresh tokens, DPoP-bound tokens with `cnf.jkt`),
    revocation; knobs for wrong `iss`, wrong `sub`, missing `atproto` scope, nonce rotation.
  - `FakePds`: protected-resource metadata, `createRecord`/`putRecord`/`getRecord`/`deleteRecord`/`uploadBlob`/
    `describeRepo`, verifying DPoP proofs (`ath`, `htu`, `htm`, its own nonce) and the token binding; records what it
    stores.
- Unit/extension tests per module; syntax and crypto test vectors; codegen golden tests (generate from the
  Standard.site and core packs, compile, round-trip real records fetched on 2026-10-26).
- **Acceptance:** `integration-tests` sample app (login page, "publish" endpoint creating a `site.standard.publication`
  then a `site.standard.document`) and `@QuarkusIntegrationTest` driving handle → PAR → authorize → callback →
  publish against the fakes; CI runs it in JVM and native.
- **Live check (manual, documented):** in dev mode with the localhost client, sign in with a real Bluesky account and
  publish a test document; verify it with `com.atproto.repo.getRecord` on the user's PDS. No hosting needed.

## Acceptance criteria

- [ ] `multikey-java` passes the AT Protocol and did:key vectors; low-S enforced for ES256/ES256K.
- [ ] `quarkus-did` resolves `did:plc`, `did:web`, `did:key`; rejects id mismatches; works without AT Protocol.
- [ ] `atproto-syntax` passes the official syntax fixtures.
- [ ] Identity resolution does the bidirectional handle check and yields the PDS.
- [ ] Codegen produces compiling, round-tripping types for the core and Standard.site packs; the validator rejects
  invalid `site.standard.document` records (missing `title`, over-long `title` by graphemes, bad `publishedAt`).
- [ ] The OAuth flow passes against the fakes including nonce rotation, refresh, revocation and every post-token
  check; tokens never appear in logs.
- [ ] The sample app signs in and writes a `site.standard.document` to the fake PDS, in JVM and native (CI).
- [ ] The manual live check against bsky.social is documented; whether it was run is stated in the final report.

## Left to the builder

- Internal package layout below the public types named here; DNS client wiring details.
- The exact Jackson strategy for generated unions and `$type`, as long as the wire rules above hold.
- Caffeine sizes, log wording, Dev UI component details.
- How the fakes are structured internally, as long as their knobs cover the listed cases.

## Amendments (2026-10-27, made during implementation)

- **Sign-in starts only with a same-origin POST.** `POST {login-path}` requires `Sec-Fetch-Site: same-origin` or an
  `Origin` equal to the app's origin; `GET` answers 405. A GET (or cross-site POST) would let another site start a
  sign-in in the victim's browser and, with an attacker-run authorization server that skips consent, attach the
  attacker's account to the victim (login CSRF). A browser-binding cookie set at login is required at the callback.
- **Link mode binds the app principal.** The pending entry records the app principal at login and the callback
  requires the same principal. When the callback arrives without the app's own cookie (a `SameSite=Strict` cookie is
  withheld on the cross-site return from the authorization server), the callback first answers a same-origin bounce
  to itself, so the cookie is sent.
- **Refresh across nodes.** The session store SPI gains an optional refresh lease (default: single-node). Without
  it, two nodes can still spend the same refresh token and the authorization server may revoke the grant.
- **Record keys are validated** against the Lexicon's `key` type before `create(rkey)`/`put`.
