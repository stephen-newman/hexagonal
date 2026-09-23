<!-- Sync Impact Report
Version change: 1.2.0 -> 2.0.0 (MAJOR: Principle I scope redefined; new Principle VIII)
Modified principles:
- I. Architecture Layout & Packaging - scope narrowed to core-domain services
- II. Dependency Isolation - core-only scope marker added
- III. Ports & Inversion of Control - core-only scope marker added
Added principles:
- VIII. Architecture Scope: Hexagonal for Core Domains Only (NON-NEGOTIABLE)
Added guidance:
- Appendix A marked CORE-domain only
- Development Workflow gate (7): recorded core/non-core classification
- Governance: reclassification is amendment-class; review covers I-VIII
- Title renamed to reflect service-estate scope
- Universality: IV, V, VI, VII, Technology Constraints, Workflow and
  Governance apply to all services
Removed sections: none
Follow-up TODOs: none
Note: the prior Sync Impact Report had already been removed from this file, so
this report covers this amendment only. It is temporary scratch material and
MUST be removed before commit.
-->

# Spring Boot Services Constitution (Hexagonal Core, Conventional Non-Core)

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
  emission must be atomic with a state change, an outbox SHOULD be used.

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

Rules:
- Persistence, web (Spring MVC), and messaging clients MUST reside ONLY in
  `adapters`; core MUST interact with them solely through driven-port
  interfaces.
- DTOs, JPA entities, Kafka message types, and other framework-specific
  models MUST be mapped at the adapter edge; shared cross-layer models
  carrying framework annotations are FORBIDDEN.
- Contract documents (`openapi/`, `asyncapi/`) MUST be committed with the
  owning adapter module and validated in CI.

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
  classification recorded (Principle VIII).
- Violations of Principles I-VIII MUST block merge; justified exceptions
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
│       │   ├── entities/              # aggregates, entities, value objects
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
        │   └── AuthorOutputPort.java   # driven-port interfaces for external systems
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
    │   └── resources/openapi/
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
  MUST verify adherence to Principles I-VIII and record any boundary, DI,
  contract, or scope-classification decisions.
- Reclassifying a service between core and non-core is an amendment-class
  change and MUST follow the amendment procedure above.

**Version**: 2.0.0 | **Ratified**: 2026-09-23 | **Last Amended**: 2026-09-23
