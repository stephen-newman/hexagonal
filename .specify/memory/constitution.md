<!-- Sync Impact Report
Version change: 2.1.0 -> 2.2.0 (MINOR: five new security principles + expanded guidance)
Modified principles (rules added, no rule removed or redefined):
- V. Contract-First REST Driving Ports (OpenAPI) - contract strictness and
  response-exposure limits now cite Principle XV
- VII. External Web Services via Simple Proxy Web Services - egress allowlist
  (SSRF), untrusted vendor payloads, proxy credential handling
- X. Data Ownership & Cross-Service Consistency - sensitive data MUST NOT be
  copied into events or commands
- XI. Production Readiness: Resilience & Observability - log hygiene, redaction,
  audit-stream separation, per-business-key velocity limits, abuse alerting,
  capacity headroom and load-amplification tests
- XII. Independent Deployability & Evolution - gateway edge duties separated
  from in-service authorization
Added principles:
- XIII. Sensitive Data Protection
- XIV. Untrusted Input & Valid-by-Construction Domain Model
- XV. Secure HTTP Contracts & Response Minimisation
- XVI. Identity-Ready, Default-Deny Authorization
- XVII. Secrets, Keys & Secure Baseline Configuration
Added guidance:
- Technology Constraints: security tooling slots (contract lint with an OWASP
  rule set, contract fuzzing, secret and dependency scanning, encryption with
  versioned keys, future authentication stack) marked (decision needed); no
  library is pinned until the decision is recorded
- Development Workflow: gates (12) threat model is a plan artifact,
  (13) threat-to-test traceability, (14) security-aware contract lint and
  contract fuzzing, (15) negative-input suite returning documented 4xx only,
  (16) access-control regression and redaction tests, (17) secret and
  dependency scans, (18) the security stage blocks merge
- Governance: compliance review covers I-XVII; downgrading a sensitivity
  classification is recorded and reviewed in the plan
Sources: Daniel Deogun, Dan Bergh Johnsson, Daniel Sawano, "Secure by Design"
  (Manning, 1st ed., 2019) and Jose Haro Peralta, "Secure APIs: Design, Build,
  and Implement" (Manning, 1st ed., 2025). Rules are paraphrased from those
  sources; no book text is quoted.
Removed sections: none
Follow-up TODOs:
- TODO(SECURITY_TOOLING): pin the contract-linting, fuzzing, secret-scanning,
  dependency-scanning, and encryption libraries in Technology Constraints.
- TODO(SPEC_001): refresh specs/001-register-corporate-behalf/spec.md for
  Principles XIII-XVII (non-enumerable application ID, anonymous-actor fields,
  validation order and size limits, audit-trail vs hard-delete tension, and the
  manual queue as an audited operation).
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

Rationale: configuration and secrets are the shortest path from a small mistake
to a breach, so they are externalized, validated, rotatable, and default-deny.

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
- End-to-end tests MUST be limited to a small set of critical user journeys and
  MUST NOT be the primary verification mechanism.
- Violations of Principles I-XVII MUST block merge; justified exceptions
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
  MUST verify adherence to Principles I-XVII and record any boundary, DI,
  contract, data-ownership, scope-classification, sensitivity-classification,
  or access-control decisions.
- Security obligations that cannot be mechanically verified MUST appear as
  named review-checklist items rather than being left implicit.
- Accepting a security risk without mitigation, or downgrading a field's
  sensitivity classification, MUST be recorded in the plan with an owner and a
  rationale; an unrecorded downgrade is a violation.
- Reclassifying a service between core and non-core is an amendment-class
  change and MUST follow the amendment procedure above.

**Version**: 2.2.0 | **Ratified**: 2026-09-23 | **Last Amended**: 2026-09-24
