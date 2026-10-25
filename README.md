# OpenWebExt: Quarkus extensions for ActivityPub, AT Protocol and web standards

This folder holds the architecture spec, the implementation plans, and one git repository per extension (the spec
calls for one repo per Tier 1 standard). The sub-repositories are ignored by this root repository and have their
own history.

| Repo | Tier | Artifacts | Status |
| --- | --- | --- | --- |
| `quarkus-safe-fetch/` | 1 | `quarkus-safe-fetch`, `-deployment` | Milestone 1, built and tested |
| `quarkus-webfinger/` | 1 | `quarkus-webfinger`, `-deployment` | Milestone 1, built and tested |
| `quarkus-http-signatures/` | 1 | `http-signatures-core`, `quarkus-http-signatures`, `-deployment`, `examples/webhook` | Milestone 1, built and tested |
| `quarkus-activitypub/` | 2 | `activitystreams-model`, `quarkus-nodeinfo`, `quarkus-activitypub`, `quarkus-activitypub-hibernate`, `quarkus-activitypub-test`, `quarkus-activitypub-bom`, `integration-tests` | Milestone 2, built and tested |

Spec: `Quarkus Open Web Extensions — Architecture Spec.md`. Plans: `docs/superpowers/plans/`.

Milestone 2 (ActivityPub server-to-server): design `docs/superpowers/specs/2026-10-25-milestone-2-activitypub-s2s-design.md`,
plan `docs/superpowers/plans/2026-10-25-milestone-2-activitypub-s2s.md`. Its Tier 1 changes live on the `milestone-2`
branches of `quarkus-safe-fetch` and `quarkus-http-signatures` until they are merged.

## Building

All repos target Java 21 and Quarkus 3.40.1. Build in dependency order; `quarkus-webfinger` needs
`quarkus-safe-fetch` installed first, and `quarkus-activitypub` needs all three Tier 1 repos installed. On this workstation keep Maven at `-T 8` at most and run one heavy build at a
time.

```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
(cd quarkus-safe-fetch      && mvn -T 8 install)
(cd quarkus-webfinger       && mvn -T 8 install)
(cd quarkus-http-signatures && mvn -T 8 install && cd examples/webhook && mvn test)
(cd quarkus-activitypub     && mvn -T 8 install)
```

Native builds are a release requirement and run in each repo's GitHub Actions workflow; `native-image` is not
installed on this host.
