# Milestone 2: ActivityPub server-to-server — Design

Date: 2026-10-25 · Status: Draft for approval · Builds on: `Quarkus Open Web Extensions — Architecture Spec.md` (Tier 2 outline "quarkus-activitypub", Milestone 2)

## Intent

Milestone 2 turns the Tier 1 foundations (`quarkus-safe-fetch`, `quarkus-webfinger`, `quarkus-http-signatures`) into a working
ActivityPub server-to-server stack that any Quarkus app can add to federate: publish actors, accept follows, deliver signed
activities, and receive activities as CDI events. QuarkPress is the first consumer; the API therefore stays domain-neutral
(activities and objects, never posts or comments).

The architecture spec defines the milestone as:

> `activitystreams-model`, `quarkus-nodeinfo`, `quarkus-activitypub`, `quarkus-activitypub-hibernate`.
> Done when Mastodon can find an actor through WebFinger, follow it, and receive a signed `Create(Note)`.

This document fills in what the spec leaves as an outline. Everything marked **(my call)** is a decision made here, not one
the spec dictates; the user can overturn any of them.

## Goals

1. A Quarkus app with `quarkus-activitypub` and `quarkus-activitypub-hibernate` on the classpath, a `base-url` configured and
   one `ActorStore` row, is discoverable from Mastodon, followable, and can push a signed `Create(Note)` to its followers.
2. Every inbound activity reaches the app as a CDI event; every decision (accept a follow, trust a domain) is an SPI.
3. The Activity Streams model is a plain Jackson library usable without Quarkus.
4. NodeInfo works alone, with no ActivityPub dependency.
5. The spec's conventions hold throughout: Vert.x routes, blocking + Mutiny APIs, `@ConfigMapping`, `@DefaultBean` SPIs,
   secure defaults, Dev UI card, Antora docs, `quarkus-skill.md`, CI with a native job.

## Non-goals

- Client-to-server (`quarkus-activitypub-c2s`): Milestone 6.
- JSON-LD expansion/compaction and Linked Data signatures: handled as plain JSON with the known context (spec boundary for
  `quarkus-jsonld`, which does not exist yet).
- Any UI, timeline, or social semantics (likes counts, threads): Tier 3 and apps.
- Media upload/storage.
- Quartz and JobRunr `DeliveryExecutor` adapters: later, once the default outbox is proven.
- Dev Services Fediverse container and automated Mastodon interop tests: deferred, same as the Milestone 1 deferrals
  (no Docker-based interop on this host so far). A manual interop procedure is documented instead; GoToSocial is the
  recommended lightweight peer when Dev Services is built **(my call)**.

## Anti-goals

- Leaking a private key or signature value into logs or Dev UI.
- Fetching remote objects on behalf of unauthenticated callers or in response to unverified inbound activities
  (classic amplification vector: a `Delete` for an unknown actor must never trigger a fetch).
- Blocking the Vert.x event loop in inbox or delivery paths.
- Storing anything the app did not ask for: no silent timeline database. Stores hold what federation itself needs.

## Constraints

- Java 21, Quarkus 3.40.1, groupId `org.sitenetsoft.quarkus`, version `0.1.0-SNAPSHOT`, Apache 2.0.
- Depends on the Tier 1 repos at `0.1.0-SNAPSHOT` from the local Maven repository. The spec says protocol repos depend on
  released Tier 1 versions; nothing is released yet, so snapshots are used until the first Tier 1 tag **(my call)**.
- Host rules (CLAUDE.md): Maven `-T 8` max, one heavy build at a time, Surefire `-XX:TieredStopAtLevel=1`, no `native-image`
  locally.
- Packages `org.sitenetsoft.quarkus.activitypub`, `org.sitenetsoft.quarkus.nodeinfo`; the plain model library uses
  `org.sitenetsoft.activitystreams` (same pattern as `org.sitenetsoft.httpsignatures`).

## Repository layout

One new git repo `OpenWebExt/quarkus-activitypub/` (the spec's "one train" protocol repo), ignored by the root repo like
the Tier 1 ones.

```
quarkus-activitypub/                      parent: quarkus-activitypub-parent
  activitystreams-model/                  plain library (no Quarkus)
  nodeinfo/runtime, nodeinfo/deployment   quarkus-nodeinfo, quarkus-nodeinfo-deployment
  runtime/, deployment/                   quarkus-activitypub, quarkus-activitypub-deployment
  hibernate/runtime, hibernate/deployment quarkus-activitypub-hibernate, -deployment
  test/                                   quarkus-activitypub-test (FakeFediverseServer for app authors)
  bom/                                    quarkus-activitypub-bom
  integration-tests/                      a sample app run under QuarkusIntegrationTest; native profile in CI only
  docs/                                   Antora
  .github/workflows/build.yml
```

`quarkus-activitypub-c2s` is reserved as a future module, not created.

## Required Tier 1 changes

Two small changes to Tier 1 repos ship first, each committed in its own repo with tests.

### Fix: config signing keys must survive a second `SigningKeyProvider` (quarkus-http-signatures)

`ConfigSigningKeyProvider` is a `@DefaultBean`, and `RequestSigner` injects `@All List<SigningKeyProvider>`. ArC drops
default beans from an `@All` list as soon as any other bean of that type exists, so adding the ActivityPub provider would
silently disable config-based keys (webhooks) in the same app. **(my call)** `ConfigSigningKeyProvider` becomes an
ordinary `@ApplicationScoped` bean with `@Priority(0)` that returns `empty` when no configured key matches; the same
change applies to `ConfigKeyResolver` for symmetry. Behaviour for existing users is unchanged; a regression test adds a
second provider and checks both still answer.

### POST in quarkus-safe-fetch

`SafeFetchClient` only has `get`. Inbox delivery is a POST, and the spec requires all outbound calls to go through
`quarkus-safe-fetch`. **(my call)** Add to `quarkus-safe-fetch` (both blocking and reactive clients):

```java
FetchResponse send(String method, URI uri, byte[] body, FetchOptions options);
default FetchResponse post(URI uri, byte[] body, FetchOptions options)
```

Rules: same SSRF/size/timeout guards; redirects are followed for GET/HEAD only (a POST that answers 3xx returns the 3xx);
`FetchOptions.headers` carries the signature headers the caller computed. This ships as a `quarkus-safe-fetch` 0.1.0
change with its own tests, committed in that repo before the ActivityPub work starts.

## Module 1: `activitystreams-model`

Activity Streams 2.0 vocabulary as Jackson types, tolerant on input, canonical on output.

**Core types** (package `org.sitenetsoft.activitystreams`):

- `ASObject` — base class. Fields: `id` (URI), `type` (`List<String>`, first entry is the primary type), `context`
  (raw `JsonNode` under `@context`), and every common property (`name`, `nameMap`, `content`, `contentMap`, `summary`,
  `mediaType`, `published`, `updated`, `url`, `attributedTo`, `to`, `cc`, `bto`, `bcc`, `audience`, `inReplyTo`, `tag`,
  `attachment`, `icon`, `image`, `replies`, `likes`, `shares`, `sensitive`, `duration`, `startTime`, `endTime`).
  Unknown properties are kept in `Map<String, JsonNode> extra` (`@JsonAnySetter`/`@JsonAnyGetter`) so round-tripping
  never drops data.
- `ASLink` — `href`, `rel`, `mediaType`, `name`, `hreflang`, `height`, `width`, `preview`.
- `Activity extends ASObject` — `actor`, `object`, `target`, `result`, `origin`, `instrument`. `IntransitiveActivity`.
- Collections: `Collection`, `OrderedCollection`, `CollectionPage`, `OrderedCollectionPage` (`totalItems`, `first`, `last`,
  `current`, `next`, `prev`, `partOf`, `items`, `orderedItems`, `startIndex`).
- Actor types: `Person`, `Application`, `Group`, `Organization`, `Service`, sharing `ActorProperties`: `inbox`, `outbox`,
  `followers`, `following`, `liked`, `preferredUsername`, `endpoints` (`sharedInbox` etc.), `publicKey`,
  `manuallyApprovesFollowers`, `discoverable`, `featured`, `alsoKnownAs`, `movedTo`.
- Object types: `Note`, `Article`, `Document`, `Image`, `Video`, `Audio`, `Page`, `Event`, `Place`, `Profile`,
  `Tombstone` (`formerType`, `deleted`), `Relationship`.
- Link types: `Mention`, `Hashtag` (Mastodon extension; treated as a known type **(my call)**).
- Activity types: every AS2 activity (`Accept`, `Add`, `Announce`, `Arrive`, `Block`, `Create`, `Delete`, `Dislike`,
  `Flag`, `Follow`, `Ignore`, `Invite`, `Join`, `Leave`, `Like`, `Listen`, `Move`, `Offer`, `Question`, `Read`, `Reject`,
  `Remove`, `TentativeAccept`, `TentativeReject`, `Travel`, `Undo`, `Update`, `View`).
- Security vocabulary: `PublicKey` (`id`, `owner`, `publicKeyPem`), modelled as a type because every actor carries one.
- `Emoji` (Mastodon `toot:Emoji`) is parsed as a generic `ASObject` with type `Emoji`; no dedicated class in this milestone.

**Tolerant parsing**

- `ObjectOrLink`: sealed interface with `Ref(URI)` (a bare string) and `Embedded(ASObject)`; `uri()` returns the id in both
  cases. Every object-valued property uses it.
- Multi-valued properties are `List<...>` and accept a single value (`@JsonFormat(with = ACCEPT_SINGLE_VALUE_AS_ARRAY)` on
  each field, so the library never needs a global mapper feature).
- Audience values: `Audience.PUBLIC` matches `https://www.w3.org/ns/activitystreams#Public`, `as:Public` and `Public`.
  `ASObject.isPublic()` and `.audience()` helpers normalise them.
- Polymorphism: a custom `ASObjectDeserializer` reads `type`, looks the primary type up in `ASTypes` (a registry
  `Map<String, Class<? extends ASObject>>`, pre-filled with the vocabulary, open for apps), and falls back to `ASObject`
  for unknown types. Multi-type values (`["Person", "toot:Actor"]`) pick the first known type.
- `type` missing: treated as `ASObject` with an empty type list, never an error.

**Canonical output**

- `ASJson` helper: `ObjectMapper activityStreams()` for plain-Java users, and `ASModule` (a Jackson `Module`) for Quarkus,
  where `quarkus-activitypub` registers it through an `ObjectMapperCustomizer`.
- Output omits nulls and empty lists; `@context` on a top-level object defaults to
  `["https://www.w3.org/ns/activitystreams", "https://w3id.org/security/v1", {"toot": ..., "Hashtag": ..., "sensitive": ...,
  "manuallyApprovesFollowers": ..., "discoverable": ...}]` when none is set. Nested objects never emit `@context`.

**Tests**: round-trip fixtures from the ActivityPub spec examples and real Mastodon documents (actor, Note with Hashtag and
Mention, Create, Follow, Accept, Undo, Delete of an actor, Announce, Question), plus tolerance cases (single vs array,
string vs object, `as:Public` forms, unknown type).

## Module 2: `quarkus-nodeinfo`

Serves NodeInfo 2.0 and 2.1 (diaspora* NodeInfo protocol), fed by an SPI.

- Routes (Vert.x, server root): `/.well-known/nodeinfo` (discovery document linking both schemas) and `/nodeinfo/2.0`,
  `/nodeinfo/2.1`. Path for the documents configurable with `quarkus.nodeinfo.path` (default `/nodeinfo`) **(my call)**.
- `NodeInfo` record: `software(name, version, repository, homepage)`, `protocols`, `services(inbound, outbound)`,
  `openRegistrations`, `usage(users(total, activeMonth, activeHalfyear), localPosts, localComments)`, `metadata` map.
- SPI `NodeInfoProvider` / `ReactiveNodeInfoProvider`: returns the base document. `@DefaultBean` reads
  `quarkus.nodeinfo.software.*` (defaults: `quarkus.application.name` / `version`), `open-registrations` (default `false`),
  `metadata.*`.
- SPI `NodeInfoContributor` / `ReactiveNodeInfoContributor` (`@All`, by `@Priority`): each may add protocols, services,
  usage numbers and metadata on top of the base document. `quarkus-activitypub` ships one that adds `activitypub` to
  `protocols`. Apps add usage counts through one.
- Response cached for `quarkus.nodeinfo.cache-ttl` (default `5m`), because usage queries can be expensive.
- Build-time `quarkus.nodeinfo.enabled` removes the routes.
- Dev UI card: preview of both documents, list of contributors.

## Module 3: `quarkus-activitypub` (server-to-server)

### Concepts and naming

- **Local actor**: an actor this server owns. Supplied by the app through `ActorStore`. The extension never invents local
  actors except the optional **instance actor**.
- **Instance actor**: an `Application` actor at `{base-url}/actor` (`quarkus.activitypub.instance-actor.enabled=true`,
  username from `quarkus.activitypub.instance-actor.username`, default `instance`). It signs outbound fetches
  (authorized fetch) and `Delete`-broadcast handling. Its key pair is generated on first start and persisted through
  `ActorStore`.
- **Remote actor**: any actor fetched from another server; cached, never trusted beyond its own id's host.

### URL layout

`quarkus.activitypub.base-url` is required at runtime (`https://example.com`). Path templates (runtime config, defaults
shown) **(my call, Mastodon-compatible)**:

| Property | Default |
| --- | --- |
| `paths.actor` | `/users/{username}` |
| `paths.inbox` | `/users/{username}/inbox` |
| `paths.outbox` | `/users/{username}/outbox` |
| `paths.followers` | `/users/{username}/followers` |
| `paths.following` | `/users/{username}/following` |
| `paths.shared-inbox` | `/inbox` |
| `paths.objects` | `/objects/{id}` |
| `paths.activities` | `/activities/{id}` |

`ActivityPubUrls` bean builds and parses these. Templates must contain `{username}` or `{id}` exactly once; validated at
startup. A `quarkus.http.root-path` other than `/` logs the same warning WebFinger uses.

### Routes (Vert.x, registered by the deployment module when `quarkus.activitypub.server.enabled=true`)

| Route | Behaviour |
| --- | --- |
| `GET actor` | Content negotiation: `application/activity+json` or `application/ld+json; profile="…activitystreams"` → the actor document (type from `LocalActor.type`, `publicKey` with id `{actor}#main-key`, `endpoints.sharedInbox`). Any other `Accept` → `302` to `LocalActor.url` when set, else `406`. |
| `POST inbox`, `POST shared inbox` | Signature verification (below) → `401`. Body > `quarkus.http-signatures.verify.max-body-size` → `413`. Parses the activity; malformed → `400`. Known activity id already processed → `202` and ignored. Otherwise `202 Accepted` immediately and processing continues on the Quarkus virtual-thread executor. |
| `GET outbox` | `OrderedCollection` with `totalItems` and `first`; `?page=N` → `OrderedCollectionPage` of public activities, newest first, `collections.page-size` (default 20). |
| `GET followers`, `GET following` | Same paging. `collections.followers-visibility` = `public` (default) / `count-only` / `hidden` (404). |
| `GET objects/{id}`, `GET activities/{id}` | From `ObjectStore`; local-only, public-only unless the request is signed by an actor in the audience **(my call)**; `Tombstone` → `410`. |

Optional `quarkus.activitypub.authorized-fetch=true` (default `false`) requires a valid signature on every GET above.
All routes answer with `Content-Type: application/activity+json; charset=utf-8`, `Vary: Accept`, and ETag/`Cache-Control`
for actor documents (`cache-control.actor`, default `max-age=300`).

### Tier 1 plug-ins

- `ActivityPubWebFingerResolver implements ReactiveWebFingerResolver`: answers `acct:{username}@{base-url host}` and the
  actor URI itself with `subject`, alias (actor id), `self` link (`application/activity+json`) and
  `http://webfinger.net/rel/profile-page` when `LocalActor.url` is set. Unknown username → empty.
- `ActorKeyResolver implements ReactiveKeyResolver`: a key id whose host is our own base URL is answered from
  `ActorStore` without a fetch. Any other key id is fetched as an actor document through `ActivityPubClient`
  (the key id with fragment stripped, then the `publicKey.id` compared); `publicKey.owner` must equal the document `id`,
  and both must share the key id's host, else `empty`. Caching is done by `CachingKeyResolver` already; a failed
  verification triggers its one refetch (key rotation).
- `ActorSigningKeyProvider implements SigningKeyProvider`: `RequestSigner.sign(..., clientName)` is called with
  `clientName = "activitypub:" + keyId`; the provider returns that local actor's private key as a `SigningKey`
  (`Algorithm.RSA_SHA256` for Cavage / `rsa-v1_5-sha256` for RFC 9421; keys are RSA 2048 because Mastodon only verifies
  RSA). Any other `clientName` → empty; with the Tier 1 fix above, config-based webhook keys in the same app keep working.
- `ActivityPubNodeInfoContributor`: adds `activitypub` to NodeInfo protocols, and `usage.users.total` from
  `ActorStore.countLocalActors()`.

### Inbound processing pipeline (`InboxProcessor`)

1. `RequestVerifier.verify(ctx)` from `quarkus-http-signatures` (works without Quarkus Security being configured).
   Only verified requests continue. Verification schemes: both, as configured in `quarkus.http-signatures.verify`.
2. Parse body with the AS mapper. The activity's `actor` (uri) must equal the signing key's owner, else `401`
   **(keeps relayed/forwarded activities out in this milestone; LD-signature forwarding is a non-goal)**.
3. `DomainPolicy` SPI (`Decision allow(String host)`; `@DefaultBean` uses `quarkus.activitypub.blocked-domains`): blocked →
   `202` and drop (never tell blocked servers they are blocked **(my call)**).
4. Dedup: `ActivityStore.markSeen(activityId)` (part of `ObjectStore`, below); if already seen → stop.
5. Respond `202`. Continue on a virtual thread inside `@ActivateRequestContext`; `InboundActivityContext`
   (request-scoped) exposes `recipient()` (the local actor or `Optional.empty()` for the shared inbox), `rawJson()`,
   `signer()` (verified key id and actor uri), `activity()`.
6. Built-in side effects by activity type (each a replaceable `@DefaultBean` handler, `InboundActivityHandler<T>`):
   - `Follow` (object is a local actor): ask `FollowPolicy` SPI → `ACCEPT` (default unless the actor has
     `manuallyApprovesFollowers`), `REJECT`, or `PENDING`. Accept: `FollowerStore.add(local, remote, ACCEPTED)` and deliver
     `Accept(Follow)` to the follower's inbox. Reject: deliver `Reject(Follow)`. Pending: store `PENDING`; the app later
     calls `ActivityPub.acceptFollow(...)`/`rejectFollow(...)`.
   - `Undo(Follow)`: `FollowerStore.remove`.
   - `Accept(Follow)` / `Reject(Follow)` whose object is a follow we sent: update `FollowerStore` "following" state.
   - `Create`, `Update`: the embedded object (or the fetched one when only a reference is given and the activity is
     addressed to us) is stored via `ObjectStore.putRemote(...)` after checking `attributedTo` host == `actor` host.
   - `Delete`: object we hold → `ObjectStore.tombstone`; actor deleting itself → `FollowerStore.removeAll(actor)` and
     `ObjectStore.deleteByActor`; actor unknown to us → ignored with no fetch.
   - `Announce`, `Like`, `Undo(Like/Announce)`, `Block`, `Flag`, `Move`, `Add`, `Remove`, others: no built-in side effect;
     event only.
7. Fire the CDI event. The payload is the activity in its concrete type (`Follow`, `Create`, …) with qualifier
   `@Inbound`, so apps write `void onFollow(@Observes @Inbound Follow f)`; supertypes (`Activity`, `ASObject`) match too.
   Sync observers run first; `@ObservesAsync` observers get a copy. Context stays available through
   `InboundActivityContext`. An observer exception is logged and does not affect the `202` already sent.

Inbound metrics: `activitypub.inbox.received{type,result}` (`result`: accepted, duplicate, blocked, rejected, invalid).

### Outbound API

`ActivityPub` (blocking) and `ReactiveActivityPub` (Mutiny), one implementation:

```java
Activity publish(LocalActor actor, Activity activity);   // assigns ids, stores, adds to outbox, queues delivery
Activity create(LocalActor actor, ASObject object);     // convenience: wraps in Create with audience copied
Activity update(LocalActor actor, ASObject object);
Activity delete(LocalActor actor, URI objectId);        // Tombstone + Delete
Activity follow(LocalActor actor, URI remoteActor);
Activity undoFollow(LocalActor actor, URI remoteActor);
void acceptFollow(LocalActor actor, URI follower);       // for FollowPolicy.PENDING
void rejectFollow(LocalActor actor, URI follower);
Activity announce(LocalActor actor, URI objectId); Activity like(LocalActor actor, URI objectId); Activity undo(LocalActor actor, URI activityId);
```

- Ids: `{base-url}/activities/{ulid}` and `{base-url}/objects/{ulid}` when the app set none; app-set ids are kept.
- Audience expansion (`RecipientResolver`): `to`/`cc`/`bto`/`bcc`/`audience` → actor inboxes. The local actor's own
  `followers` collection expands to follower inboxes; remote collections are not expanded (Mastodon behaviour). Remote
  actors are resolved through `ActivityPubClient` to find `inbox`/`endpoints.sharedInbox`; shared inboxes are preferred
  and deduplicated per host. `bto`/`bcc` are stripped before serialisation. The result is one `Delivery` per inbox.
- `publish` stores the activity and its embedded object in `ObjectStore` (so `GET objects/…` works and the outbox lists
  it), then hands deliveries to `DeliveryExecutor`.

### Delivery

- `Delivery` record: id, inbox URI, activity id, serialized body (bytes, computed once), signing key id, attempts,
  next attempt time, last error, state (`PENDING`, `DELIVERED`, `FAILED`, `GIVEN_UP`).
- `DeliveryExecutor` SPI: `void submit(List<Delivery> deliveries)`. Default **(spec default)**: `OutboxDeliveryExecutor`
  stores them in `DeliveryStore` and a Quarkus Scheduler job (`quarkus.activitypub.delivery.poll-interval`, default `5s`,
  runs on virtual threads, `concurrentExecution = SKIP`) claims due deliveries in batches (`batch-size` 50) and sends them
  with bounded parallelism (`parallelism` 8). `quarkus-scheduler` is a direct dependency of the runtime module **(my call)**.
- Sending: POST through `SafeFetchClient.send` with headers from `RequestSigner.sign` (scheme from
  `delivery.signature-scheme`, default `cavage` because the Fediverse majority still verifies only Cavage; `SchemeMemory`
  double knocking flips a host to RFC 9421 when it rejects Cavage **(my call)**), `Content-Type: application/activity+json`,
  `Digest`/`Content-Digest` as the scheme requires, `User-Agent` from `quarkus.activitypub.user-agent`
  (default `quarkus-activitypub/{version} (+{base-url})`).
- Outcome: 2xx → `DELIVERED`. 4xx other than 408/429 → `FAILED` with no retry (except 401/403 which triggers the one
  scheme flip and a retry). 5xx, 408, 429, timeouts, connection errors → retry with backoff
  `1m, 5m, 30m, 2h, 8h, 24h, 48h` then `GIVEN_UP` (`max-attempts` 8).
- Per-host failure tracking (`HostHealth` in `DeliveryStore`): consecutive failures per host; a host with ≥ 5 consecutive
  failures is "paused" and its deliveries wait for the host's own backoff rather than retrying each; a host failing
  continuously for `dead-after` (default `30d`) gets new deliveries marked `GIVEN_UP` immediately until it succeeds once
  (a probe is still sent once per backoff period).
- Delivery events: `DeliveryCompleted` and `DeliveryFailed` (CDI). Metrics: `activitypub.delivery.attempts{result}`,
  `activitypub.delivery.queue` (gauge). SmallRye Health readiness check `activitypub-delivery` reports the backlog and
  turns DOWN when the oldest pending delivery is older than `health.max-backlog-age` (default `1h`).

### Remote fetching: `ActivityPubClient` / `ReactiveActivityPubClient`

- `fetch(URI)`: GET with `Accept: application/activity+json, application/ld+json; profile="https://www.w3.org/ns/activitystreams"`,
  signed with the instance actor when enabled (Mastodon's secure mode requires it), through `SafeFetchClient`
  (`fetch.timeout` 10s, `fetch.max-size` 1M). The response's `id` host must equal the final URL's host, else
  `ActivityPubException(ID_MISMATCH)`. Result cached in Caffeine for `fetch.cache-ttl` (default `10m`; actors `1h`).
- `fetchActor(URI)` and `resolveActor(String acctOrUri)` (WebFinger via `WebFingerClient` for `user@host`, then fetch).
- `ObjectStore.putRemote` is only called by the inbox pipeline, never by the client; fetching is a cache, not a store.

### Storage SPIs (all with a `Reactive…` twin; defaults are in-memory `@DefaultBean`s that log a warning in prod)

```java
interface ActorStore {
  Optional<LocalActor> findByUsername(String username);
  Optional<LocalActor> findById(URI id);
  long countLocalActors();
  Optional<ActorKeyPair> keyPair(URI actorId);          // private key PEM + public key PEM
  void save(LocalActor actor, ActorKeyPair keys);       // create or update; used by apps and for the instance actor
  void delete(URI actorId);
}
record LocalActor(String username, String type, URI id, Optional<String> name, Optional<String> summary,
                  Optional<URI> url, Optional<URI> icon, Optional<URI> image, boolean manuallyApprovesFollowers,
                  boolean discoverable, Instant published, Map<String, JsonNode> extra)

interface ObjectStore {
  void putLocal(ASObject object, boolean isPublic);  void putRemote(ASObject object, URI receivedFrom);
  Optional<ASObject> find(URI id);  void tombstone(URI id);  void deleteByActor(URI actorId);
  boolean markSeen(URI activityId, Duration retention);         // dedup
  Page<Activity> outbox(URI actorId, int page, int size); long outboxCount(URI actorId);
}

interface FollowerStore {
  void add(URI local, URI remote, FollowState state, URI followActivityId);  void remove(URI local, URI remote);
  void removeAll(URI remote);  Page<URI> followers(URI local, int page, int size); long followerCount(URI local);
  List<URI> followerInboxes(URI local);                           // resolved inbox or sharedInbox per follower
  void addFollowing(URI local, URI remote, FollowState state, URI followActivityId); void removeFollowing(URI local, URI remote);
  Page<URI> following(URI local, int page, int size); long followingCount(URI local);
  Optional<FollowState> followingState(URI local, URI remote);
}

interface DeliveryStore {
  void save(List<Delivery> deliveries);  List<Delivery> claimDue(Instant now, int limit); void update(Delivery delivery);
  HostHealth host(String host); void recordHostResult(String host, boolean success, Instant at);
  long pendingCount(); Optional<Instant> oldestPending();
}
```

`LocalActor` and `ActorKeyPair` are records. Key generation helper `ActorKeys.generateRsa2048()` is provided for apps.

**Provisioning local actors (my call).** An app creates an actor with `ActorStore.save(actor, ActorKeys.generateRsa2048())`;
with the Hibernate module that is the only sanctioned write path, and apps never touch the `ap_` entities. An app whose
users already live in its own tables implements `ActorStore` itself instead and still uses the Hibernate module for the
other stores (each store is a separate bean, so partial replacement works). `LocalActors` is a thin convenience bean
over the store: `LocalActor register(String username, String type, Consumer<LocalActor.Builder>)`.

**Transactions (my call).** Store writes join the caller's transaction when one is active
(`QuarkusTransaction.joiningExisting()` semantics) and open their own otherwise. So an app that saves its post and calls
`activityPub.create(...)` in one transaction gets a true transactional outbox: a rollback discards the activity and its
deliveries. The scheduler claims and updates deliveries in its own transactions. The inbox pipeline groups
`markSeen` + the follower write + the queued `Accept` in one transaction. The in-memory defaults are not transactional
and say so in their prod warning.

**What `putRemote` keeps (my call).** A `RemoteObjectPolicy` SPI decides per inbound object: `KEEP` or `IGNORE`. The
`@DefaultBean` keeps an object only when it is addressed to a local actor (`to`/`cc`/`audience`), or its `inReplyTo`
is a local object, or it is a local object's `Update`; everything else (a followed actor's general timeline) is event-only.
Kept objects expire after `quarkus.activitypub.remote-objects.retention` (default `90d`, `0` disables) through a daily
scheduler job. This keeps the "no silent timeline" anti-goal while giving a blog its replies for free.

**App-assigned ids (my call).** The extension serves only ids under `{base-url}/objects/…` and `/activities/…`. The
recommended pattern is to let the extension assign the id and set the object's `url` to the app's HTML page. An app that
insists on its own ids (`https://blog/posts/hello`) mounts `ActivityPubContentNegotiation` (a `Handler<RoutingContext>`
helper that answers ActivityPub `Accept` values from `ObjectStore` and otherwise calls `next()`) in front of its own
route; the docs show both. Mastodon's refetch of extension-assigned ids therefore always works.

### Security and privacy defaults

- HTTPS only through safe-fetch; private ranges refused (dev mode may relax via `quarkus.safe-fetch.*`).
- The inbox never fetches on behalf of an unverified request. The only fetch an inbound activity can trigger is the
  signer's key, bounded by the key cache, and (for `Create`/`Update` by reference) the referenced object, only when
  the activity is addressed to a local actor.
- `blocked-domains` is matched by suffix (`example.com` blocks `sub.example.com`).
- Private keys live only in `ActorStore`; `LocalActor` carries none. Logs show key ids, never key material or
  signature values.

### Configuration summary (`quarkus.activitypub.*`)

Build time: `enabled` (true), `server.enabled` (true), `devui.enabled` (true).
Runtime: `base-url` (required), `paths.*`, `instance-actor.enabled` (true) / `.username` (`instance`),
`authorized-fetch` (false), `blocked-domains` (empty), `collections.page-size` (20), `collections.followers-visibility`
(`public`), `cache-control.actor` (`max-age=300`), `fetch.timeout` (10s) / `.max-size` (1M) / `.cache-ttl` (10m) /
`.actor-cache-ttl` (1h), `delivery.poll-interval` (5s) / `.batch-size` (50) / `.parallelism` (8) / `.max-attempts` (8) /
`.signature-scheme` (`cavage`) / `.dead-after` (30d), `health.max-backlog-age` (1h), `user-agent`, `dedup.retention` (7d).

### Dev UI

Card: base URL, instance actor id, local actor count, delivery queue (pending/failed/given-up), paused hosts.
Pages: "Actors" (list from `ActorStore`, preview of the actor JSON and the WebFinger answer), "Inbox log"
(last 100 inbound activities kept in memory in dev mode: type, actor, result), "Fetch" (enter a URI or `user@host`,
shows the fetched document), "Deliveries" (table with retry-now).

### Tests (deployment module, `QuarkusUnitTest`)

- `FakeFediverseServer` (from `quarkus-activitypub-test`): an embedded Vert.x server with its own RSA actor, inbox that
  verifies signatures with `http-signatures-core`, WebFinger, and recording of everything received. It plays "Mastodon".
- Route tests: actor negotiation (JSON, 302, 406), WebFinger answer, outbox/followers paging and visibility, object 410.
- Inbox tests: unsigned → 401; signed by the fake with a mismatched `actor` → 401; `Follow` → `Accept` delivered to the fake
  and follower stored; `Undo(Follow)`; duplicate activity id ignored; blocked domain → 202 and no event; `Create(Note)`
  from a followed actor → `@Inbound Create` event with `InboundActivityContext`; `Delete` of unknown actor → no fetch
  (fake's request counter unchanged); body too large → 413.
- Outbound tests: `create(note)` to followers → one signed POST per follower inbox, shared inbox dedup, `bcc` stripped;
  the fake verifies Cavage; a fake that rejects Cavage flips to RFC 9421 once and remembers; 500 → retried with backoff
  (clock-controlled scheduler); 410 → `FAILED` without retry; host paused after 5 failures.
- Key resolver tests: key id host mismatch rejected, `owner` mismatch rejected, rotation refetch.
- Model tests as listed under Module 1. Hibernate tests: every store against H2 (`quarkus-jdbc-h2`, test scope) through
  the same test suite, run by a `StoreContractTest` shared between the in-memory and Hibernate modules.
- `integration-tests` sample app runs the Follow + Create flow against the fake under `@QuarkusIntegrationTest`; CI adds
  the native profile (not run locally).
- Mastodon interop: documented manual procedure (`docs/.../interop.adoc`): expose the sample app with a public HTTPS
  host, search the actor from a Mastodon account, follow, publish, observe. Recorded as not run in the final report.

## Module 4: `quarkus-activitypub-hibernate`

- Hibernate ORM entities (`quarkus-hibernate-orm`, no Panache **(my call)**, so the module makes no demand on app style):
  `ApLocalActor` (+ key PEMs, `@Lob`-free `TEXT` columns), `ApObject` (id, primary type, actor, is_local, is_public,
  published, JSON body as `TEXT`, tombstoned_at), `ApSeenActivity` (id, seen_at), `ApFollower`, `ApFollowing`,
  `ApDelivery`, `ApHostHealth`. Table prefix `ap_`.
- Store beans are plain `@ApplicationScoped` beans (no `@DefaultBean`), so they replace the in-memory defaults by
  presence. Transactions via `QuarkusTransaction.requiringNew()` inside the stores; reactive twins delegate to the blocking
  ones on a worker thread (Hibernate Reactive is a non-goal for this milestone **(my call)**).
- Entities registered with `AdditionalJpaModelBuildItem` in the default persistence unit; `quarkus.activitypub.hibernate.persistence-unit`
  selects another.
- Schema: no migrations shipped; the docs give the DDL for PostgreSQL and recommend Flyway in the app. Tests use H2 with
  `quarkus.hibernate-orm.database.generation=drop-and-create`.

## Acceptance criteria

- [ ] `activitystreams-model` round-trips every fixture without loss; unknown types and single/array forms parse.
- [ ] `quarkus-nodeinfo` serves 2.0 and 2.1 validated against the published JSON schemas, with no ActivityPub on the classpath.
- [ ] A `FakeFediverseServer` can WebFinger a local actor, fetch it, follow it, receive the `Accept`, and receive a
  signed `Create(Note)` that verifies, with both the in-memory stores and the Hibernate stores.
- [ ] Inbound `Follow`, `Create`, `Delete`, `Undo` produce the documented side effects and CDI events.
- [ ] Delivery retries, backoff, host pausing and double knocking behave as specified under a controlled clock.
- [ ] Builds green with `mvn -T 8 install` in each repo, in order: safe-fetch (with POST), webfinger, http-signatures
  (with the provider fix), activitypub. Native and Mastodon interop are documented as not run on this host.

## Left to the builder

- Exact Jackson mix-in/annotation strategy, as long as the model has no Quarkus dependency and round-trips.
- Package and class layout inside each module; names above are the public surface and should be kept.
- ULID vs UUIDv7 for generated ids (either; must be sortable).
- Caffeine sizes, log messages, Dev UI component details.
- Whether the in-memory `DeliveryStore` persists across dev-mode reloads (nice, not required).
