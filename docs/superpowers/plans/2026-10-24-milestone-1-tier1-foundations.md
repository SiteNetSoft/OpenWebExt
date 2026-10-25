# Milestone 1: Tier 1 Foundations Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `quarkus-safe-fetch`, `quarkus-webfinger`, `http-signatures-core` and `quarkus-http-signatures` as three independent Maven repos that build and pass their tests on Quarkus 3.40.1.

**Architecture:** Three sibling git repos under `OpenWebExt/`, one per standard, each with a parent POM, a runtime module and a deployment module (plus a plain `core` library for HTTP Signatures). `quarkus-webfinger` consumes `quarkus-safe-fetch` for every outbound call. All server endpoints are Vert.x routes; all SPIs are CDI interfaces with `@DefaultBean` defaults; config is `@ConfigMapping` split into build-time and runtime roots.

**Tech Stack:** Java 21, Maven 3.9, Quarkus 3.40.1 (`io.quarkus:quarkus-bom`), Vert.x Web Client, Jackson, Caffeine (`quarkus-caffeine`), JUnit 5, `quarkus-junit5-internal` (`QuarkusUnitTest`).

**Spec:** `../../../Quarkus Open Web Extensions — Architecture Spec.md` (sections "Conventions for every module", "Quarkus extension practices", "Tier 1 spec: quarkus-webfinger", "Tier 1 spec: quarkus-http-signatures", "Tier 1 outline" row for `quarkus-safe-fetch`).

## Global Constraints

- Java 21; Quarkus `3.40.1`; license Apache 2.0.
- groupId `org.sitenetsoft.quarkus`; packages `org.sitenetsoft.quarkus.<name>`; plain library `http-signatures-core` uses package `org.sitenetsoft.httpsignatures`.
- Artifacts: `quarkus-<name>` and `quarkus-<name>-deployment`; config prefix `quarkus.<name>.*`; logging category `org.sitenetsoft.quarkus.<name>`.
- Version `0.1.0-SNAPSHOT` for all three repos.
- Every extension module runs `io.quarkus:quarkus-extension-processor` as an annotation processor; runtime modules run `quarkus-extension-maven-plugin` goal `extension-descriptor`; each runtime ships `META-INF/quarkus-extension.yaml` (name, description, categories, keywords, status `experimental`) and `quarkus-skill.md`.
- `@ConfigMapping` + `@ConfigRoot(phase=…)`; build-time and runtime roots in separate interfaces; defaults only via `@WithDefault`.
- Blocking client is `FooClient`; Mutiny client is `ReactiveFooClient`; both CDI beans over one implementation.
- Jackson only; Vert.x routes for server endpoints; `/.well-known/*` at server root (absolute routes), build warning when `quarkus.http.root-path` is not `/`.
- Secure defaults: HTTPS only, timeouts and size limits on outbound calls; loopback, link-local and private ranges refused by default; secrets, keys, signature values never logged.
- Host constraint (CLAUDE.md): Maven `-T 8` at most; Surefire `argLine` includes `-XX:TieredStopAtLevel=1`; one heavy build at a time. `JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64`.
- No `native-image` on this host: native CI jobs are written but not run locally; say so in the final report.

## Review Focus

1. **`resource` with a bare `user@host` (no `acct:` scheme) or an `https:` resource**: the server must treat the string as a URI, pass it to resolvers, and return 400 only for unparseable values. Test added to Task 9.
2. **`rel` filter matching no links**: the response must still be 200 with an empty `links` array, not 404. Test added to Task 9.
3. **Redirect to a private address after a public first hop**: safe-fetch must re-check every hop and refuse. Test added to Task 4.
4. **Signature header present but malformed (bad structured field, unknown key id, unknown algorithm)**: verification must fail closed with a typed reason and the auth mechanism must answer 401, never 500. Tests added to Task 14 and Task 19.
5. **Cavage request whose `Date` header is unparseable or missing while `(created)` is absent**: must fail as stale/invalid, not pass. Test added to Task 15.

---

## Repo layout

```
OpenWebExt/
  quarkus-safe-fetch/          git repo, parent pom artifactId quarkus-safe-fetch-parent
    runtime/                   artifactId quarkus-safe-fetch
    deployment/                artifactId quarkus-safe-fetch-deployment
  quarkus-webfinger/           git repo, quarkus-webfinger-parent
    runtime/  deployment/
  quarkus-http-signatures/     git repo, quarkus-http-signatures-parent
    core/                      artifactId http-signatures-core (no Quarkus dependency)
    runtime/  deployment/
```

Build order: safe-fetch → `mvn install` → webfinger; http-signatures is independent.

Common parent POM content (every repo): `<maven.compiler.release>21`, `quarkus.version=3.40.1`, `dependencyManagement` importing `io.quarkus:quarkus-bom:${quarkus.version}` (pom, import), `maven-surefire-plugin` 3.5.x with `<argLine>-XX:TieredStopAtLevel=1</argLine>` and system properties `java.util.logging.manager=org.jboss.logmanager.LogManager`, `maven-compiler-plugin` with annotation processor path `io.quarkus:quarkus-extension-processor:${quarkus.version}` (extension modules only), `maven-release`-free, Apache 2.0 `<licenses>`.

---

## Part A: quarkus-safe-fetch

### Task 1: Repo skeleton and extension metadata

**Files:**
- Create: `quarkus-safe-fetch/pom.xml`, `runtime/pom.xml`, `deployment/pom.xml`
- Create: `runtime/src/main/resources/META-INF/quarkus-extension.yaml`
- Create: `deployment/src/main/java/org/sitenetsoft/quarkus/safefetch/deployment/SafeFetchProcessor.java`
- Create: `LICENSE` (Apache 2.0), `README.md`, `.gitignore`, `quarkus-skill.md`

**Interfaces:**
- Produces: feature name `safe-fetch`; `SafeFetchProcessor.feature()` returns `new FeatureBuildItem("safe-fetch")`.

- [ ] **Step 1: `git init` and write the three POMs.** Runtime depends on `quarkus-core`, `quarkus-vertx`, `quarkus-arc`, `quarkus-jackson`; deployment depends on the `-deployment` counterparts and `quarkus-junit5-internal` (test) plus `io.rest-assured:rest-assured` (test).
- [ ] **Step 2: Write `quarkus-extension.yaml`**: name `Safe Fetch`, description `Outbound HTTP with SSRF guard, timeouts, size and redirect limits`, metadata keywords `[http, ssrf, fetch, security]`, categories `[web]`, status `experimental`, guide `https://github.com/SiteNetSoft/quarkus-safe-fetch`.
- [ ] **Step 3: Write `SafeFetchProcessor` with the feature build step.**
- [ ] **Step 4: Verify** `JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 mvn -q -T 8 install -DskipTests` → BUILD SUCCESS.
- [ ] **Step 5: Commit** `chore: scaffold quarkus-safe-fetch extension`.

### Task 2: Config and address policy

**Files:**
- Create: `runtime/src/main/java/org/sitenetsoft/quarkus/safefetch/SafeFetchConfig.java`
- Create: `runtime/.../AddressPolicy.java`, `runtime/.../DefaultAddressPolicy.java`, `runtime/.../BlockedAddressException.java`, `runtime/.../SafeFetchException.java`
- Test: `runtime/src/test/java/org/sitenetsoft/quarkus/safefetch/DefaultAddressPolicyTest.java`

**Interfaces:**
- Produces:
  - `@ConfigMapping(prefix="quarkus.safe-fetch") @ConfigRoot(phase=RUN_TIME) interface SafeFetchConfig { @WithDefault("true") boolean httpsOnly(); @WithDefault("false") boolean allowPrivateAddresses(); @WithDefault("10s") Duration timeout(); @WithDefault("1M") MemorySize maxResponseSize(); @WithDefault("3") int maxRedirects(); @WithDefault("quarkus-safe-fetch") String userAgent(); }`
  - `public interface AddressPolicy { boolean allows(InetAddress address); }`
  - `@DefaultBean @ApplicationScoped class DefaultAddressPolicy implements AddressPolicy` refusing loopback, any-local, link-local, site-local, multicast, and IPv6 unique-local (`fc00::/7`), IPv4 shared (`100.64.0.0/10`), `0.0.0.0/8`, `169.254.169.254`; allows when `config.allowPrivateAddresses()`.
  - `class SafeFetchException extends RuntimeException`; `class BlockedAddressException extends SafeFetchException`.

- [ ] **Step 1: Write failing tests** `refusesLoopback` (`127.0.0.1`, `::1`), `refusesPrivate` (`10.0.0.1`, `192.168.1.1`, `172.16.0.1`, `fd00::1`, `169.254.169.254`, `100.64.0.1`), `allowsPublic` (`93.184.216.34`, `2606:2800:220:1:248:1893:25c8:1946`), `allowsPrivateWhenConfigured`. Construct `DefaultAddressPolicy(boolean allowPrivate)` directly.
- [ ] **Step 2: Run** `mvn -q test -pl runtime` → FAIL (class missing).
- [ ] **Step 3: Implement** the interfaces above; the default policy has a constructor `DefaultAddressPolicy(SafeFetchConfig)` for CDI and a package-private `DefaultAddressPolicy(boolean)` for tests.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(safe-fetch): config mapping and default address policy`.

### Task 3: Fetch client (blocking and reactive)

**Files:**
- Create: `runtime/.../FetchOptions.java`, `FetchResponse.java`, `SafeFetchClient.java`, `ReactiveSafeFetchClient.java`, `SafeFetchClientImpl.java`, `ResponseTooLargeException.java`, `TooManyRedirectsException.java`, `FetchTimeoutException.java`, `UnreachableException.java`
- Test: `deployment/src/test/java/org/sitenetsoft/quarkus/safefetch/deployment/SafeFetchClientTest.java`

**Interfaces:**
- Produces:
  - `record FetchOptions(Optional<Duration> timeout, Optional<Long> maxResponseSize, Map<String,String> headers) { static FetchOptions defaults(); FetchOptions withTimeout(Duration); FetchOptions withMaxResponseSize(long); FetchOptions withHeader(String,String); }` — empty optionals mean "use config".
  - `record FetchResponse(int status, Map<String,List<String>> headers, byte[] body) { Optional<String> header(String name); String bodyAsString(); }` header lookup case-insensitive.
  - `public interface SafeFetchClient { FetchResponse get(URI uri, FetchOptions options); default FetchResponse get(URI uri) }`
  - `public interface ReactiveSafeFetchClient { Uni<FetchResponse> get(URI uri, FetchOptions options); }`
  - `@ApplicationScoped class SafeFetchClientImpl implements SafeFetchClient, ReactiveSafeFetchClient` injecting `Vertx` (io.vertx.mutiny.core.Vertx), `SafeFetchConfig`, `AddressPolicy`. Non-2xx statuses are returned, not thrown; only transport failures throw.
- Algorithm (not determined by signature): for each hop (≤ `maxRedirects`): reject non-`https` scheme unless `httpsOnly=false`; resolve host with `InetAddress.getAllByName` on a worker (`vertx.executeBlocking`); every address must pass `AddressPolicy`; connect to the first address with `WebClient` request `.host(ip).port(p).ssl(scheme==https).virtualHost(host)`, `followRedirects(false)`, `timeout(ms)`, `putHeader("User-Agent", ua)`, `putHeader("Accept-Encoding", "identity")`; on 301/302/303/307/308 resolve `Location` against the current URI and loop; a `Content-Length` greater than the limit or a buffered body exceeding it throws `ResponseTooLargeException` (use `BodyCodec.buffer()` then check length; also pre-check `Content-Length` via `expect` handler so large bodies are not downloaded). The blocking `get` is `reactive.get(...).await().atMost(timeout + 1s)`.

- [ ] **Step 1: Write failing `QuarkusUnitTest`** with an embedded Vert.x `HttpServer` on `127.0.0.1` random port started in `@BeforeAll` serving: `/ok` → 200 `{"a":1}`; `/big` → 100 KB body; `/redirect` → 302 to `/ok`; `/loop` → 302 to `/loop`; `/slow` → responds after 3s; `/to-private` → 302 to `http://10.0.0.1/`. Config: `quarkus.safe-fetch.https-only=false`, `allow-private-addresses=true`, `max-response-size=10K`, `max-redirects=2`, `timeout=1s`. Tests: `getsBody`, `followsRedirectOnce`, `rejectsRedirectLoop` (TooManyRedirectsException), `rejectsOversizedBody` (ResponseTooLargeException), `timesOut` (FetchTimeoutException), `returnsNon2xxWithoutThrowing` (`/missing` → 404), `reactiveFormWorks`.
- [ ] **Step 2: Run** `mvn -q test -pl deployment -Dtest=SafeFetchClientTest` → FAIL.
- [ ] **Step 3: Implement** the types above.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(safe-fetch): blocking and reactive fetch client`.

### Task 4: SSRF guard in prod mode and redirect re-checks

**Files:**
- Test: `deployment/src/test/java/.../SafeFetchBlocksPrivateTest.java`

- [ ] **Step 1: Write failing `QuarkusUnitTest`** with default config except `https-only=false`: `refusesLoopback` (`http://127.0.0.1:<port>/ok` → `BlockedAddressException`), `refusesHostResolvingToPrivate` (`http://localhost/` → BlockedAddressException), `refusesHttpWhenHttpsOnly` (second test class or `.overrideConfigKey`; `http://example.com/` → `SafeFetchException` with message containing `https`). Review Focus 3: with `allow-private-addresses=false` a server reachable only through a test hook is not available, so test the policy path by asserting `/to-private` style redirect against a public-looking hostname fails with `BlockedAddressException` using a `AddressPolicy` test bean that allows only the test server's loopback address for the first hop and refuses `10.0.0.1`.
- [ ] **Step 2: Run** → FAIL or PASS; if PASS already, keep as regression and proceed.
- [ ] **Step 3: Fix** anything failing.
- [ ] **Step 4: Run** `mvn -q test` (whole repo) → PASS.
- [ ] **Step 5: Commit** `test(safe-fetch): SSRF guard and redirect re-check coverage`.

### Task 5: Metrics (conditional), Dev UI card, CI, docs

**Files:**
- Create: `runtime/.../metrics/SafeFetchMetrics.java` (optional dependency `quarkus-micrometer`, bean added by deployment only when `capabilities.isPresent(Capability.MICROMETER)` — check the constant name in `io.quarkus.deployment.Capability`; fall back to the string `"io.quarkus.micrometer"`)
- Create: `deployment/.../devui/SafeFetchDevUiProcessor.java` (`CardPageBuildItem` with an external link page to the README and a `Page.webComponentPageBuilder` is NOT required; a single `Page.externalPageBuilder("Configuration")` pointing at the config guide is enough for 0.1)
- Create: `.github/workflows/build.yml` (matrix `quarkus: [3.40.1, latest]`, Java 21; separate `native` job using `-Dnative -Dquarkus.native.container-build=true -Dquarkus.native.additional-build-args=-J-XX:TieredStopAtLevel=1,--parallelism=8`), `docs/modules/ROOT/pages/index.adoc`, `docs/antora.yml`
- Modify: `README.md` (usage, config table, minimum Quarkus `3.40`)

**Interfaces:**
- Produces: `SafeFetchMetrics.record(String hostCategory, int status, Duration elapsed)` — counter `safe_fetch_requests_total{outcome}` and timer `safe_fetch_request_duration`.

- [ ] **Step 1: Implement** metrics hook called from `SafeFetchClientImpl` through an `Instance<SafeFetchMetrics>` (unsatisfied when Micrometer is absent).
- [ ] **Step 2: Add Dev UI card, CI workflow and Antora skeleton.**
- [ ] **Step 3: Verify** `mvn -q -T 8 install` → BUILD SUCCESS, tests pass.
- [ ] **Step 4: Commit** `feat(safe-fetch): metrics hook, dev ui card, ci and docs`.

---

## Part B: quarkus-webfinger

### Task 6: Repo skeleton

**Files:** same shape as Task 1 under `quarkus-webfinger/`; runtime depends on `org.sitenetsoft.quarkus:quarkus-safe-fetch:0.1.0-SNAPSHOT`, `quarkus-vertx-http`, `quarkus-jackson`, `quarkus-caffeine`; deployment on their `-deployment` artifacts.

- [ ] **Step 1: Write POMs, `quarkus-extension.yaml`** (name `WebFinger`, description `Serve and query WebFinger (RFC 7033) and host-meta (RFC 6415)`, keywords `[webfinger, host-meta, discovery, activitypub, oidc]`), `WebFingerProcessor.feature()` → `"webfinger"`, LICENSE, README, `.gitignore`, `quarkus-skill.md`.
- [ ] **Step 2: Verify** `mvn -q -T 8 install -DskipTests` → BUILD SUCCESS.
- [ ] **Step 3: Commit** `chore: scaffold quarkus-webfinger extension`.

### Task 7: Model and JSON codec

**Files:**
- Create: `runtime/src/main/java/org/sitenetsoft/quarkus/webfinger/Link.java`, `JsonResourceDescriptor.java`, `AcctUri.java`, `WebFingerJacksonModule.java`, `WebFingerObjectMapperCustomizer.java`
- Test: `runtime/src/test/java/.../AcctUriTest.java`, `JrdJsonTest.java`

**Interfaces:**
- Produces (exactly the spec's records, with Jackson annotations on constructor parameters):
  - `record Link(String rel, String type, URI href, String template, Map<String,String> titles, Map<URI,String> properties)` with `static Link of(String rel, URI href)`, `static Link template(String rel, String template)`.
  - `record JsonResourceDescriptor(URI subject, List<URI> aliases, Map<URI,String> properties, List<Link> links)` with `static JsonResourceDescriptor of(URI subject)`, `JsonResourceDescriptor withLink(Link)`, `JsonResourceDescriptor filterRels(Set<String> rels)` (empty set → unchanged), `JsonResourceDescriptor merge(JsonResourceDescriptor lowerPriority)` (keeps own subject, concatenates aliases distinct, concatenates links, properties: own wins).
  - `record AcctUri(String user, String host) { static AcctUri parse(String value); URI toUri(); }` — accepts `acct:user@host` and bare `user@host`; percent-decodes; converts host with `java.net.IDN.toASCII`; throws `IllegalArgumentException` on missing `@`, empty user or host.
  - `WebFingerJacksonModule extends SimpleModule`: serializes JRD/Link with JSON keys `subject`, `aliases`, `properties`, `links`, `rel`, `type`, `href`, `template`, `titles`, `properties`; omits null and empty collections; deserializes tolerant of missing fields and unknown fields.
  - `@Singleton class WebFingerObjectMapperCustomizer implements ObjectMapperCustomizer` registering the module.

- [ ] **Step 1: Write failing tests.** `AcctUriTest`: `parsesAcctScheme` (`acct:alice@example.com` → user `alice`, host `example.com`, `toUri()` = `acct:alice@example.com`), `parsesBare`, `decodesPercent` (`acct:al%40ice@example.com` → `al@ice`), `idnHostToAscii` (`acct:bob@bücher.example` → host `xn--bcher-kva.example`), `rejectsMissingAt`. `JrdJsonTest`: `rfc7033Section3Example1RoundTrips` using the RFC 7033 §3.1 example JRD (`subject` `acct:carol@example.com`, link rel `http://openid.net/specs/connect/1.0/issuer` href `https://openid.example.com`), `rfc7033Section4Example` (the `http://blog.example.com/article/id/314` example with `aliases`, `properties`, `links` with `titles` and `properties`) round-trips to equal records, `omitsEmptyFields` (serialize `JsonResourceDescriptor.of(acct)` → JSON only has `subject`), `filterRelsKeepsMatching`, `mergeConcatenatesAndPrefersHigherPriorityProperties`.
- [ ] **Step 2: Run** `mvn -q test -pl runtime` → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(webfinger): JRD model, acct URIs and Jackson module`.

### Task 8: Server SPI, config roots and build-time resolver discovery

**Files:**
- Create: `runtime/.../WebFingerRequest.java`, `WebFingerResolver.java`, `ReactiveWebFingerResolver.java`, `WebFingerBuildTimeConfig.java`, `WebFingerRuntimeConfig.java`, `WebFingerRecorder.java`, `server/WebFingerHandler.java`, `server/HostMetaHandler.java`
- Modify: `deployment/.../WebFingerProcessor.java`
- Test: `deployment/src/test/java/.../NoResolverDisablesEndpointTest.java`

**Interfaces:**
- Produces:
  - `record WebFingerRequest(URI resource, Set<String> rels, String host)`
  - `public interface WebFingerResolver { Optional<JsonResourceDescriptor> resolve(WebFingerRequest request); }`
  - `public interface ReactiveWebFingerResolver { Uni<Optional<JsonResourceDescriptor>> resolve(WebFingerRequest request); }`
  - `@ConfigMapping(prefix="quarkus.webfinger") @ConfigRoot(phase=BUILD_TIME) interface WebFingerBuildTimeConfig { Server server(); Oidc oidc(); interface Server { @WithDefault("true") boolean enabled(); HostMeta hostMeta(); interface HostMeta { @WithDefault("false") boolean enabled(); } } interface Oidc { @WithDefault("false") boolean enabled(); } }`
  - `@ConfigMapping(prefix="quarkus.webfinger") @ConfigRoot(phase=RUN_TIME) interface WebFingerRuntimeConfig { Server server(); Oidc oidc(); Client client(); interface Server { @WithDefault("true") boolean cors(); } interface Oidc { Optional<URI> issuer(); Optional<List<String>> domains(); } interface Client { @WithDefault("10s") Duration timeout(); @WithDefault("64K") MemorySize maxResponseSize(); @WithDefault("true") boolean hostMetaFallback(); Cache cache(); interface Cache { @WithDefault("1000") int maxEntries(); @WithDefault("1h") Duration maxTtl(); } } }`
  - `@Recorder class WebFingerRecorder { Handler<RoutingContext> webfingerHandler(); Handler<RoutingContext> hostMetaXrdHandler(); Handler<RoutingContext> hostMetaJsonHandler(); }` — handlers resolve beans lazily through `Arc.container()`.
  - `WebFingerHandler` (ApplicationScoped bean, `implements Handler<RoutingContext>`): injects `@All List<WebFingerResolver>`, `@All List<ReactiveWebFingerResolver>`, `ObjectMapper`, `WebFingerRuntimeConfig`, `Vertx`. Behavior: missing/blank `resource` or `URI.create` failure → 400 `application/json` `{"error":"invalid_resource"}`; run every resolver (blocking ones via `vertx.executeBlocking`, ordered highest `@Priority` first — ArC `@All` already sorts), merge non-empty results in order with `merge`, apply `filterRels(rels)`; none → 404; else 200 `application/jrd+json`, header `Access-Control-Allow-Origin: *` when `server.cors()`.
  - `HostMetaHandler`: XRD `application/xrd+xml` body `<?xml version="1.0" encoding="UTF-8"?><XRD xmlns="http://docs.oasis-open.org/ns/xri/xrd-1.0"><Link rel="lrdd" template="https://{host}/.well-known/webfinger?resource={uri}" type="application/jrd+json"/></XRD>` with `{host}` replaced by the request `Host`; JSON variant via the JRD codec, with the same single link.
- Deployment: build step `registerRoutes(WebFingerBuildTimeConfig, CombinedIndexBuildItem, HttpRootPathBuildItem, HttpBuildTimeConfig, WebFingerRecorder, BuildProducer<RouteBuildItem>)` — collects implementors of both resolver interfaces from the Jandex index (`index.getAllKnownImplementors`); if `server.enabled` and (implementors non-empty or `oidc.enabled`), produces absolute routes `RouteBuildItem.builder().route("/.well-known/webfinger").handler(recorder.webfingerHandler()).build()` plus host-meta routes when enabled; else logs at INFO `WebFinger endpoint not registered: no WebFingerResolver or ReactiveWebFingerResolver bean found`. Warn when `quarkus.http.root-path` is not `/`. Also produces `UnremovableBeanBuildItem.beanTypes(WebFingerResolver.class, ReactiveWebFingerResolver.class)` and `AdditionalBeanBuildItem` for `WebFingerHandler`, `HostMetaHandler`, `WebFingerObjectMapperCustomizer`.

- [ ] **Step 1: Write failing `QuarkusUnitTest`** `NoResolverDisablesEndpointTest`: no resolver bean, `setLogRecordPredicate` capturing the INFO message; `GET /.well-known/webfinger?resource=acct:a@b` → 404 and log record contains `not registered`.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** everything above.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(webfinger): server SPI, config roots and conditional route registration`.

### Task 9: Server endpoint behavior

**Files:**
- Test: `deployment/src/test/java/.../WebFingerServerTest.java`, `HostMetaTest.java`, test resolvers `AliceResolver` (`@Priority(10)`, answers `acct:alice@example.com` with link `self` href `https://example.com/users/alice` and property `http://example.com/p` = `low`), `OidcLikeResolver` (`@Priority(100)`, same subject, link `http://openid.net/specs/connect/1.0/issuer`, property `http://example.com/p` = `high`, alias `https://example.com/@alice`), `ReactiveBobResolver` (`ReactiveWebFingerResolver`, answers `acct:bob@example.com`).

- [ ] **Step 1: Write failing tests** with RestAssured: `mergesTwoResolvers` (two links, property value `high`, alias present, content type `application/jrd+json`, header `Access-Control-Allow-Origin: *`), `reactiveResolverAnswers`, `unknownSubjectIs404`, `missingResourceIs400`, `malformedResourceIs400` (`resource=::not a uri`), `relFilterKeepsOnlyMatching` (`rel=self` → one link), `relFilterNoMatchIs200WithEmptyLinks` (Review Focus 2), `httpsResourceIsPassedThrough` (resolver answering `https://example.com/article/1`; Review Focus 1), `corsCanBeDisabled` (second test class with `quarkus.webfinger.server.cors=false` → header absent). `HostMetaTest` (`quarkus.webfinger.server.host-meta.enabled=true`): `xrdHasLrddTemplate` (content type starts with `application/xrd+xml`, body contains `template="https://localhost:8081/.well-known/webfinger?resource={uri}"` using the request Host), `jsonHasLrddLink`.
- [ ] **Step 2: Run** → FAIL on at least some.
- [ ] **Step 3: Fix implementation** until green.
- [ ] **Step 4: Run** `mvn -q test` → PASS.
- [ ] **Step 5: Commit** `feat(webfinger): merged, rel-filtered JRD responses and host-meta`.

### Task 10: Built-in OIDC issuer resolver

**Files:**
- Create: `runtime/.../oidc/OidcIssuerResolver.java`
- Modify: `WebFingerProcessor` (add `AdditionalBeanBuildItem` for the resolver when `oidc.enabled`)
- Test: `deployment/src/test/java/.../OidcIssuerResolverTest.java`

**Interfaces:**
- Produces: `@ApplicationScoped @Priority(0) class OidcIssuerResolver implements WebFingerResolver` — answers when the resource parses as `AcctUri` whose host is in `oidc.domains` (case-insensitive) with `JsonResourceDescriptor.of(resource).withLink(Link.of("http://openid.net/specs/connect/1.0/issuer", issuer))`. Missing `issuer` or `domains` at runtime → `IllegalStateException` on first request with a clear message naming the property.

- [ ] **Step 1: Write failing test** (`quarkus.webfinger.oidc.enabled=true`, `issuer=https://openid.example.com`, `domains=example.com,example.org`): `answersConfiguredDomain` (exact RFC 7033 §3.1 shape), `ignoresOtherDomain` (404), `ignoresNonAcct` (404).
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(webfinger): built-in OIDC issuer resolver`.

### Task 11: Client with LRDD fallback and caching

**Files:**
- Create: `runtime/.../client/WebFingerClient.java`, `ReactiveWebFingerClient.java`, `WebFingerClientImpl.java`, `WebFingerException.java`, `WebFingerNotFoundException.java`, `WebFingerInvalidResponseException.java`, `WebFingerUnreachableException.java`, `HostMetaTemplates.java`
- Test: `deployment/src/test/java/.../WebFingerClientTest.java`

**Interfaces:**
- Produces:
  - `public interface WebFingerClient { JsonResourceDescriptor lookup(URI resource, Set<String> rels); JsonResourceDescriptor lookup(AcctUri account, Set<String> rels); }`
  - `public interface ReactiveWebFingerClient { Uni<JsonResourceDescriptor> lookup(URI resource, Set<String> rels); Uni<JsonResourceDescriptor> lookup(AcctUri account, Set<String> rels); }`
  - `@ApplicationScoped class WebFingerClientImpl implements WebFingerClient, ReactiveWebFingerClient` injecting `ReactiveSafeFetchClient`, `ObjectMapper`, `WebFingerRuntimeConfig`. Host for a URI: `AcctUri` host for `acct:`, else `uri.getHost()`; `null` host → `WebFingerInvalidResponseException`? No: `IllegalArgumentException("resource has no host")`.
  - `HostMetaTemplates`: Caffeine cache `host → Optional<String template>` (max `cache.maxEntries`, expiry `cache.maxTtl`).
  - Result cache: Caffeine `resource+rels → JRD` with per-entry expiry = min(`Cache-Control: max-age`, `cache.maxTtl`); no `max-age` → not cached.
  - Resolution: GET `https://<host>/.well-known/webfinger?resource=<enc>&rel=<enc>…` with `Accept: application/jrd+json, application/json`; 200 → parse (parse error or `subject` missing → `WebFingerInvalidResponseException`); 404 and `hostMetaFallback` → fetch `https://<host>/.well-known/host-meta.json`, then `host-meta` (XRD, extract `template` attribute of the `rel="lrdd"` Link with a regex), expand `{uri}` once, GET; still 404 or no template → `WebFingerNotFoundException`; other 4xx/5xx → `WebFingerInvalidResponseException` with status; any `SafeFetchException` → `WebFingerUnreachableException` (cause attached).

- [ ] **Step 1: Write failing `QuarkusUnitTest`** with an embedded HTTP server on loopback playing two hosts selected by `Host` header: `direct.test` (serves `/.well-known/webfinger` with `Cache-Control: max-age=60`), `lrdd.test` (webfinger 404; `host-meta` XRD with template pointing to `/finger?r={uri}`; `/finger` serves the JRD). Config: `quarkus.safe-fetch.https-only=false`, `allow-private-addresses=true`; the test registers an `AddressPolicy`-independent way to reach the fake hosts: a test `@Alternative @Priority(1) HostResolver` is NOT in the API, so instead run the embedded server on port 80? Not possible unprivileged. **Decision:** `WebFingerClientImpl` reads an internal, undocumented runtime property `quarkus.webfinger.client.base-url-override` (`Optional<String>`, e.g. `http://127.0.0.1:PORT`) that replaces scheme and authority for every request while preserving the `Host` header via `FetchOptions.withHeader("Host", host)`; tests set it via `overrideConfigKey` after the server starts (use a fixed free port picked in a static initializer). Tests: `looksUpDirect`, `cachesWithinMaxAge` (second call does not hit the server: assert request counter), `fallsBackToLrdd`, `remembersTemplatePerHost` (second lookup on `lrdd.test` skips host-meta: counter), `notFoundIsTyped`, `invalidJsonIsTyped`, `unreachableIsTyped` (host `down.test` → server returns connection reset / or base override to a closed port), `acctOverloadDelegates`, `reactiveForm`.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** `mvn -q test` → PASS.
- [ ] **Step 5: Commit** `feat(webfinger): client with LRDD fallback and caching`.

### Task 12: Dev UI, CI, docs

**Files:**
- Create: `deployment/.../devui/WebFingerDevUiProcessor.java` (`CardPageBuildItem` with `Page.webComponentPageBuilder().title("Lookup").componentLink("qwc-webfinger-lookup.js")`), `deployment/src/main/resources/dev-ui/qwc-webfinger-lookup.js` (Lit component: text box + button calling JSON-RPC `lookup`), `runtime/.../devui/WebFingerJsonRpcService.java` (`public JsonNode lookup(String resource)` via `WebFingerClient`; `public List<String> resolvers()` via bean names), `JsonRPCProvidersBuildItem`.
- Create: `.github/workflows/build.yml`, `docs/antora.yml`, `docs/modules/ROOT/pages/index.adoc`; update `README.md` with the config table from the spec.

- [ ] **Step 1: Implement.** **Step 2: Verify** `mvn -q -T 8 install` → BUILD SUCCESS and tests pass.
- [ ] **Step 3: Commit** `feat(webfinger): dev ui lookup page, ci and docs`.

---

## Part C: quarkus-http-signatures

### Task 13: Repo skeleton with `core` module

**Files:** `quarkus-http-signatures/pom.xml`, `core/pom.xml` (artifactId `http-signatures-core`, deps: `com.fasterxml.jackson.core:jackson-databind` only, version from the Quarkus BOM import; test: JUnit 5), `runtime/pom.xml` (depends on core, `quarkus-core`, `quarkus-arc`, `quarkus-vertx-http`, `quarkus-caffeine`; `optional` deps `quarkus-security`, `quarkus-rest`, `quarkus-rest-client`), `deployment/pom.xml`, `quarkus-extension.yaml` (name `HTTP Signatures`, description `Sign and verify HTTP messages with RFC 9421 and Cavage signatures and body digests`, keywords `[http-signatures, rfc9421, cavage, webhook, activitypub, security]`), `HttpSignaturesProcessor.feature()` → `"http-signatures"`, LICENSE, README, `.gitignore`, `quarkus-skill.md`.

- [ ] **Step 1: Write files. Step 2: Verify** `mvn -q -T 8 install -DskipTests` → SUCCESS. **Step 3: Commit** `chore: scaffold quarkus-http-signatures with core library`.

### Task 14: Core primitives: structured fields, digests, keys, algorithms

**Files:**
- Create under `core/src/main/java/org/sitenetsoft/httpsignatures/`: `Scheme.java`, `Algorithm.java`, `KeyType.java`, `Component.java`, `Headers.java`, `StructuredFields.java`, `Digests.java`, `DigestAlgorithm.java`, `Keys.java`, `SignatureException.java` (checked? No: unchecked `HttpSignatureException extends RuntimeException` with `enum Reason`), `BodyDigest.java`
- Test: `core/src/test/java/.../StructuredFieldsTest.java`, `DigestsTest.java`, `KeysTest.java`, `AlgorithmTest.java`

**Interfaces:**
- `enum Scheme { RFC9421, CAVAGE }`
- `enum Algorithm { RSA_V1_5_SHA256("rsa-v1_5-sha256", KeyType.RSA), RSA_PSS_SHA512("rsa-pss-sha512", RSA), ECDSA_P256_SHA256("ecdsa-p256-sha256", EC_P256), ECDSA_P384_SHA384("ecdsa-p384-sha384", EC_P384), ED25519("ed25519", ED25519), HMAC_SHA256("hmac-sha256", HMAC), RSA_SHA256("rsa-sha256", RSA), HS2019("hs2019", null); String id(); KeyType keyType(); static Optional<Algorithm> fromId(String); boolean isCavage(); boolean isRfc9421(); }` — `ED25519` belongs to both schemes; `RSA_SHA256`, `HS2019` Cavage only; the rest RFC 9421 only.
- `enum KeyType { RSA, EC_P256, EC_P384, ED25519, HMAC; static KeyType of(java.security.Key key); }` (EC curve detected from `ECKey.getParams().getCurve().getField().getFieldSize()` 256/384).
- `record Component(String name, Map<String,String> params) { static Component of(String name); boolean isDerived(); String serialized(); }` — name lowercased; derived names start with `@`.
- `interface Headers { List<String> values(String name); default Optional<String> first(String); default String combined(String name) (values joined by ", ", each trimmed and inner whitespace collapsed per RFC 9421 §2.1); static Headers of(Map<String,List<String>>); static Headers empty(); }`
- `StructuredFields`: `record Item(Object value, Map<String,Object> params)` where value is `String`, `Long`, `Boolean`, `byte[]` (byte sequence) or `Token`(record); `record InnerList(List<Item> items, Map<String,Object> params)`; `Map<String,Object> parseDictionary(String)` (values `Item` or `InnerList`); `String serializeInnerList(InnerList)`; `String serializeDictionary(Map<String,Object>)`; byte sequences serialize as `:base64:`; strings as quoted with `\"` and `\\` escapes; integers as decimal; booleans as `?1`/`?0`.
- `enum DigestAlgorithm { SHA_256("sha-256","SHA-256"), SHA_512("sha-512","SHA-512"); }`
- `Digests`: `String contentDigest(byte[] body, DigestAlgorithm alg)` → e.g. `sha-256=:X48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE=:`; `boolean verifyContentDigest(String headerValue, byte[] body)` (dictionary may hold several algorithms; all known ones must match; unknown-only → false); `String legacyDigest(byte[] body)` → `SHA-256=<base64>`; `boolean verifyLegacyDigest(String headerValue, byte[] body)` (comma-separated list, `SHA-256` or `sha-256` token).
- `record BodyDigest(byte[] body) { static BodyDigest of(byte[]); boolean isEmpty(); }`
- `Keys`: `java.security.Key parsePem(String pem)` handling `PRIVATE KEY` (PKCS#8), `PUBLIC KEY` (X.509 SPKI), `RSA PRIVATE KEY` (PKCS#1, wrapped into PKCS#8 with the RSA OID), `RSA PUBLIC KEY` (PKCS#1 → wrap into SPKI); `java.security.Key parseJwk(String json)` for `kty` `RSA` (n,e[,d,p,q,dp,dq,qi → use `RSAPrivateCrtKeySpec` when present else `RSAPrivateKeySpec`), `EC` (crv P-256/P-384, x,y[,d]), `OKP` (crv Ed25519, x[,d]), `oct` (k → `SecretKeySpec` HmacSHA256); `SecretKey hmacKey(byte[])`.
- `HttpSignatureException(Reason reason, String message)`; `enum Reason { MALFORMED, UNSUPPORTED_ALGORITHM, UNKNOWN_KEY, ALGORITHM_KEY_MISMATCH, MISSING_COMPONENT, DIGEST_MISMATCH, MISSING_DIGEST, EXPIRED, STALE, FUTURE_CREATED, INVALID_SIGNATURE, REPLAY, SCHEME_NOT_ALLOWED }`.

- [ ] **Step 1: Write failing tests.** `StructuredFieldsTest`: parses `sig1=("@method" "@authority" "@path");created=1618884473;keyid="test-key-rsa-pss"` into one `InnerList` with three string items and two params (Long, String); parses `sig1=:base64:` into a byte sequence; serializes back to the same text; rejects unterminated string (Review Focus 4: `MALFORMED`). `DigestsTest`: `contentDigestSha256OfRfc9530Example` (body `{"hello": "world"}` → `sha-256=:X48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE=:`), `verifyAcceptsMatchingAndRejectsTampered`, `legacyDigestRoundTrip`. `KeysTest`: generate RSA, EC P-256, Ed25519 keypairs with `KeyPairGenerator`, export to PEM (write a test helper), parse back and compare encodings; parse an RSA JWK and an `oct` JWK; `KeyType.of` for each. `AlgorithmTest`: `fromId` for every id in the spec table; `ED25519` is both schemes.
- [ ] **Step 2: Run** `mvn -q test -pl core` → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(core): structured fields, digests, key parsing and algorithms`.

### Task 15: Core signing and verification for both schemes

**Files:**
- Create: `core/.../HttpMessageView.java`, `SimpleHttpMessage.java`, `SignatureSpec.java`, `SigningKey.java`, `VerificationKey.java`, `SignedHeaders.java`, `Signer.java`, `HttpSigner.java`, `Verifier.java`, `HttpVerifier.java`, `VerificationPolicy.java`, `VerificationResult.java`, `KeyResolver.java`, `NonceCheck.java`, `SignatureBase.java`, `CavageSignature.java`, `Jca.java`
- Test: `core/src/test/java/.../Rfc9421VectorsTest.java`, `CavageTest.java`, `VerifierPolicyTest.java`; resources `core/src/test/resources/rfc9421/` (key PEMs and vectors extracted from `https://www.rfc-editor.org/rfc/rfc9421.txt` Appendix B: `test-key-rsa.pem`, `test-key-rsa-pss.pem` (private+public), `test-key-ecc-p256.pem`, `test-key-ed25519.pem`, `test-shared-secret` base64, and the B.2.x `Signature-Input`/`Signature` lines).

**Interfaces:**
- `interface HttpMessageView { String method(); URI targetUri(); Headers headers(); Optional<BodyDigest> body(); static HttpMessageView of(String method, URI target, Headers headers, byte[] body); }` (`body` empty when `body == null || body.length == 0`).
- `record SignatureSpec(Scheme scheme, String keyId, Algorithm alg, List<Component> covered, Instant created, Optional<Instant> expires, Optional<String> nonce, Optional<String> tag) { static SignatureSpec rfc9421(String keyId, Algorithm alg, Instant created, boolean hasBody) /* covered: @method, @target-uri, content-digest if body */; static SignatureSpec cavage(String keyId, Algorithm alg, Instant created, boolean hasBody) /* (request-target), host, date, digest if body */; SignatureSpec withCovered(List<Component>); SignatureSpec withExpires(Instant); SignatureSpec withNonce(String); SignatureSpec withTag(String); }`
- `record SigningKey(String keyId, Algorithm algorithm, java.security.Key key)` (constructor validates `KeyType.of(key)` matches `algorithm.keyType()` unless `HS2019`).
- `record VerificationKey(String keyId, Optional<Algorithm> algorithm, java.security.Key key)`.
- `record SignedHeaders(Map<String,String> headers) { void applyTo(BiConsumer<String,String>); }` — contains `Signature-Input`+`Signature` (RFC 9421) or `Signature` (Cavage), plus `Content-Digest` and/or `Digest` when a body is present, and `Date` for Cavage when the message lacks one. Label `sig1`.
- `interface Signer { SignedHeaders sign(HttpMessageView msg, SignatureSpec spec, SigningKey key); }`, `class HttpSigner implements Signer`.
- `interface KeyResolver { Optional<VerificationKey> resolve(String keyId); }`
- `interface NonceCheck { boolean firstUse(String nonce, Instant expires); }`
- `record VerificationPolicy(Set<Scheme> schemes, Duration clockSkew, Duration maxAge, boolean requireDigest, Set<Component> requiredComponents, KeyResolver keys, Optional<NonceCheck> nonces, Clock clock) { static VerificationPolicy defaults(KeyResolver keys) /* both schemes, 5m, 1h, requireDigest true, requiredComponents empty, no nonces, system UTC clock */; with… helpers }`
- `record VerificationResult(Scheme scheme, String keyId, Algorithm alg, List<Component> covered, Instant created, VerificationKey key)`
- `interface Verifier { VerificationResult verify(HttpMessageView msg, VerificationPolicy policy) throws HttpSignatureException; }`, `class HttpVerifier implements Verifier` — scheme detection: `Signature-Input` present → RFC9421; lone `Signature` → CAVAGE; neither → `MALFORMED`. Steps: scheme allowed; parse; resolve key (`UNKNOWN_KEY`); algorithm from header or key (`UNSUPPORTED_ALGORITHM`; Cavage `hs2019` → derived from key type: RSA→`RSA_PSS_SHA512`, ED25519→`ED25519`, EC_P256→`ECDSA_P256_SHA256` (DER), EC_P384→`ECDSA_P384_SHA384` (DER)); algorithm vs key type (`ALGORITHM_KEY_MISMATCH`, also when the key carries an algorithm that differs); required components present: RFC 9421 needs `@method` and (`@target-uri` or `@authority`+`@path`) and `content-digest` with a body; Cavage needs `(request-target)`, `host`, `date` and `digest` with a body; plus `policy.requiredComponents` (`MISSING_COMPONENT`); body digest header present and matching (`MISSING_DIGEST`/`DIGEST_MISMATCH`; with `requireDigest=false` a body without a digest header passes); `created` (RFC 9421 param; Cavage `created` param else `Date` header in RFC 7231 format) within `now - maxAge .. now + clockSkew` (`STALE`/`FUTURE_CREATED`; unparseable or missing → `STALE`), `expires` ≥ now (`EXPIRED`); nonce first use (`REPLAY`); cryptographic check (`INVALID_SIGNATURE`), HMAC via `MessageDigest.isEqual`.
- `SignatureBase`: `String rfc9421(HttpMessageView msg, List<Component> covered, String signatureParams)` producing lines `"<component>": <value>` and final `"@signature-params": <params>`; derived components `@method`, `@target-uri`, `@authority`, `@scheme`, `@request-target`, `@path`, `@query`, `@query-param;name="x"`; header values via `Headers.combined`. `String cavage(HttpMessageView msg, List<String> headers, Optional<Long> created, Optional<Long> expires)` with `(request-target)` → `method.lowercase + " " + path[?query]`, `(created)`, `(expires)`, and header lines `name: value`.
- `CavageSignature`: `record Parsed(String keyId, Optional<String> algorithm, List<String> headers, byte[] signature, Optional<Long> created, Optional<Long> expires)`; `static Parsed parse(String headerValue)` (comma-separated `k=v`, quoted values; default `headers` = `(created)` per draft? **Decision:** default `headers` = `date`); `static String serialize(...)`.
- `Jca`: `byte[] sign(Algorithm alg, Key key, byte[] data)` / `boolean verify(Algorithm, Key, byte[] data, byte[] sig)`; mapping: `RSA_V1_5_SHA256`/`RSA_SHA256` → `SHA256withRSA`; `RSA_PSS_SHA512` → `RSASSA-PSS` + `PSSParameterSpec("SHA-512", "MGF1", MGF1ParameterSpec.SHA512, 64, 1)`; `ECDSA_P256_SHA256` → `SHA256withECDSAinP1363Format` under RFC 9421 and `SHA256withECDSA` (DER) under Cavage (pass `Scheme`); `ECDSA_P384_SHA384` → `SHA384withECDSAinP1363Format` / `SHA384withECDSA`; `ED25519` → `Ed25519`; `HMAC_SHA256` → `HmacSHA256`.

- [ ] **Step 1: Download RFC 9421** to the scratchpad and copy Appendix B keys and the B.2.x vectors into test resources (B.1 keys; B.2.1 minimal `rsa-pss-sha512`, B.2.2 selective, B.2.3 full coverage, B.2.4 `ecdsa-p256-sha256`, B.2.5 `hmac-sha256`, B.2.6 `ed25519`; the request is `POST /foo?param=Value&Pet=dog` on `example.com` with the `{"hello": "world"}` body and headers from B.2).
- [ ] **Step 2: Write failing tests.** `Rfc9421VectorsTest`: for each of B.2.1–B.2.6 `verifiesPublishedSignature` (build the message exactly as in the RFC, policy with a resolver returning the matching key, clock fixed to `1618884473` + 10s, `requireDigest=false` where the vector does not cover content-digest); `hmacVectorSignsByteForByte` and `ed25519VectorSignsByteForByte` (deterministic: our `Signature` header equals the RFC's); `rsaPssSignatureVerifiesWithOurOwnSigner` (non-deterministic: sign then verify). `CavageTest`: `signsAndVerifiesRsaSha256` with generated key; `signatureBaseMatchesDraft12Example` (base for `(request-target)`, `host`, `date` on `POST /foo?param=value&pet=dog`, host `example.com`, date `Sun, 05 Jan 2014 21:31:40 GMT` equals the three-line string from the draft); `hs2019UsesKeyType`; `missingDateIsStale` (Review Focus 5); `unparseableDateIsStale`. `VerifierPolicyTest` (RFC 9421 with generated Ed25519 key): `tamperedBodyFails` (DIGEST_MISMATCH), `wrongAlgorithmFails` (sign with ed25519, declare `rsa-v1_5-sha256` → ALGORITHM_KEY_MISMATCH), `staleCreatedFails` (created 2h ago → STALE), `futureCreatedBeyondSkewFails`, `missingCoveredComponentFails` (covered only `@method` → MISSING_COMPONENT), `expiredFails`, `unknownKeyFails`, `replayFails` (nonce used twice), `schemeNotAllowedFails`, `malformedSignatureInputFails` (Review Focus 4 → MALFORMED, not an unchecked parse error).
- [ ] **Step 3: Run** → FAIL. **Step 4: Implement.** **Step 5: Run** `mvn -q test -pl core` → PASS.
- [ ] **Step 6: Commit** `feat(core): RFC 9421 and Cavage signing and verification`.

### Task 16: Runtime config, SPIs, static keys and nonce store

**Files:**
- Create under `runtime/src/main/java/org/sitenetsoft/quarkus/httpsignatures/`: `HttpSignaturesConfig.java` (runtime root), `HttpSignaturesBuildTimeConfig.java`, `ReactiveKeyResolver.java`, `SigningKeyProvider.java`, `ReactiveSigningKeyProvider.java`, `NonceStore.java`, `ReactiveNonceStore.java`, `StaticKeys.java`, `ConfigKeyResolver.java`, `ConfigSigningKeyProvider.java`, `InMemoryNonceStore.java`, `CachingKeyResolver.java`, `VerificationPolicies.java`, `SignatureContext.java`
- Modify: `deployment/.../HttpSignaturesProcessor.java` (AdditionalBeans, unremovable SPI types)
- Test: `deployment/src/test/java/.../StaticKeysTest.java`

**Interfaces:**
- `@ConfigMapping(prefix="quarkus.http-signatures") @ConfigRoot(phase=RUN_TIME) interface HttpSignaturesConfig { Verify verify(); Sign sign(); Map<String,KeyConfig> keys(); KeyCache keyCache(); Nonce nonce(); Map<String,ClientConfig> clients(); interface Verify { @WithDefault("5m") Duration clockSkew(); @WithDefault("1h") Duration maxAge(); @WithDefault("true") boolean requireDigest(); @WithDefault("rfc9421,cavage") List<Scheme> schemes(); Optional<List<String>> paths(); } interface Sign { @WithDefault("rfc9421") Scheme preferredScheme(); @WithDefault("true") boolean fallback(); } interface KeyConfig { String location(); Optional<Algorithm> algorithm(); } interface KeyCache { @WithDefault("1h") Duration ttl(); } interface Nonce { @WithDefault("false") boolean enabled(); } interface ClientConfig { Optional<String> keyId(); Optional<Scheme> scheme(); Optional<List<String>> components(); } }` — `Scheme`/`Algorithm` need converters (`@WithConverter` or implement `org.eclipse.microprofile.config.spi.Converter` registered via `META-INF/services`); `Scheme` config values are `rfc9421` and `cavage` (lowercase).
- `@ConfigMapping(prefix="quarkus.http-signatures") @ConfigRoot(phase=BUILD_TIME) interface HttpSignaturesBuildTimeConfig { @WithDefault("true") boolean enabled(); }`
- SPIs: `KeyResolver` is the core interface; `interface ReactiveKeyResolver { Uni<Optional<VerificationKey>> resolve(String keyId); }`; `interface SigningKeyProvider { Optional<SigningKey> select(URI target, Optional<String> clientName); }`; `interface ReactiveSigningKeyProvider { Uni<Optional<SigningKey>> select(URI target, Optional<String> clientName); }`; `interface NonceStore extends NonceCheck {}`; `interface ReactiveNonceStore { Uni<Boolean> firstUse(String nonce, Instant expires); }`.
- `@ApplicationScoped class StaticKeys`: loads each `keys.<id>.location` at first use (`file:` path, classpath resource, or plain path), via `Keys.parsePem`/`parseJwk` by extension `.pem`/`.jwk`/`.json`; exposes `Optional<SigningKey> signing(String id)` (private/secret keys) and `Optional<VerificationKey> verification(String id)` (public key derived from private where possible: RSA via `RSAPublicKeySpec` from CRT key; EC/Ed25519 require a public key too → **Decision:** for EC/Ed25519 the PEM file may contain both `PRIVATE KEY` and `PUBLIC KEY` blocks; `Keys.parsePemAll(String) → List<Key>`).
- `@DefaultBean @ApplicationScoped class ConfigKeyResolver implements KeyResolver` → `StaticKeys.verification(id)`; `@DefaultBean @ApplicationScoped class ConfigSigningKeyProvider implements SigningKeyProvider` → client's `keyId` else the single configured key else the first by name; `@DefaultBean @ApplicationScoped class InMemoryNonceStore implements NonceStore` (Caffeine with per-entry expiry).
- `@ApplicationScoped class CachingKeyResolver implements KeyResolver` wrapping `@All List<KeyResolver>` and `@All List<ReactiveKeyResolver>` (reactive ones awaited on a worker), Caffeine cache TTL `keyCache.ttl`; `void invalidate(String keyId)`.
- `@ApplicationScoped class VerificationPolicies { VerificationPolicy current(); }` building from config + `CachingKeyResolver` + nonce store when enabled.
- `@RequestScoped class SignatureContext { boolean isVerified(); Optional<VerificationResult> result(); void set(VerificationResult) }`.
- Deployment: `AdditionalBeanBuildItem` for all of the above (unremovable), `UnremovableBeanBuildItem.beanTypes(KeyResolver, ReactiveKeyResolver, SigningKeyProvider, ReactiveSigningKeyProvider, NonceStore, ReactiveNonceStore)`.

- [ ] **Step 1: Write failing `QuarkusUnitTest`** `StaticKeysTest` with test resources `keys/rsa.pem` (PKCS#8 + public), `keys/ed25519.pem`, `keys/hmac.jwk`; config `quarkus.http-signatures.keys.rsa.location=keys/rsa.pem`, `keys.ed.location=keys/ed25519.pem`, `keys.hook.location=keys/hmac.jwk`, `keys.hook.algorithm=hmac-sha256`: `loadsRsaSigningAndVerificationKeys`, `loadsEd25519Pair`, `loadsHmacSecret`, `defaultResolverFindsConfiguredKey`, `defaultProviderPrefersClientKeyId` (`clients.partner.key-id=hook`), `nonceStoreRejectsSecondUse`.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(http-signatures): config, SPIs, static keys, nonce store`.

### Task 17: Vert.x verification: auth mechanism, identity provider, path handler

**Files:**
- Create: `runtime/.../security/HttpSignatureAuthenticationRequest.java`, `HttpSignatureAuthenticationMechanism.java`, `HttpSignatureIdentityProvider.java`, `runtime/.../vertx/VertxMessageView.java`, `VertxSignatureHandler.java`, `HttpSignaturesRecorder.java`
- Modify: `HttpSignaturesProcessor` (register security beans only when `capabilities.isPresent(Capability.SECURITY)`; register `RouteBuildItem`s for `verify.paths` — paths are runtime config, so register one absolute route per pattern via a recorder-produced handler is impossible at build time; **Decision:** register a single global `RouteBuildItem` with `routeFunction(router -> router.route().order(-1))` whose handler checks `verify.paths` patterns (Vert.x style `/inbox/*`) at runtime and calls `ctx.next()` for non-matching paths)
- Test: `deployment/src/test/java/.../VertxPathVerificationTest.java`, `SecurityIdentityTest.java` (with `quarkus-security`, a `@Authenticated` Quarkus REST resource returning the principal name)

**Interfaces:**
- `VertxMessageView.of(RoutingContext ctx, byte[] bodyOrNull) → HttpMessageView` (target URI from `ctx.request().absoluteURI()`; headers from `ctx.request().headers()`).
- `HttpSignatureAuthenticationMechanism implements HttpAuthenticationMechanism`: when no `Signature` header → `Uni.createFrom().nullItem()`; else obtain the body (if `ctx.body().buffer()` is non-null use it; else if the request has a body (`Content-Length` > 0 or `Transfer-Encoding: chunked`) and a digest header is present, read it with `ctx.request().body()` and make it available to downstream by `((io.vertx.ext.web.impl.RoutingContextInternal) ctx).setBody(buffer)` — if that method is absent in the Vert.x version in use, store under `ctx.put("org.sitenetsoft.quarkus.httpsignatures.body", buffer)` and have `VertxSignatureHandler` and the REST filter read it; verify with `HttpVerifier`; on `HttpSignatureException` → `AuthenticationFailedException` (401); on success → `identityProviderManager.authenticate(new HttpSignatureAuthenticationRequest(result))`. `getCredentialTransport` → `OTHER_HEADER` `signature`. `getChallenge` → 401 with `WWW-Authenticate: Signature`. In dev/test mode (`LaunchMode.current().isDevOrTest()`) log the signature base at DEBUG on failure; never log the signature value.
- `HttpSignatureIdentityProvider implements IdentityProvider<HttpSignatureAuthenticationRequest>`: builds `QuarkusSecurityIdentity` with `QuarkusPrincipal(result.keyId())`, attribute `http-signature` = result, role `signed`; also sets `SignatureContext` when the request scope is active (`Arc.container().requestContext().isActive()`).
- `VertxSignatureHandler implements Handler<RoutingContext>`: for matching `verify.paths`, run the same verification; failure → `401` empty body; success → `ctx.put("org.sitenetsoft.quarkus.httpsignatures.result", result)` and `ctx.next()`.

- [ ] **Step 1: Write failing tests.** `VertxPathVerificationTest` (`quarkus.http-signatures.verify.paths=/inbox/*`, static Ed25519 key `k1`, a test route `/inbox/alice` registered via `@Observes Router` returning 200 `ok`): `unsignedIs401`, `signedRfc9421Is200` (sign with `HttpSigner` + `java.net.http`-free RestAssured call adding the headers), `signedCavageIs200`, `tamperedBodyIs401`, `garbageSignatureIs401NotError` (Review Focus 4), `otherPathsUntouched` (`/public` → 200 without signature). `SecurityIdentityTest`: `@Authenticated` endpoint `/whoami` returns `k1` when signed; 401 unsigned.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** `feat(http-signatures): Vert.x verification handler and Quarkus Security mechanism`.

### Task 18: `@SignatureRequired` for Quarkus REST

**Files:**
- Create: `runtime/.../rest/SignatureRequired.java` (`@Target({METHOD, TYPE}) @Retention(RUNTIME)`, `String[] components() default {}`), `runtime/.../rest/SignatureRequiredFilter.java` (`@Provider ContainerRequestFilter` registered only when `Capability.REST` present; reads `SignatureContext` or the `RoutingContext` attribute; aborts `401` when unverified or when any required component is not in `result.covered()`), deployment step adding `ResteasyReactiveJaxrsProviderBuildItem`/`AdditionalBeanBuildItem` and scanning `@SignatureRequired` with the Jandex index to produce a `AnnotationsTransformerBuildItem` is unnecessary — the filter uses `ResourceInfo` to look up the annotation at runtime.
- Test: `deployment/src/test/java/.../SignatureRequiredTest.java` — resource `/hook` with `@SignatureRequired(components={"@method","@target-uri","content-digest"})` POST echoing the body: `signedWithComponentsIs200`, `signedWithoutContentDigestIs401`, `unsignedIs401`, `hmacWebhookWorks` (key `hook` from `hmac.jwk`; satisfies the spec's HMAC webhook criterion with no ActivityPub dependency).

- [ ] **Steps 1–5:** failing tests → run → implement → run → commit `feat(http-signatures): @SignatureRequired filter for Quarkus REST`.

### Task 19: Signing outgoing requests: helper, REST client filter, double knocking

**Files:**
- Create: `runtime/.../client/RequestSigner.java`, `SchemeMemory.java`, `SigningHttpClient.java` (java.net.http), `VertxRequestSigner.java` (Vert.x `HttpRequest<Buffer>`), `rest/HttpSignatureClientFilter.java`, `rest/SignRequests.java`
- Modify: `HttpSignaturesProcessor` (when `Capability.REST_CLIENT` present: `AnnotationsTransformerBuildItem` adding `@RegisterProvider(HttpSignatureClientFilter.class)` to interfaces annotated `@SignRequests`; `AdditionalBeanBuildItem` for the filter)
- Test: `deployment/src/test/java/.../OutboundSigningTest.java`

**Interfaces:**
- `@ApplicationScoped class SchemeMemory { Scheme preferredFor(String host); void remember(String host, Scheme scheme); }` (Caffeine, 24h).
- `@ApplicationScoped class RequestSigner { SignedHeaders sign(String method, URI target, Headers existing, byte[] body, Optional<String> clientName, Scheme scheme); Scheme schemeFor(URI target, Optional<String> clientName); }` — picks the key through `SigningKeyProvider`/`ReactiveSigningKeyProvider` (`@All`, first non-empty), spec via `SignatureSpec.rfc9421/cavage` (client `components` override), adds `Date` when absent (Cavage needs it), `Content-Digest` (RFC 9421) and `Digest` (Cavage) for bodies.
- `@ApplicationScoped class SigningHttpClient { HttpResponse<byte[]> send(HttpClient client, HttpRequest request, byte[] body, Optional<String> clientName); }` — signs with `schemeFor`; on `401`/`403` and `sign.fallback`, re-signs with the other scheme, resends once, and `remember`s the scheme that answered non-401/403.
- `VertxRequestSigner.sign(HttpRequest<Buffer> request, URI target, byte[] body, Optional<String> clientName)` → applies headers (no retry; caller may use `SchemeMemory`).
- `HttpSignatureClientFilter implements ClientRequestFilter, WriterInterceptor, ClientResponseFilter`: `ClientRequestFilter` signs body-less requests; `WriterInterceptor` buffers the entity bytes, signs, sets headers, writes bytes; `ClientResponseFilter` on `401`/`403` flips `SchemeMemory` for the host (retry is left to the caller; documented in README). Client name from the `io.quarkus.rest.client.reactive` config key property if present, else `Optional.empty()`.
- `@Target(TYPE) @Retention(RUNTIME) @interface SignRequests { String client() default ""; }`.

- [ ] **Step 1: Write failing `QuarkusUnitTest`** with an embedded Vert.x server that verifies incoming requests using `HttpVerifier` and a static test key and that can be switched to "Cavage-only" (returns 401 for RFC 9421): `javaNetHelperSignsRfc9421`, `javaNetHelperFallsBackToCavageOnceAndRemembers` (server counter: first request RFC9421 → 401, second Cavage → 200; a third call goes straight to Cavage: exactly 3 requests total), `vertxSignerProducesVerifiableHeaders`, `restClientFilterSignsPostWithDigest` (REST Client interface annotated `@SignRequests` + `@RegisterRestClient(baseUri=...)`), `restClientFilterSignsGet`.
- [ ] **Step 2: Run** → FAIL. **Step 3: Implement.** **Step 4: Run** `mvn -q test` → PASS.
- [ ] **Step 5: Commit** `feat(http-signatures): outbound signing helpers, REST client filter, double knocking`.

### Task 20: Dev UI, CI, docs, webhook example

**Files:**
- Create: `deployment/.../devui/HttpSignaturesDevUiProcessor.java` (card: config in effect via `Page.webComponentPageBuilder().title("Verify").componentLink("qwc-http-signatures-verify.js")`), `deployment/src/main/resources/dev-ui/qwc-http-signatures-verify.js` (textarea for a raw request → JSON-RPC `verify` returning `{ok, reason, signatureBase}`), `runtime/.../devui/HttpSignaturesJsonRpcService.java` (`public JsonNode verify(String rawRequest)` parsing a `METHOD SP target` first line, headers, blank line, body; `public JsonNode policy()`).
- Create: `.github/workflows/build.yml`, `docs/antora.yml`, `docs/modules/ROOT/pages/index.adoc`, `examples/webhook/` (minimal Quarkus app: one `@SignatureRequired` POST `/webhook` with an HMAC key from config, README showing a `curl` signed by the `SigningHttpClient` helper via a tiny `main`), README with the config table and the "double knocking" note.

- [ ] **Step 1: Implement. Step 2: Verify** `mvn -q -T 8 install` → BUILD SUCCESS and all tests pass; `cd examples/webhook && mvn -q test` → PASS.
- [ ] **Step 3: Commit** `feat(http-signatures): dev ui verify page, ci, docs and webhook example`.

---

## Task 21: Top-level README and final verification

**Files:** `OpenWebExt/README.md` (what lives here, build order, host constraints), keep the spec file.

- [ ] **Step 1:** Run the three builds in order, one at a time: safe-fetch `install`, webfinger `install`, http-signatures `install` (each with `-T 8`).
- [ ] **Step 2:** Record pass/fail counts and anything skipped (native builds, Mastodon interop) in the final report.
