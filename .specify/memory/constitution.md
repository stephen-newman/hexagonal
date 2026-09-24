<!-- Sync Impact Report
Version change: 2.2.0 -> 2.3.0 (MINOR: five new principles, materially
  expanded guidance in eight existing principles, ten new quality gates)
Modified principles (rules added, no rule removed or redefined):
- VI. Kafka + AsyncAPI - per-topic key/ordering/competing-consumer policy,
  atomic duplicate suppression, named outbox relay, resumability and
  relay-lag alerting, broker-outage recoverability
- IX. Service Boundaries & Ownership - greenfield rule and strangler-fig
  migration guidance with recorded extraction order and glue choice
- XII. Independent Deployability & Evolution - deployment platform and
  service-discovery mechanism MUST be recorded; one database instance per
  service instance; registry MUST NOT be a routing single point of failure
- XIII. Sensitive Data Protection - read-once mechanics (single release,
  final, non-serialisable, transient fields, masked toString, defensive
  copies) and the recorded-exception path for repeated reads
- XIV. Untrusted Input & Valid-by-Construction Domain Model - value-based
  equality over a canonical form, construction as the only failure point,
  null is not absence, canonical/confusable text, parameterised data access
- XV. Secure HTTP Contracts & Response Minimisation - output encoding on
  rendered surfaces, page-bounded collection responses, 404-vs-403
  existence policy
- XVI. Identity-Ready, Default-Deny Authorization - per-instance
  object-level authorization with ownership predicate in the matrix,
  algorithm allow-list rejecting alg=none, provider-issued tokens preferred
- XVII. Secrets, Keys & Secure Baseline Configuration - CORS allow-list,
  centralised security headers, no-store for SENSITIVE responses, no tokens
  in URLs, header and TLS smoke test
Added principles:
- XVIII. Saga Design & Compensation (NON-NEGOTIABLE)
- XIX. Aggregate & Transaction Boundaries (NON-NEGOTIABLE)
- XX. Transport Security & Workload Identity (NON-NEGOTIABLE)
- XXI. API Lifecycle, Deprecation & Change Communication
- XXII. Security Governance: Risk Acceptance, Disclosure & Incident Response
Added guidance:
- Appendix A: outbox relay naming in `spring-boot-assembly`, and the ADR
  location for boundary/classification/sensitivity decisions
- Technology Constraints: service discovery and deployment platform, test
  tooling, transport security, event store, and ADR slots marked (decision
  needed); no library is pinned until the decision is recorded
- Development Workflow: test taxonomy and deployment-pipeline stages are
  recorded; gates (19) saga step semantics and compensation idempotency,
  (20) isolation countermeasure per anomaly, (21) aggregate boundary rules
  under ArchUnit, (22) ordering, duplicate suppression, and relay survival
  of a broker outage, (23) test taxonomy including provider-side contract
  verification, (24) pipeline stages with unchanged promotion, (25) transport
  security assertion, (26) object-level authorization and existence
  non-disclosure, (27) deprecation/sunset and de-provisioning, (28) ADR
  presence and no expired risk acceptance
- Governance: compliance review covers I-XXII; risk acceptance MUST expire
  and be re-reviewed; boundary, classification, and sensitivity decisions
  MUST be dated ADRs in the repository
Sources: Daniel Deogun, Dan Bergh Johnsson, Daniel Sawano, "Secure by Design"
  (Manning, 1st ed., 2019); Jose Haro Peralta, "Secure APIs: Design, Build,
  and Implement" (Manning, 1st ed., 2025); Chris Richardson, "Microservices
  Patterns: With examples in Java" (Manning, 1st ed., 2019). Rules are
  paraphrased from those sources; no book text is quoted.
Provenance correction: the non-enumerable identifier rule in Principle XV
  (server-generated value with at least 122 bits of randomness) is drawn from
  "Secure APIs" (predictable identifiers), not from "Secure by Design"; the
  rule itself is unchanged.
Removed sections: none
Follow-up TODOs:
- TODO(SECURITY_TOOLING): pin the contract-linting, fuzzing, secret-scanning,
  dependency-scanning, and encryption libraries in Technology Constraints.
- TODO(PLATFORM): record the service-discovery mechanism and the deployment
  platform (Principles XII and XX) before the first inter-service call.
- TODO(TEST_TOOLING): pin the acceptance-test and contract-test tooling
  (gate 23) before the first provider-side contract test is required.
- TODO(SPEC_001): refresh specs/001-register-corporate-behalf/spec.md for
  Principles XIII-XXII (non-enumerable application ID, anonymous-actor fields,
  validation order and size limits, audit-trail vs hard-delete tension, the
  manual queue as an audited operation, saga step classification, aggregate
  boundary and object-level matrix rows, and the deprecation policy).
NOTE: temporary scratch material for human review; MUST be removed before commit.
-->

# Spring Boot MicroServices Constitution (Hexagonal Core, Conventional Non-Core)

## Core Principles

### I. Architecture Layout & Packaging (NON-NEGOTIABLE)

In every core-domain service, each feature MUST be organized into three
distinct layers with clear structural separation: `domain` (core business
logic: entities, value objects, domain services, business rules), `ports`
(interfaces: inbound use-case ports and outbound driven ports), and
`adapters` (infrastructure, web, and database implementations plus
configuration).

Scope: this principle applies to CORE-domain services only; service
classification is defined in Principle VIII.

Rules:
- `domain` MUST NOT reference `adapters` packages or types; `ports` MUST
  NOT reference `adapters` packages or types.
- `adapters` MAY depend inward on `ports` and `domain`; NEVER the reverse.
- Infrastructure types (DTOs, entities, HTTP types) MUST be translated at
  adapter boundaries and MUST NEVER leak into `domain` or `ports`.
- Each bounded context MUST mirror the `domain` / `ports` / `adapters`
  package split so the boundary is verifiable by package inspection and
  ArchUnit tests.

Rationale: explicit layering makes the hexagonal boundary enforceable by
tooling and review, keeping business logic independent of delivery and
persistence mechanisms.

### II. Dependency Isolation - Framework-Free Core (NON-NEGOTIABLE)

Core-domain services only; service classification is defined in Principle
VIII.

The `domain` and `ports` layers MUST be pure Java. They MUST NEVER import
Spring Framework (`org.springframework.*`), JPA / Jakarta Persistence
(`jakarta.persistence.*`, `javax.persistence.*`), or any external framework
or infrastructure library.

Rules:
- Spring annotations including `@RestController`, `@Service`, `@Component`,
  `@Repository`, `@Configuration`, and `@Autowired` are FORBIDDEN in
  `domain` and `ports` and are permitted ONLY in `adapters` and dedicated
  configuration packages.
- Core logic MUST be unit-testable with plain JUnit without a Spring
  context; any test requiring Spring context for `domain`/`ports` is a
  violation.
- Boundary violations MUST fail the build via ArchUnit or equivalent
  package-dependency tests.

Rationale: a framework-free core guarantees portability, fast isolated
testing, and that framework upgrades never force business-logic rewrites.

### III. Ports & Inversion of Control

All cross-boundary communication MUST go through ports defined inside the
core and follow dependency inversion: the core owns the interface, adapters
provide the implementation.

Scope: this principle applies to CORE-domain services only (see Principle
VIII).

Rules:
- Primary/Driving ports (use cases invoked by controllers, CLI, schedulers)
  MUST be interfaces defined inside the core (e.g., `ports.in.*UseCase`).
  Controllers and other inbound adapters MUST depend only on these
  interfaces, NEVER on concrete domain services directly across packages.
- Secondary/Driven ports (persistence, messaging, external APIs, clock,
  UUID generation) MUST be interfaces defined inside the core
  (e.g., `ports.out.*Repository`, `*Client`, `*Publisher`). Implementations
  MUST live ONLY in `adapters` (e.g., `adapters.persistence.*`,
  `adapters.messaging.*`, `adapters.rest.*`).
- `domain` services MUST accept driven ports via constructors and MUST NEVER
  instantiate adapter classes or use static infrastructure lookups.

Rationale: core-owned ports invert dependencies so infrastructure is
pluggable; swapping a database or client requires only a new adapter, never
a core change.

### IV. Dependency Injection & Wiring

Constructor injection is REQUIRED for all Spring components. Field injection
is FORBIDDEN.

Rules:
- All Spring beans MUST declare dependencies via constructors (explicit or
  single-constructor implicit injection). `@Autowired` on fields and setter
  injection as a substitute for mandatory dependencies are FORBIDDEN.
- Domain services MUST NOT carry Spring stereotypes; they MUST be plain
  classes instantiated via a central `@Configuration` class or custom
  factory beans in the configuration/adapter layer that wires ports to
  implementations.
- Configuration classes MUST be the single place where adapter
  implementations are bound to core port interfaces for each context.

Rationale: constructor injection enforces immutability and explicit
dependencies, while configuration-centralized wiring keeps the core free of
framework metadata and makes object graphs auditable.

### V. Contract-First REST Driving Ports (OpenAPI)

Every driving port exposed over HTTP MUST be described by a versioned
OpenAPI 3.1 document committed in the owning adapter module at
`src/main/resources/openapi/<name>.openapi.yaml`.

Rules:
- Contract first: the contract MUST be authored and reviewed before
  implementation; controllers MUST implement the generated server
  interface, or equivalent contract tests MUST prove conformance.
- Generated sources are build outputs; they MUST NOT be hand-edited and
  MUST NOT be committed.
- `domain` and `ports` MUST remain free of HTTP and OpenAPI types; contract
  DTOs MUST map to commands/queries at the adapter edge.
- Changes within a contract version MUST be additive; breaking changes MUST
  bump the contract version and the affected path.
- CI MUST validate every contract and MUST fail on drift between the
  contract and the implementation.
- Contract strictness and output exposure are governed by Principle XV:
  request schemas MUST be strict with bounded fields, and responses MUST be
  serialised from an allowlisted response DTO.

Rationale: the contract is the published interface, so making it the source
of truth keeps consumers stable and the core HTTP-agnostic.

### VI. Kafka + AsyncAPI for Domain Events & Commands

All asynchronous integration - domain events and commands alike - MUST use
Kafka, described by an AsyncAPI 3.x document committed in the Kafka adapter
module at `src/main/resources/asyncapi/asyncapi.yaml`.

Rules:
- Driven ports stay core-owned and framework-free (`ports.out.*Publisher`,
  `ports.out.*Consumer`) with plain-Java event and command types; Kafka
  types MUST NEVER appear in `domain` or `ports`.
- Topic names MUST follow `<domain>.<entity>.<event|command>.v<major>`;
  topic, message key, and schema version MUST be declared in the AsyncAPI
  document.
- Schemas MUST evolve additively within a topic major version; breaking
  changes MUST publish a new major topic version.
- Consumers MUST be idempotent and MUST process by message key; messages
  MUST carry an event or command id and an occurred-at timestamp.
- Publication MUST NOT occur before the local transaction commits; where
  emission must be atomic with a state change, a transactional outbox MUST be
  used (Principle X).
- Every topic MUST declare in the AsyncAPI document its message key, its
  ordering requirement, and its competing-consumer policy. Messages whose order
  matters MUST be keyed by aggregate identity so that they travel one shard and
  are consumed in order.
- Every consumer MUST record processed message ids and suppress duplicates in
  the same local transaction as the effect, so a redelivery cannot apply twice.
- The outbox relay mechanism MUST be named in the plan (polling publisher or
  transaction-log tailing) and MUST be resumable after a restart without losing
  or reordering committed messages.
- Relay lag MUST be exposed as a metric and MUST alert; a broker outage MUST NOT
  lose committed state, and messages that could not be published MUST remain
  republishable from the outbox (Principle XI).

Rationale: a documented, versioned asynchronous contract keeps publishers
and consumers independently deployable and the core transport-agnostic.

### VII. External Web Services via Simple Proxy Web Services

External and third-party web services MUST be reached only through a simple
proxy web service: an internal service deployed separately from the
application. The application MUST NOT call vendor endpoints directly.

Rules:
- Each external system MUST have its own proxy web service exposing an
  OpenAPI 3.1 contract; Principle V applies to proxies as well.
- Proxies MUST be thin pass-throughs: request/response translation,
  timeouts, retries, and error mapping only. Business rules and domain
  dependencies MUST NOT exist in a proxy.
- Vendor endpoints, credentials, and vendor-specific payloads MUST exist
  ONLY inside the proxy service; vendor SDKs MUST NOT appear in any
  application module.
- The application MUST integrate through a `<system>-proxy-adapter` driven
  adapter that implements the core-owned port and calls the proxy over HTTP
  with a generated OpenAPI client, mapping proxy failures to core errors.
- Application modules MUST NOT depend on proxy modules at build time, and
  proxies MUST NOT depend on `domain`, `ports`, or `application-core`.
- Outbound destinations MUST come from a static allowlist of hosts and
  schemes; fetching a user-supplied or caller-supplied URL is FORBIDDEN (SSRF).
- Vendor responses MUST be treated as untrusted: they MUST be validated
  against the proxy contract before use or persistence, and a malformed or
  unavailable vendor payload MUST map to a core error with no partial write.
- Proxy credentials and vendor secrets MUST be injected from secret storage and
  MUST have a documented rotation procedure (Principle XVII).

Rationale: proxies absorb vendor churn and secrets, keeping core contracts
stable and driven adapters stubbable in tests.

### VIII. Architecture Scope: Hexagonal for Core Domains Only (NON-NEGOTIABLE)

Every service MUST be classified as either a CORE-domain service or a
NON-CORE-domain service, and that classification decides whether the
hexagonal rules apply.

Rules:
- Core-domain services MUST follow Principles I, II, and III and MUST use
  the Appendix A module layout.
- Non-core-domain services MUST NOT adopt hexagonal layering: no
  `domain` / `ports` / `adapters` split, no core-owned port interfaces, no
  hexagonal ArchUnit gates, and no copy of the Appendix A layout. They MUST
  use a conventional Spring Boot structure (layered or package-by-feature)
  and MUST NOT be described as hexagonal.
- Classification criteria: a domain is CORE when it holds differentiating
  business rules and a long-lived domain model; it is NON-CORE when it is
  supporting or generic (admin CRUD, reporting, notifications, integration
  plumbing).
- Unresolved classification MUST default to CORE.
- Each service MUST record its classification in its own architecture
  documentation and state it in the Constitution Check of its plan.
- Reclassifying a domain REQUIRES a constitution amendment, because it
  changes which principles apply.
- Everything else is universal: Principle IV (constructor injection),
  Principle V (OpenAPI contracts), Principle VI (Kafka + AsyncAPI),
  Principle VII (proxy web services), the Technology Constraints, the
  Development Workflow, and Governance apply to ALL services, core and
  non-core alike.

Rationale: ports and adapters earn their keep where business rules are the
product; imposing them on generic services adds ceremony and slows
delivery, while the contract, wiring, and platform rules that protect
operations stay universal.

### IX. Service Boundaries & Ownership (NON-NEGOTIABLE)

Services MUST be decomposed by business capability or by DDD subdomain. A
service MUST NOT be defined by a technical layer, and each service MUST have
exactly one owning team able to build, test, and deploy it independently.

Rules:
- Services MUST be self-contained: serving a synchronous request MUST NOT
  require waiting synchronously on another service. Work that depends on a
  remote system MUST be arranged asynchronously through the event/command path
  (Principle VI).
- Cross-service synchronous calls MUST be minimised; where unavoidable the
  caller MUST apply Principle XI (timeout, bounded retry, circuit breaker).
- Data owned by another service MUST NOT be reached directly; see Principle X.
- New services MUST start from the service template: Appendix A for
  core-domain services, the conventional Spring Boot layout for non-core
  services, with all configuration externalized per environment.
- A boundary MUST be introduced or changed only with the decision recorded in
  the plan: the capability or subdomain, the owning team, and the data owned.
- Greenfield services MUST be built whole. The strangler-fig approach applies
  only where a monolith is being replaced: new features MUST be implemented as
  services first, and extraction MUST be prioritised by change frequency,
  coupling, and business value, with the order recorded in the plan.
- Where a service is extracted from a monolith, the integration glue between the
  monolith and the new service (API, event, or database view) MUST be chosen
  deliberately and recorded; the monolith MUST keep working throughout, and each
  extraction MUST be reversible until caller migration completes.

Rationale: boundaries drawn on the business rather than on layers keep teams
autonomous and coupling low; self-containment keeps user-facing latency
independent of unrelated services.

### X. Data Ownership & Cross-Service Consistency (NON-NEGOTIABLE)

Each service MUST own its data privately - its own PostgreSQL schema, or a
dedicated database instance where isolation demands it. Reading or writing
another service's tables or schema is FORBIDDEN.

Rules:
- Data MUST be shared between services only through API contracts
  (Principle V) or events and commands (Principle VI).
- Distributed transactions and two-phase commit are FORBIDDEN. Consistency
  spanning services MUST use a saga: a sequence of local transactions with
  compensating actions. The coordination style (choreography or orchestration)
  MUST be recorded in the plan.
- A state change and the emission of its event or command MUST be atomic;
  therefore the transactional outbox pattern MUST be used (Principle VI).
- Consumers MUST be idempotent and MUST detect duplicates, because delivery is
  at-least-once.
- Cross-service queries MUST use API composition. CQRS with materialized views
  or a command-side replica MAY be used only when composition is measurably
  inadequate, with the justification recorded.
- Schema changes MUST be backward compatible with the released contract
  version (Principle XII).
- SENSITIVE data (Principle XIII) MUST NOT be copied into events, commands, or
  inter-service payloads; those MUST carry identifiers, and every fact MUST
  have exactly one authoritative owning service.

Rationale: private data removes the cheapest and most damaging form of
coupling, while sagas plus an outbox give eventual consistency without
distributed locking.

### XI. Production Readiness: Resilience & Observability (NON-NEGOTIABLE)

Every service MUST be production ready: it MUST degrade under failure rather
than cascade, and its behaviour MUST be observable in production.

Resilience rules:
- Every remote call MUST set an explicit timeout; unbounded waits are
  FORBIDDEN.
- Retries MUST be bounded, MUST use backoff with jitter, and MUST apply only
  to idempotent operations.
- A circuit breaker MUST guard every synchronous dependency on another service
  or on a proxy web service (Principle VII).
- Bulkheads MUST isolate scarce resources such as thread pools and database
  connections, so one slow dependency cannot exhaust the service.
- Rate limiting MUST protect public and unauthenticated endpoints, including
  the business registration intake.
- Failed or unprocessable messages MUST be routed to a dead-letter channel with
  alerting; infinite retry loops are FORBIDDEN.
- Abuse controls on public and unauthenticated endpoints MUST apply velocity
  limits per caller AND per business key (for example a business identifier
  supplied by the caller), MUST return 429 with `Retry-After`, and MUST raise
  an alert when a limiter trips.
- Capacity MUST be verified by tests, not assumed: the service MUST assert its
  headroom and MUST test legitimate domain rules that an attacker could exploit
  for load amplification (repeated replacement, retry, or resubmission paths).

Observability rules:
- A correlation/trace identifier MUST be created at the entry point and
  propagated across every REST call and Kafka message, and MUST appear in
  structured logs for all hops.
- Every service MUST expose health and readiness endpoints covering its
  dependencies (database, broker, proxy services).
- Metrics MUST cover at least latency, error rate and throughput plus the
  domain signals of the service (for example applications submitted and
  verification outcomes).
- Every state-changing operation MUST be audit logged with actor, timestamp,
  identifiers and outcome; log aggregation, distributed tracing, exception
  tracking, and a record of deployments and configuration changes MUST be in
  place before production release.
- Logging MUST go through a core-owned domain-oriented logger port: logging
  domain objects through `toString()` or reflection is FORBIDDEN, and unchecked
  input values MUST NOT be logged.
- Log records MUST be structured JSON shipped to a central logging service; a
  local file appender as the system of record is FORBIDDEN.
- Records MUST be categorised as audit, behaviour, or error, and the categories
  MUST NOT be intermixed in one stream: audit records MUST be append-only,
  write-restricted and retained, and MUST carry service name, version, instance
  id, and the correlation identifier.
- Every service MUST maintain an explicit redaction list covering credentials,
  tokens, sensitive fields, and full request payloads, and MUST prove redaction
  in tests (Principle XIII).

Rationale: fast flow depends on observability and on failures staying local;
without these, team autonomy becomes unmanageable production risk.

### XII. Independent Deployability & Evolution (NON-NEGOTIABLE)

A service MUST be deployable and rollback-able on its own, without changing or
coordinating with other services.

Rules:
- Each service MUST build one immutable artefact, produced once and promoted
  unchanged through environments, with one service instance per container.
- There MUST be no shared deployable unit and no release train that requires
  several services to ship together.
- API and event contracts MUST evolve additively within a version; breaking
  changes MUST create a new version (Principle V for HTTP, Principle VI for
  topics) and MUST keep the previous version supported until consumers have
  migrated.
- Database schema changes MUST stay backward compatible with the currently
  deployed service version (expand, migrate, then contract).
- External clients MUST reach services through an API gateway, and a dedicated
  backend-for-frontend MUST be used where client needs diverge materially
  instead of overloading one API. This becomes mandatory as soon as more than
  one service is externally exposed.
- The gateway MUST own edge concerns only - route inventory, request and
  response schema validation, secure headers, and rate limiting - while
  business authorization stays inside the services (Principle XVI).
- CI MUST enforce the deployment-pipeline gates in Development Workflow before
  an artefact can be promoted.
- The deployment platform MUST be recorded in the plan and MUST give each
  service isolated instances with their own database instance; a shared
  deployment unit or shared database instance across services is FORBIDDEN
  (Principle X).
- The service-discovery mechanism MUST be recorded in the plan. The platform's
  own registry and DNS resolution MUST be preferred; self-registration or
  third-party registration MUST be justified, tested against a registry
  restart, and MUST NOT become a routing single point of failure.

Rationale: independent deployability is the property that justifies the
architecture's cost, and contract plus schema discipline is what preserves it.

### XIII. Sensitive Data Protection (NON-NEGOTIABLE)

Every persisted, logged, or published field MUST carry an explicit sensitivity
classification, and data classified SENSITIVE MUST be protected wherever it
travels.

Rules:
- Every field MUST be classified PUBLIC, INTERNAL, CONFIDENTIAL, or SENSITIVE
  in the plan. Fields that are individually harmless but jointly identify or
  locate a person MUST be classified SENSITIVE as a combination.
- SENSITIVE values MUST NEVER appear in logs, traces, metric labels, error
  responses, dead-letter messages, or events. They MUST travel in a read-once
  domain primitive that permits a single release at the one place that needs
  it: the encryption adapter, or the masked projection returned to a caller.
- A read-once primitive MUST release its value exactly once and MUST throw on
  re-release. It MUST be final, MUST NOT be serialisable, MUST mark sensitive
  fields transient, MUST mask its value in `toString()`, and MUST hand out
  defensive copies for any mutable carrier such as `char[]`. Where a value is
  legitimately read more than once, the exception MUST be recorded in the plan
  with its justification.
- SENSITIVE data MUST be encrypted at rest, and the key version used MUST be
  stored with the ciphertext so keys can be rotated without rewriting history.
- Responses MUST expose the minimum projection the consumer needs; a government
  identity number or equivalent identifier MUST be returned masked only, never
  in full.
- Every SENSITIVE field MUST have a retention and deletion rule recorded in the
  plan, and replacement or deletion semantics MUST NOT silently destroy audit
  evidence (Principle XI).

Rationale: classification is the decision that determines what can leak, so it
is made at design time and enforced where the value is created, stored, and
emitted - not audited after the fact.

### XIV. Untrusted Input & Valid-by-Construction Domain Model (NON-NEGOTIABLE)

All data crossing a trust boundary MUST be treated as untrusted until validated,
and the domain model MUST make invalid state unrepresentable.

Rules:
- In `domain`, a concept MUST NOT be represented by a language primitive or
  generic type (`String`, `int`, `long`, `boolean`, `UUID`, `List<String>`) in
  constructors, entity fields, or port signatures. Each concept MUST be a
  dedicated domain primitive - an immutable `record` or `final` class - that is
  valid by construction: invariants are checked at creation and invalid input is
  rejected, so an existing instance is always valid.
- Validation MUST run at the boundary in this order: origin (expected source and
  path), size (declared maxima enforced before parsing, allocation, or regex
  matching), lexical content (allowed characters and encoding), syntax (format),
  then semantics (meaningful and permitted in this context). Input MUST be
  validated before any normalisation; silently repairing input so it satisfies a
  contract is FORBIDDEN.
- Entities and handlers MUST NOT re-validate what a domain primitive already
  guarantees; they MUST trust domain primitives and add only semantic rules
  about their own state.
- Domain types MUST be immutable: fields `final`, public setters FORBIDDEN,
  mutable objects MUST NOT be handed out, and collections MUST be exposed
  unmodifiable with immutable elements.
- Domain entities MUST be consistent on creation: no public no-arg constructor,
  all mandatory fields supplied at construction, and conditional or advanced
  constraints upheld by construction.
- State changes MUST go through explicit named transition operations, or an
  explicit state object, that re-establish the invariants; an unpermitted
  transition MUST be rejected rather than merely documented.
- Every domain rule MUST have normal, boundary, and invalid-input tests; an
  invariant without a failing-case test is not verified.
- Domain primitives MUST NOT be published across a boundary (Principle I): HTTP,
  Kafka, and proxy contracts carry DTOs mapped at the adapter edge.
- A domain primitive MUST define value-based `equals` and `hashCode` over one
  canonical representation, and two instances built from equal input MUST be
  interchangeable.
- Construction MUST be the only place a domain primitive can fail: an existing
  instance MUST NOT throw on read or on a legal transition, and MUST NOT be
  mutable through any reference it hands out.
- `null` MUST NOT represent absence; absence MUST be an explicit domain concept
  (for example an `Empty` value or an optional domain primitive).
- Lexical validation MUST state the canonical form it accepts, and text MUST be
  normalised to that form before comparison; confusable or homoglyph input MUST
  be rejected rather than silently folded.
- Data access MUST be parameterised: SQL, JPQL, and any query text built by
  interpolating caller input are FORBIDDEN, and dynamic query fragments MUST come
  from allow-listed code constants.

Rationale: validating once, in one place, at the boundary removes defensive
re-checks from the whole codebase and leaves every reachable state a legal one.

### XV. Secure HTTP Contracts & Response Minimisation

HTTP contracts MUST be strict about input and minimal about output.

Rules:
- Every request schema MUST set `additionalProperties: false` and MUST mark all
  mandatory fields as `required`; every request field MUST be bounded with
  `maxLength`, `maximum`/`minimum`, `maxItems`, `enum`, or `pattern`. Unbounded
  strings, numbers, or lists MUST NOT appear in any contract.
- Request and response bodies MUST use separate schemas. Server-managed
  properties - identifier, status, verification outcome, timestamps, audit
  fields - MUST NOT appear in any request schema and MUST be rejected when
  supplied; this is the mass-assignment guard.
- Responses MUST be serialised from an explicit response DTO that is the field
  allowlist; serialising JPA entities, domain objects, `Map`/`Object`, or vendor
  payloads is FORBIDDEN.
- Externally visible identifiers MUST be non-enumerable, server-generated
  values with at least 122 bits of randomness (UUIDv4 or equivalent). Sequential
  and auto-increment keys MUST NOT be exposed, and client-supplied identifiers
  MUST be rejected on create.
- Every operation MUST declare its security requirement explicitly. An operation
  that is deliberately public MUST declare an empty requirement and MUST be
  justified in the plan, and the reserved future authentication scheme MUST be
  declared in the contract from the start so that adding authentication is
  additive rather than a rewrite.
- Errors MUST use one shared, documented error schema: no stack traces, no SQL,
  no internal identifiers, and the offending submitted value MUST NOT be echoed
  back to the caller.
- Every exposed endpoint MUST be registered against a versioned contract in the
  API inventory; an undocumented or shadow route MUST be treated as an incident.
- Collection responses MUST be page-bounded with a declared maximum page size; a
  response whose size grows without a bound MUST NOT be exposed.
- Every value rendered into HTML, JavaScript, a URL, or an operator-facing view
  MUST be output-encoded for that context; input validation MUST NOT be treated
  as a substitute for output encoding.
- The policy for absent versus forbidden resources (404 versus 403) MUST be
  recorded per resource type and MUST NOT disclose the existence of a resource
  the caller is not permitted to access.

Rationale: a strict contract is the only point that can reject unknown input and
refuse to over-share output before any domain code runs, so exposure becomes a
contract property rather than a code-review judgement.

### XVI. Identity-Ready, Default-Deny Authorization (NON-NEGOTIABLE)

Every operation is denied unless the plan states who may call it, and services
MUST be able to adopt authentication without redesign.

Rules:
- Caller identity MUST enter the core through a core-owned port (for example
  `CallerIdentity` / `ActorContext`) even while a service has no authentication:
  an explicit anonymous actor MUST be resolved and recorded rather than omitted,
  and persisted records MUST carry the actor reference.
- Every operation MUST appear in an access-control matrix in the plan naming the
  permitted actor class, including the degenerate no-authentication case. An
  operation with no matrix row MUST fail review.
- Access control MUST be object-level, not operation-level: every access to a
  resource instance MUST verify that the actor is permitted on THAT instance,
  and each matrix row MUST record the ownership or scoping predicate that
  decides it. An operation-level check alone is a defect.
- Authorization MUST be enforced inside the core or its ports for every
  operation; the gateway MUST NOT be the only enforcement point. Internal
  service-to-service endpoints MUST receive the same validation as public ones;
  no endpoint is exempt because of network location.
- Token and credential handling MUST NOT be hand-rolled: permitted algorithms,
  issuer, audience, and expiry MUST be pinned, signing keys MUST come from the
  trusted discovery or JWKS endpoint with a bounded cache that refetches on an
  unknown key id, and key rotation MUST NOT require a release.
- Destructive, irreversible, or state-overwriting flows MUST NOT be reachable
  without authentication; where a v1 scope requires one to be, it MUST be
  velocity-limited (Principle XI), tested as an abuse scenario, and alertable.
- Administrative, manual, and back-office actions MUST be first-class,
  authenticated, audit-logged operations of the service; using standing direct
  database access to change state is FORBIDDEN.
- Token verification MUST use an algorithm allow-list that rejects `alg=none`
  case-insensitively, MUST bind issuer, audience, and expiry with a bounded clock
  skew, and SHOULD accept asymmetric algorithms only.
- Tokens SHOULD be issued and rotated by a managed provider rather than by this
  platform; issuing tokens ourselves REQUIRES the decision recorded in the plan.

Rationale: retrofitting identity is most expensive exactly where it matters
most, so identity is threaded through the design from v1 even when it is
anonymous.

### XVII. Secrets, Keys & Secure Baseline Configuration

Secrets MUST live outside the artefact, configuration MUST be validated before
use, and the secure default MUST be the only default.

Rules:
- Secrets, credentials, and encryption keys MUST NEVER appear in code, committed
  configuration, contracts, images, or logs; they MUST be injected from the
  environment or from a secret store.
- CI MUST run a secret scan and a dependency vulnerability scan. A committed
  secret MUST be treated as an incident that requires rotation, not merely
  removal.
- Configuration MUST be validated at startup; every default MUST be known and
  asserted by a test; invalid configuration MUST prevent startup rather than
  fail open at first use.
- Every secret and key MUST have a documented rotation procedure, and rotation
  MUST NOT require a coordinated release across services.
- Only the required HTTP methods MUST be enabled; management, actuator, and
  debug endpoints MUST NOT be publicly reachable; error responses MUST be
  generic; and security headers MUST be set at the edge.
- Feature toggles MUST be owned, time-boxed, and audited, and MUST NOT
  substitute for a release strategy.
- CORS MUST use an explicit origin allow-list; a wildcard origin combined with
  credentials is FORBIDDEN, and the policy MUST be asserted by test.
- Security response headers (at least `X-Content-Type-Options: nosniff`, a
  content security policy, and `frame-ancestors` where the API is browsable)
  MUST be set centrally in middleware and asserted by a deployment smoke test;
  responses carrying SENSITIVE data MUST set `Cache-Control: no-store`.
- Tokens, secrets, and session identifiers MUST NOT travel in URLs, query
  strings, or referrers.

Rationale: configuration and secrets are the shortest path from a small mistake
to a breach, so they are externalized, validated, rotatable, and default-deny.

### XVIII. Saga Design & Compensation (NON-NEGOTIABLE)

Every cross-service consistency flow MUST be designed as a saga whose steps and
compensations are explicit, classified, and testable.

Rules:
- Every saga step MUST be classified in the plan as compensatable, pivot, or
  retryable, and the classification MUST be visible in the code that implements
  the step.
- A saga MUST contain at most one pivot step; every step after the pivot MUST be
  retryable, so the flow can always be completed forward.
- Compensating actions MUST be idempotent, MUST tolerate replay and out-of-order
  delivery, and MUST NOT fail the saga when the step they compensate never
  committed.
- Saga state MUST be persisted after every step and the coordinator MUST be
  replay-safe; an in-memory coordinator is FORBIDDEN.
- Because sagas are not isolated, every anomaly the flow can produce MUST have a
  named countermeasure in the plan: semantic lock, commutative update,
  pessimistic view, or reread value.
- Coordination style (choreography or orchestration) MUST be justified against
  complexity, coupling, and observability criteria recorded in the plan, not
  merely noted.
- Every saga MUST define its terminal state for exhausted retries, and that
  terminal state MUST be an audited, operator-visible outcome (Principle XI).

Rationale: a saga is the only permitted cross-service consistency mechanism
(Principle X), so its rollback path carries the same weight as its happy path and
must be designed, classified, and tested rather than improvised.

### XIX. Aggregate & Transaction Boundaries (NON-NEGOTIABLE)

The aggregate is the unit of consistency: it defines what one local transaction
may change and what MUST be arranged as a saga.

Rules:
- One local transaction MUST create or update exactly one aggregate; changing two
  aggregates atomically is FORBIDDEN and MUST be arranged as a saga
  (Principle XVIII) or through an eventual consistency flow.
- Aggregates MUST reference other aggregates by identity only; object references
  across aggregate boundaries are FORBIDDEN, and only the aggregate root MAY be
  referenced or mutated from outside its aggregate.
- Consistency between aggregates of one service MUST be eventual, driven by
  domain events derived from aggregate state; an event MUST be derivable from one
  aggregate and MUST be returned to the caller or published by an adapter, never
  emitted from inside a domain type (Principle VI).
- Each aggregate root MUST have exactly one repository port, and no repository or
  adapter MAY span two aggregate roots.
- Aggregate boundaries and the invariant each one protects MUST be recorded in
  the plan; an aggregate with no stated invariant is a violation.
- Event sourcing MAY be used as the persistence model for an aggregate only when
  the plan records the justification, the snapshot and replay strategy,
  optimistic-concurrency handling, and the stored-event versioning policy;
  without that record it is FORBIDDEN.
- Stored and published domain events MUST evolve additively, and every reader
  MUST tolerate older event versions.

Rationale: aggregate boundaries decide whether a change is a local transaction
or a saga, so they are a design decision rather than an implementation detail,
and they are what makes the consistency rules of Principle X enforceable.

### XX. Transport Security & Workload Identity (NON-NEGOTIABLE)

Every network hop MUST be encrypted, and the identity of the workload making a
call MUST be verifiable independently of the network it sits on.

Rules:
- All traffic - client to gateway, service to service, and proxy egress - MUST use
  TLS; plaintext HTTP is FORBIDDEN in every deployed environment.
- TLS 1.2 MUST be the minimum accepted version; TLS 1.0 and 1.1 and any protocol
  or cipher outside an explicit allow-list MUST be disabled and asserted by test.
- Externally reachable hosts MUST send HSTS with a minimum one-year max-age and
  MUST redirect plain HTTP to HTTPS.
- Service-to-service calls SHOULD use mutual TLS with short-lived, rotatable
  certificates. Terminating TLS at the gateway MUST NOT discard the identity of
  the originating workload: the callee MUST still validate it (Principle XVI).
- Certificate, key, and trust-store rotation MUST NOT require a coordinated
  release (Principle XVII).
- The deployed environment MUST be asserted by a test or smoke check that refuses
  connections below the minimum TLS version and reports the negotiated version.

Rationale: without transport identity, "internal" is an assumption rather than a
control, and every in-service authorization decision (Principle XVI) rests on an
unverified caller.

### XXI. API Lifecycle, Deprecation & Change Communication

A published API version has a lifecycle, and retiring one is a scheduled,
communicated act rather than a deletion.

Rules:
- Every exposed version MUST carry a lifecycle state (active, deprecated, or
  retired) in the API inventory required by Principle XV.
- Deprecation MUST be announced with `Deprecation` and `Sunset` response headers
  plus a `Link` to the migration note, over a minimum notice period recorded in
  the plan.
- A retired version MUST be de-provisioned at the gateway so it cannot be
  re-exposed accidentally, and a retired version label MUST NOT be reused for a
  new contract.
- Breaking changes MUST follow Principle V for HTTP and Principle VI for topics,
  and the number of simultaneously supported versions MUST be bounded and
  recorded in the plan.
- Consumers of a deprecated version MUST be identifiable from telemetry before
  retirement is scheduled.

Rationale: additive evolution (Principle XII) preserves consumers only if the end
of a version's life is declared, communicated, and enforced instead of drifting.

### XXII. Security Governance: Risk Acceptance, Disclosure & Incident Response

Security decisions that are not implemented MUST be recorded as owned,
time-boxed risk, and security failures MUST have a defined path from report to
fix.

Rules:
- Accepting a security risk without mitigation MUST record the owner, the
  rationale, the compensating controls, and an expiry date. An acceptance with no
  expiry is invalid, and an expired acceptance MUST be re-reviewed and either
  renewed or fixed.
- Downgrading a sensitivity classification (Principle XIII) MUST be recorded in
  the plan with an owner and a rationale, and MUST be re-reviewed at the next
  compliance review.
- An externally reachable channel for vulnerability reports MUST exist and be
  published; reports MUST be triaged against a stated window and tracked to
  closure.
- A confirmed exposure of SENSITIVE data or credentials MUST be handled as an
  incident: contained, rotated (Principle XVII), assessed against notification
  duties, and followed by corrective actions that trace to a test, a gate, or a
  constitution amendment.
- Compliance review covers Principles I-XXII, and every boundary, classification,
  and sensitivity decision MUST have a dated architecture decision record (ADR)
  in the repository rather than existing only inside a transient plan.

Rationale: unrecorded risk acceptance and undocumented decisions are how a
security posture erodes silently, and an unowned disclosure channel turns a
small defect into an unmanaged incident.

## Technology Constraints & Build Standards

The stack is Java 25 with Spring Boot 4 for adapters and configuration
only. The build tool is Gradle with Groovy DSL (`build.gradle`); Java,
Spring Boot, and dependency versions MUST be locked in the build files and
verified in CI. The pinned integration stack is:

- **Persistence**: PostgreSQL with Spring Data JPA/Hibernate, used ONLY in
  `postgres-adapter`.
- **HTTP driving interfaces**: REST described by OpenAPI 3.1 contracts;
  springdoc and the openapi-generator Gradle tasks produce server
  interfaces and clients.
- **Asynchronous integration**: Apache Kafka (Spring for Apache Kafka) as
  the only asynchronous transport, documented with AsyncAPI 3.x.
- **External systems**: internal simple proxy web services (Principle VII)
  consumed through generated OpenAPI clients.
- **Resilience**: Resilience4j for timeouts, bounded retries, circuit breakers
  and bulkheads (Principle XI).
- **Edge**: Spring Cloud Gateway as the API gateway (Principle XII).
- **Observability**: OpenTelemetry for distributed tracing, alongside
  structured logs and metrics (Principle XI).
- **Configuration**: externalized per environment; environment-specific values
  MUST NOT be baked into build artefacts.
- **Security tooling** (decision needed): contract linting with an OWASP API
  rule set, contract fuzzing, secret scanning, and dependency vulnerability
  scanning MUST run in the pipeline; the specific tools MUST be pinned in the
  build files before the first security gate is required.
- **Cryptography and identity** (decision needed): encryption of SENSITIVE
  data MUST use a managed key store with versioned keys (Principle XIII), and
  the future authentication stack MUST be an OIDC provider with JWT
  verification (Principle XVI); libraries MUST be pinned once chosen.
- **Deployment platform and service discovery** (decision needed): the platform
  (one container instance per service on Kubernetes, VM, or serverless) and the
  discovery mechanism (platform registry and DNS, self-registration, or
  third-party registration) MUST be recorded in the plan before the first
  inter-service call (Principle XII).
- **Test tooling** (decision needed): executable acceptance specifications and
  contract-test tooling for both the consumer and the provider side MUST be
  chosen and pinned before gate 23 is required.
- **Transport security** (decision needed): TLS and mTLS configuration, HSTS, and
  the central header and CORS middleware MUST be pinned (Principle XX) before any
  environment is exposed beyond local development.
- **Event store and saga support** (decision needed): where an aggregate is
  event-sourced (Principle XIX), the event store, snapshot policy, and
  optimistic-concurrency implementation MUST be pinned in the plan that justifies
  it.
- **Decision records** (decision needed): the ADR location and format MUST be
  chosen so that boundary, classification, and sensitivity decisions have a
  durable home (Principle XXII).

Rules:
- Persistence, web (Spring MVC), and messaging clients MUST reside ONLY in
  `adapters`; core MUST interact with them solely through driven-port
  interfaces.
- DTOs, JPA entities, Kafka message types, and other framework-specific
  models MUST be mapped at the adapter edge; shared cross-layer models
  carrying framework annotations are FORBIDDEN.
- Contract documents (`openapi/`, `asyncapi/`) MUST be committed with the
  owning adapter module and validated in CI.
- Each service MUST own a private PostgreSQL schema, or a dedicated database
  instance where isolation demands it; cross-schema access is FORBIDDEN
  (Principle X).
- SENSITIVE fields MUST be encrypted and decrypted by an adapter that owns the
  cipher and its keys; core MUST handle them only as domain primitives, and
  unmasked values MUST NOT be returned across a port boundary
  (Principles XIII and XIV).

## Development Workflow & Quality Gates

Every change MUST verify compliance with the principles that apply to the
service (Principle VIII) before merge:

- In core-domain services, new business logic MUST be added to `domain` with
  core-owned ports; new I/O MUST be added as an adapter behind a driven port
  interface.
- Core-domain services MUST also satisfy: (1) ArchUnit/boundary tests green,
  (2) plain-JUnit core tests without Spring context, (3) adapter tests for
  each new implementation, (4) a reviewer checklist confirming no Spring/JPA
  imports in `domain`/`ports`.
- All services MUST satisfy: (5) constructor injection only, (6) OpenAPI and
  AsyncAPI contract validation with no implementation drift, (7) Kafka schema
  compatibility check for any broker change, (8) the service's core/non-core
  classification recorded (Principle VIII), (9) service component tests that
  exercise the service in isolation using doubles for invoked services,
  (10) consumer-driven contract tests for every HTTP and Kafka consumer, run in
  CI, (11) the resilience, observability and deployment-pipeline requirements
  of Principles XI and XII.
- Security gates apply to every service: (12) a threat model recorded in the
  plan before tasks are generated, covering decomposed flows, ranked threats,
  and their mitigations; (13) every threat-model scenario traced to at least one
  automated test before release; (14) security-aware contract linting (strict
  schemas, bounded fields, declared security requirements) plus contract
  fuzzing in CI; (15) a negative-input suite proving that unknown properties,
  oversize bodies, malformed payloads, and control characters return documented
  4xx responses and never 5xx; (16) access-control regression tests over the
  matrix of Principle XVI, plus a redaction test proving that no SENSITIVE value
  reaches logs, traces, or error bodies; (17) secret and dependency
  vulnerability scans; (18) a security stage that blocks merge while red.
- Test taxonomy: every test MUST be classified as unit, integration, component,
  contract, or end-to-end, and the classification MUST match what the test
  actually verifies. Unit tests cover domain logic with no framework and no I/O;
  integration tests cover an adapter against its real dependency; component tests
  exercise one service in isolation with doubles for every service it calls;
  contract tests verify the published interface from BOTH the consumer and the
  provider side; end-to-end tests cover only critical journeys. A coverage
  percentage MUST NOT be used as a substitute for these categories.
- Deployment pipeline: the pipeline stages (pre-commit, commit, acceptance, and
  production release) MUST be defined in the plan, and an artefact MUST be built
  once and promoted unchanged through every stage (Principle XII).
- Principles XVIII-XXII add these gates: (19) saga steps classified with at most
  one pivot, and compensation idempotency proven under replay and out-of-order
  delivery; (20) a named countermeasure per isolation anomaly, with a test that
  reproduces the anomaly; (21) ArchUnit (or equivalent) rules proving one
  aggregate root per transaction, identity-only cross-aggregate references, and
  every domain event traceable to one aggregate; (22) ordering preserved and
  duplicates suppressed for keyed topics, a relay test that survives a broker
  outage, and a relay-lag alert; (23) the test taxonomy above recorded per test,
  including provider-side contract verification; (24) pipeline stages defined
  with unchanged artefact promotion; (25) a transport-security assertion that
  refuses connections below the minimum TLS version and reports the negotiated
  version; (26) an object-level authorization regression test per resource type,
  plus an existence-non-disclosure assertion (Principle XVI); (27) deprecation
  and `Sunset` headers asserted and retired versions verified as de-provisioned;
  (28) ADR presence for boundary, classification, and sensitivity decisions, and
  no expired risk acceptance left unreviewed.
- End-to-end tests MUST be limited to a small set of critical user journeys and
  MUST NOT be the primary verification mechanism.
- Violations of Principles I-XXII MUST block merge; justified exceptions
  require a constitution amendment, not an ad-hoc waiver.

## Appendix A: Normative Directory Structure Template

### Core Modules (shared-kernel, application-core)

This appendix is NORMATIVE and applies to CORE-domain services only;
non-core-domain services MUST NOT copy this layout (Principle VIII). New
projects and features MUST follow this Gradle (Groovy DSL) multi-module
layout, adapted from the reference
architecture https://github.com/emedina/hexagonal-spring-ref-app.git
(Java 25, Spring Boot 4.0.1; reference uses Maven — mapped here to Gradle
Groovy per constitution stack pin, with a `postgres-adapter` added as the
REQUIRED production persistence adapter, plus `kafka-adapter`,
`<system>-proxy-adapter` and the separately deployed `proxy-gateway` for
Principles VI and VII). `<pkg>` is the base package
(e.g., `com.example.<app>`).

```text
settings.gradle                        # includes all modules below
build.gradle                           # root: Java 25 toolchain, Spring Boot 4 BOM, versions locked
gradle.properties

shared-kernel/
├── build.gradle
└── src/
    ├── main/java/<pkg>/shared/
    │   ├── dto/                       # cross-layer DTOs (no framework annotations)
    │   │   ├── ArticleDTO.java
    │   │   └── AuthorDTO.java
    │   ├── error/
    │   │   └── Error.java             # functional error type (e.g., Either<Error, T>)
    │   └── validation/
    │       ├── ValidationError.java
    │       └── Validations.java
    └── test/java/<pkg>/shared/        # mirror of main (dto, error, validation tests)

application-core/                      # aggregator only (no sources at this level)
├── domain/
│   ├── build.gradle                   # pure Java ONLY; NO Spring/JPA deps
│   └── src/
│       ├── main/java/<pkg>/domain/
│       │   ├── entities/              # aggregates, entities, domain primitives (Principle XIV)
│       │   │   ├── Article.java
│       │   │   ├── ArticleId.java
│       │   │   ├── Author.java
│       │   │   ├── AuthorId.java
│       │   │   ├── Content.java
│       │   │   ├── PersonName.java
│       │   │   └── Title.java
│       │   └── repositories/          # domain-owned driven ports
│       │       └── ArticleRepository.java
│       └── test/java/<pkg>/domain/    # plain-JUnit entity tests (NO Spring context)
├── application/
│   ├── build.gradle                   # depends on domain, input-ports, output-ports, shared-kernel
│   └── src/
│       ├── main/java/<pkg>/application/
│       │   ├── CreateArticleHandler.java
│       │   ├── UpdateArticleHandler.java
│       │   ├── DeleteArticleHandler.java
│       │   ├── FindArticleHandler.java
│       │   ├── GetAllArticlesHandler.java
│       │   └── ArticleMapper.java     # domain <-> DTO mapping
│       └── test/java/<pkg>/application/  # handler tests with mocked ports
├── input-ports/
│   ├── build.gradle                   # depends on domain + shared-kernel ONLY
│   └── src/
│       ├── main/java/<pkg>/application/
│       │   ├── command/               # driving-port commands (validated input)
│       │   │   ├── CreateArticleCommand.java
│       │   │   ├── UpdateArticleCommand.java
│       │   │   └── DeleteArticleCommand.java
│       │   ├── query/                 # driving-port queries
│       │   │   ├── FindArticleQuery.java
│       │   │   └── GetAllArticlesQuery.java
│       │   └── ports/in/              # driving-port interfaces (use cases)
│       │       ├── CreateArticleUseCase.java
│       │       ├── UpdateArticleUseCase.java
│       │       ├── DeleteArticleUseCase.java
│       │       ├── FindArticleUseCase.java
│       │       └── GetAllArticlesUseCase.java
│       └── test/java/<pkg>/application/  # command/query validation tests
└── output-ports/
    ├── build.gradle                   # depends on domain + shared-kernel ONLY
    └── src/
        ├── main/java/<pkg>/application/ports/out/
        │   ├── AuthorOutputPort.java   # driven-port interfaces for external systems
        │   ├── DomainLogger.java       # core-owned logging port (Principle XI)
        │   ├── CipherPort.java         # encrypt / mask SENSITIVE values (Principle XIII)
        │   └── CallerIdentityPort.java # anonymous actor now, authenticated later (Principle XVI)
        └── test/                      # port contract tests if applicable
```

### Adapters & Assembly (api, postgres, kafka, in-memory, proxy adapters, proxy gateway, wiring)

```text
api-adapter/                           # DRIVING adapter (REST in)
├── build.gradle                       # Spring Web + openapi-generator ONLY here
└── src/
    ├── main/
    │   ├── java/<pkg>/api/
    │   │   ├── ArticleController.java # @RestController; implements the generated
    │   │   │                          # interface; depends ONLY on ports/in
    │   │   ├── ArticleApi.java        # routes/annotations bound to the contract
    │   │   ├── ApiRequest.java        # inbound DTOs
    │   │   ├── ApiResponse.java       # outbound DTOs
    │   │   ├── ApiMapper.java         # request/response <-> command/query/DTO
    │   │   ├── ApiErrorHandler.java
    │   │   ├── ApiGlobalExceptionHandler.java
    │   │   └── ApiResultUtils.java    # Either<Error, T> -> ResponseEntity mapping
    │   └── resources/openapi/           # strict schemas, declared security (Principle XV)
    │       └── business-registration.openapi.yaml  # OpenAPI 3.1 SOURCE OF TRUTH
    └── test/java/<pkg>/api/
        ├── ArticleControllerTest.java
        └── ApiContractTest.java       # contract-vs-implementation drift check

postgres-adapter/                      # DRIVEN adapter (REQUIRED production persistence)
├── build.gradle                       # Spring Data JPA/Hibernate + Postgres driver ONLY here
└── src/
    ├── main/java/<pkg>/persistence/
    │   ├── ArticleEntity.java         # JPA @Entity (NEVER leaks to core)
    │   ├── ArticleJpaRepository.java  # Spring Data repository interface
    │   ├── ArticlePersistenceMapper.java  # entity <-> domain mapping
    │   └── PostgresArticleRepositoryAdapter.java  # implements domain ArticleRepository
    └── test/java/<pkg>/persistence/   # @DataJpaTest vs PostgreSQL (Testcontainers)

in-memory-repositories/                # DRIVEN adapter (test/dev ONLY; NEVER production default)
├── build.gradle                       # depends on domain ONLY; NO Spring Data
└── src/
    ├── main/java/<pkg>/repositories/
    │   └── InMemoryArticleRepository.java  # implements domain ArticleRepository
    └── test/java/<pkg>/repositories/

kafka-adapter/                         # DRIVEN adapter (Kafka in/out; Principle VI)
├── build.gradle                       # Spring for Apache Kafka ONLY here
└── src/
    ├── main/
    │   ├── java/<pkg>/kafka/
    │   │   ├── publisher/             # implements ports.out.*Publisher
    │   │   │   └── KafkaArticleEventPublisher.java
    │   │   ├── consumer/              # implements ports.out.*Consumer
    │   │   │   └── KafkaArticleCommandConsumer.java
    │   │   └── KafkaMessageMapper.java  # core types <-> Kafka payloads
    │   └── resources/asyncapi/
    │       └── asyncapi.yaml          # AsyncAPI 3.x SOURCE OF TRUTH (topics, schemas)
    └── test/java/<pkg>/kafka/         # Testcontainers Kafka + schema compatibility

proxy-gateway/                         # SEPARATELY DEPLOYED simple proxy web services (Principle VII)
└── author-proxy/                      # one deployable per external system
    ├── build.gradle                   # Boot plugin; NO application-core dependency
    └── src/
        ├── main/java/<pkg>/proxy/author/
        │   ├── AuthorProxyApplication.java  # Boot entry point for the proxy
        │   ├── AuthorProxyController.java   # thin pass-through; generated interface
        │   ├── AuthorVendorClient.java      # ONLY place vendor URL/creds are known
        │   └── VendorAuthConfig.java
        └── resources/openapi/author-proxy.openapi.yaml  # proxy contract (Principle V)

spring-boot-assembly/                  # WIRING ONLY; the single Spring Boot application
├── build.gradle                       # depends on ALL modules; Boot plugin ONLY here
└── src/
    ├── main/
    │   ├── java/<pkg>/
    │   │   ├── Application.java       # @SpringBootApplication entry point
    │   │   └── assembly/
    │   │       ├── ApplicationAssembler.java       # @Configuration: binds adapters to ports
    │   │       └── CommandQueryBusAssembler.java   # @Configuration: wires handlers
    │   └── resources/
    │       └── application.yaml       # datasource (PostgreSQL), ArchUnit layer config
    └── test/
        ├── java/<pkg>/
        │   ├── AdaptersArchitectureTest.java
        │   ├── CommandArchitectureTest.java
        │   ├── DomainArchitectureTest.java
        │   ├── HandlerArchitectureTest.java
        │   ├── InputPortsArchitectureTest.java
        │   ├── OutputPortsArchitectureTest.java
        │   ├── QueryArchitectureTest.java
        │   └── SharedKernelArchitectureTest.java
        └── resources/
            └── application.yaml
```

Module dependency rules (MUST hold; enforced by ArchUnit in
`spring-boot-assembly` tests):

- `spring-boot-assembly` MAY depend on all modules; NO other module MAY
  depend on `spring-boot-assembly`.
- Driving/driven adapters (`api-adapter`, `postgres-adapter`,
  `kafka-adapter`, `in-memory-repositories`, `author-proxy-adapter`) MAY
  depend on `input-ports`/`output-ports`/`domain` and `shared-kernel`; they
  MUST NEVER depend on each other.
- `application` MAY depend on `domain`, `input-ports`, `output-ports`, and
  `shared-kernel`.
- `input-ports` and `output-ports` MAY depend on `domain` and
  `shared-kernel` ONLY.
- `domain` MAY depend on `shared-kernel` ONLY (plus plain-Java stdlib).
- `shared-kernel` MUST NOT depend on any sibling module.
- Every module MUST mirror `src/main` with `src/test` using the same
  package split.
- A new adapter (queue consumer, scheduler, proxy client) MUST be a new
  top-level module following the `*-adapter` shape above; adding I/O inside
  `application-core` is FORBIDDEN.
- A new external system MUST add both a `<system>-proxy-adapter` module and
  a `proxy-gateway/<system>-proxy` deployable, with the OpenAPI contract
  owned by the proxy (Principle VII).
- `proxy-gateway` modules MUST NOT depend on any application module, and
  application modules MUST NOT depend on `proxy-gateway`.
- A new asynchronous contract MUST be declared in the AsyncAPI document
  with a versioned topic name (Principle VI).

Cross-cutting wiring for core-domain services:

- Transactional outbox and saga state MUST be persisted through
  `postgres-adapter` in the service's own private schema (Principle X).
- Health and readiness endpoints, metrics, tracing, correlation-id propagation
  and rate limiting MUST be wired in `spring-boot-assembly` configuration
  (Principle XI).
- Encryption and masking of SENSITIVE fields, and the identity port, MUST be
  wired in `spring-boot-assembly` configuration: the cipher adapter next to the
  persistence adapter, and an anonymous actor adapter wherever no authentication
  exists yet (Principles XIII and XVI).
- Configuration MUST be externalized per environment (Principles IX and XII).
- The outbox relay mechanism (polling publisher or transaction-log tailing) and
  its lag metric MUST be named in `spring-boot-assembly` configuration
  (Principle VI).
- Architecture decision records MUST live in `docs/adr/` (or the location
  recorded in the plan) as dated, immutable Markdown files (Principle XXII).

## Governance

This constitution supersedes all other coding practices for this project.
Conflicts between team conventions and these principles MUST be resolved in
favor of the constitution.

- Amendments REQUIRE a documented proposal, explicit approval, and a
  migration plan for affected code and tests.
- Versioning follows semantic versioning: MAJOR for backward-incompatible
  governance/principle removals or redefinitions, MINOR for new
  principles/sections or materially expanded guidance, PATCH for
  clarifications, wording, or typo fixes.
- Compliance review is MANDATORY: all specs, plans, tasks, and pull requests
  MUST verify adherence to Principles I-XXII and record any boundary, DI,
  contract, data-ownership, scope-classification, sensitivity-classification,
  access-control, saga-and-aggregate, lifecycle, or transport-security
  decisions.
- Security obligations that cannot be mechanically verified MUST appear as
  named review-checklist items rather than being left implicit.
- Accepting a security risk without mitigation, or downgrading a field's
  sensitivity classification, MUST be recorded in the plan with an owner, a
  rationale, any compensating controls, and an expiry date; an unrecorded
  downgrade or an acceptance without an expiry is a violation (Principle XXII).
- Boundary, classification, and sensitivity decisions MUST have a dated
  architecture decision record in the repository; a decision that exists only in
  a plan document is not recorded (Principle XXII).
- Reclassifying a service between core and non-core is an amendment-class
  change and MUST follow the amendment procedure above.

**Version**: 2.3.0 | **Ratified**: 2026-09-23 | **Last Amended**: 2026-09-25
