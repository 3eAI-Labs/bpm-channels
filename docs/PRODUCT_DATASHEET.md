# Product Datasheet

**nats-bpm-channels 0.13.0** · NATS.io messaging and database offload for open-source BPM engines
Business Source License 1.1 (Apache 2.0 through 0.8.1) · 3eAI Labs Ltd · Maven Central namespace `com.3eai-labs` (Apache-era releases)

---

## Overview

A Spring Boot library that connects a BPM engine to NATS.io and moves high-volume engine work off
the relational database transaction. Four engines are supported through their public extension
points; no engine is forked or patched. Every capability is independent and opt-in: the offload
paths are off until switched on, so adding the library for one of them does not start the others.

---

## Artifacts

| Artifact | Purpose | On Maven Central (through 0.8.1) |
|---|---|---|
| `com.3eai-labs:nats-channel-parent` | Dependency and plugin management (pom) | Yes |
| `com.3eai-labs:nats-core` | Connection, auth, TLS, JetStream management, metrics, headers | Yes |
| `com.3eai-labs:flowable-nats-channel` | Flowable 7.x Event Registry adapter | Yes |
| `com.3eai-labs:camunda-nats-channel` | Camunda 7.x adapter | Yes |
| `com.3eai-labs:cibseven-nats-channel` | CIBSeven 2.x adapter | Yes |
| `com.3eai-labs:nats-history-projection` | History projection, retention, erasure, reconciliation | Yes |
| `com.3eai-labs:cadenzaflow-nats-channel` | CadenzaFlow 1.x adapter | Yes (from 0.8.0) |

All published artifacts ship `.jar`, `-sources.jar`, `-javadoc.jar` and `.pom`, each GPG-signed.
Signing key `E610505884534DB9` (`oss@3eai-labs.com`) is on `keyserver.ubuntu.com`.

Releases from 0.10.0 are not published to Maven Central: build from source with `mvn install` on
Java 21 (free for development and testing under the BSL 1.1); production artifacts ship with the
commercial license.

---

## Capability matrix

### Messaging — all engines

| Feature | Detail |
|---|---|
| Core NATS pub/sub | Inbound and outbound channels; queue groups |
| JetStream durable delivery | Ack/nak, exponential backoff (`nakWithDelay`), configurable `ackWait`, `maxDeliver`, `deliverPolicy` |
| Dead-letter queue | JetStream primary with core-NATS fallback; in-band detection at `maxDeliver + 1`; no dlq-of-dlq, never ack-drop |
| Request-reply | BPMN service task to external worker, any language |
| Message correlation | `RuntimeService.createMessageCorrelation()`; sets `natsPayload` and `natsSubject` |
| Auth | Username/password, token, credentials file, NKey |
| Transport security | TLS with client certificate, key and CA |
| Concurrency | Java 21 virtual threads for non-blocking I/O |

### Database offload — Camunda 7, CIBSeven and CadenzaFlow

Flowable ships the messaging set above and external-worker dispatch (next table); history,
large-variable and outbox parity is on the roadmap.

| Capability | Version | Prefix | Effect |
|---|---|---|---|
| External task dispatch over JetStream | v0.2.0 | `spring.nats.<engine>.a2` | Workers stop polling the engine database; `fetchAndLock` = 0 on the happy path |
| History offload | v0.3.0 | `spring.nats.<engine>.history` | `ACT_HI_*` traffic is published to NATS and projected into a separate PostgreSQL database; it leaves the engine database per history class, after that class is cut over |
| Large variable externalisation | v0.4.0 | `history.large-variable` | Variables above threshold move to a content-addressed store with SHA-256 dedup and reference counting |
| Outbound handoff | v0.5.0 | `spring.nats.outbound` | Dual-write-safe delivery of message-throw and send-task |

### Dispatch, definitions and fleet topology — 0.10.0 to 0.13.0

| Capability | Version | Engines | Prefix | Effect |
|---|---|---|---|---|
| Flowable external-worker dispatch over JetStream | v0.10.0 | Flowable | `spring.nats.flowable.external-worker` | `flowable:type="external-worker"` jobs are push-dispatched post-commit; the polling `acquireAndLock` loop is replaced |
| Startup topology self-check | v0.10.0 | All | — | Boot-time comparison of the live broker against multi-node requirements; one WARN per finding with a stable code; reports, never repairs |
| Declarative event/channel definitions | v0.11.0 | Camunda lineage | `spring.nats.<engine>.eventing` | Portable JSON subset of Flowable's `.event`/`.channel` formats, deployed through the engine's own deployment API; one definition format across all four engines |
| Instance sharding (single fleet) | v0.12.0 | Camunda lineage | `spring.nats.<engine>.sharding` | N independent engine+database pairs behind one NATS routing fabric: businessKey-hashed placement, stateless router, envelope-addressed replies, birth guard and hard boot validator; off by default |
| Asynchronous post-commit publishers | v0.13.0 | All | `<prefix>.async-publish` | Bounded in-flight publish on every post-commit path; guarantees unchanged; `=false` restores the synchronous path per family |

---

### Data governance

| Feature | Detail |
|---|---|
| Retention | Per class; audit-critical default `P7Y`, bulk default 90 days |
| Erasure | Reference-counted deletion — releasing the last reference deletes the payload itself |
| Pseudonymisation | Opt-in, tenant-keyed with key versioning; separate vault datasource with column encryption key reference |
| Reconciliation | Scheduled (default `0 0 3 * * *`) with clean-streak targets before cutover |
| Cutover | Gradual, dual-store with reconciliation gates and a query API that hides which store answered |

---

## Delivery guarantees

| Path | Guarantee | Cost |
|---|---|---|
| Transactional outbox + leader relay | At-least-once, no loss | One row insert inside the transaction |
| Post-commit publish | At-most-once | No additional database write |
| JetStream consume | At-least-once | Consumers must be idempotent |
| External task net path | At-least-once | post-commit at-most-once + orphan sweep at-least-once |

Selection is per history class or per message type, by configuration. There is no exactly-once mode;
duplicates are suppressed with `Nats-Msg-Id` and idempotent complete/correlate.

---

## Reliability evidence

| Property | Result | Method | Suite |
|---|---|---|---|
| Test suite | 1,988 tests passing (0.13.0, measured 2026-08-25) | Testcontainers (PostgreSQL, NATS) + fault injection; `mvn verify`, seven modules | default |
| Line coverage | ≥ 90% per production module, 93.0% weighted (measured at 0.5.1; not re-measured since) | JaCoCo; branch 80.4% | default |
| Audit-critical loss on relay failover | RPO = 0 | Three competing relay instances race for one real JetStream KV lease; successor drains every seeded row | `bench` |
| Recovery time | ≈ lease TTL — measured 60.4 s against a 60 s TTL | TTL-expiry-driven handover; a floor, not a bound with headroom | `bench` |
| Leader-lease split-brain | 0 | N-candidate race against real KV compare-and-swap | `reliability` |

The `reliability` and `bench` suites are excluded from the default `mvn verify` run and from CI, so
they are not counted in the 1,988 figure. The failover test exercises the application-level lease
and outbox contract: the test broker is a single node and the lease bucket is created with one
replica. Production auto-configuration provisions leader buckets with three replicas, but a
replicated-broker failover has not been measured.

**Measured on one host, published with its scope.** Throughput ceilings, latency under overload and
database cost against the engine's native external-task polling were measured on a single 16-vCPU
host (100,000 process instances per cell) and are published with the 0.13.0 announcement at
<https://3eai-labs.com/en/blog/nats-bpm-channels-0130>, together with what the method does not
cover. Absolute values do not transfer to other hardware; directions do. The `nats-bpm-bench`
module in the repository compares native polling against JetStream push in your own environment.

---

## Technology stack

| Component | Technology | Version | Licence |
|---|---|---|---|
| Runtime | Java | 21+ | — |
| Framework | Spring Boot | 3.3+ | Apache 2.0 |
| Messaging | NATS.io + JetStream | 2.10+ | Apache 2.0 (CNCF incubating) |
| NATS client | `io.nats:jnats` | 2.20+ | Apache 2.0 |
| Engine | Flowable | 7.1+ | Apache 2.0 |
| Engine | Camunda 7 | 7.24+ | Apache 2.0 |
| Engine | CIBSeven | 2.2+ | Apache 2.0 |
| Engine | CadenzaFlow | 1.2+ | Apache 2.0 |
| Projection store | PostgreSQL | 12+ | PostgreSQL Licence |
| Metrics | Micrometer | via Spring Boot | Apache 2.0 |
| Test | JUnit 5, Testcontainers, Mockito, AssertJ | — | — |

Total licence cost for the stack: none.

---

## Prerequisites

- Java 21 or later
- Spring Boot 3.x application
- NATS server 2.10+ (2.10 is required for `nakWithDelay`)
- One supported BPM engine
- PostgreSQL 12+ — only if history offload or large variable externalisation is enabled
- `spring.threads.virtual.enabled: true` recommended
- Actuator and a Micrometer registry in the host application, if metrics are to be exported

---

## Deployment

| Option | Notes |
|---|---|
| Embedded library | The adapter runs inside the application hosting the engine; it is not a separate service |
| Additional infrastructure | NATS cluster and, for offload, a projection PostgreSQL instance |
| NATS replication | Factor 3 — recommended production setting, provisioned by the auto-configuration for the leader-election buckets the KV lease depends on. Not the configuration the reliability evidence was produced against; that measurement ran on a single-node broker |
| Workers | Independent processes in any language; require only a NATS client |
| Projection scaling | By partition, default 8 |
| Container / Kubernetes | No special requirement — a standard Spring Boot deployment |

---

## Observability

41 Micrometer metrics. Recommended alerts:

| Metric | Signals |
|---|---|
| `nats.history.outbox.oldest_row_age_seconds` | History relay stalled — data at risk of falling behind |
| `nats.outbound.outbox.oldest_row_age_seconds` | Outbound relay stalled |
| `nats.history.projection.lag_seconds` | Projection falling behind the stream |
| `nats.a2.sweep.oldest_orphan_age_seconds` | External tasks dispatched but not completed |
| `nats.jetstream.dlq.publish.failures` | DLQ route itself failing — the last line before loss |
| `nats.connection.slow.consumers` | Consumer cannot keep up with the subject |

Structured logging with SLF4J `kv()` and `trace_id` in MDC.

---

## Configuration surface

Documented properties across these prefixes:

`spring.nats` · `spring.nats.<engine>.subscriptions` · `spring.nats.<engine>.a2` ·
`spring.nats.<engine>.history` · `spring.nats.<engine>.eventing` · `spring.nats.<engine>.sharding` ·
`spring.nats.flowable.external-worker` · `spring.nats.outbound` · `history.projection` ·
`history.large-variable` · `history.retention` · `history.reconciliation` · `history.cutover` ·
`history.vault.datasource` · `<prefix>.async-publish`

Full reference with types and defaults: [User Guide](user/USER_GUIDE.md).

---

## Licensing and support

Business Source License 1.1 from 0.10.0: free for development, testing and evaluation with the
full source public; production use requires a commercial license from 3eAI Labs, which includes a
measured proof-of-concept on your workload and a support agreement. Each release converts to the
Apache License 2.0 four years after its publication — the cap is written into the license text.
Releases through 0.8.1 remain Apache 2.0 on Maven Central.

Commercial services from 3eAI Labs: adoption and integration consulting, NATS deployment and
tuning, and support agreements.

Repository <https://github.com/3eAI-Labs/nats-bpm-channels> · Commercial license and pilot
<https://3eai-labs.com> · Contact `oss@3eai-labs.com`

---

*Copyright 2026 3eAI Labs Ltd. Camunda is a trademark of Camunda Services GmbH; Flowable of
Flowable AG; CIBSeven is a project of CIB software GmbH.*
