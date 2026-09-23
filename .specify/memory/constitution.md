<!-- Sync Impact Report
Version change: 1.0.1 -> 1.1.0 (MINOR: new normative directory structure template)
Modified principles: none
Added sections:
- Appendix A: Normative Directory Structure Template (Gradle Groovy multi-module, adapted from emedina/hexagonal-spring-ref-app)
Removed sections: none
Follow-up TODOs: none
Reference: https://github.com/emedina/hexagonal-spring-ref-app.git (Java 25, Spring Boot 4.0.1; Maven -> mapped to Gradle Groovy; added postgres-adapter)
NOTE: This HTML comment is temporary scratch material for human review and MUST be removed before commit.
-->

# Spring Boot Hexagonal Application Constitution

## Core Principles

### I. Architecture Layout & Packaging (NON-NEGOTIABLE)

Every feature MUST be organized into three distinct layers with clear
structural separation: `domain` (core business logic: entities, value
objects, domain services, business rules), `ports` (interfaces: inbound
use-case ports and outbound driven ports), and `adapters` (infrastructure,
web, and database implementations plus configuration).

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

## Technology Constraints & Build Standards

The stack is Java 25 with Spring Boot 4 for adapters and configuration
only. The build tool is Gradle with Groovy DSL (`build.gradle`); Java,
Spring Boot, and dependency versions MUST be locked in the build file and
verified in CI. PostgreSQL is the REQUIRED primary relational database;
persistence (Spring Data JPA/Hibernate against PostgreSQL), web
(Spring MVC), and messaging clients MUST
  reside ONLY in `adapters`; core MUST interact with them solely through
  driven-port interfaces.
- DTOs, JPA entities, and framework-specific models MUST be mapped at the
  adapter edge; shared cross-layer models carrying framework annotations
  are FORBIDDEN.

## Development Workflow & Quality Gates

Every change MUST verify hexagonal compliance before merge:

- New business logic MUST be added to `domain` with core-owned ports; new
  I/O MUST be added as an adapter behind a driven port interface.
- Pull requests MUST include: (1) ArchUnit/boundary tests green,
  (2) plain-JUnit core tests without Spring context, (3) adapter tests for
  each new implementation, (4) a reviewer checklist confirming no Spring/JPA
  imports in `domain`/`ports` and constructor injection only.
- Violations of Principles I-IV MUST block merge; justified exceptions
  require a constitution amendment, not an ad-hoc waiver.

## Appendix A: Normative Directory Structure Template

### Core Modules (shared-kernel, application-core)

This appendix is NORMATIVE. New projects and features MUST follow this
Gradle (Groovy DSL) multi-module layout, adapted from the reference
architecture https://github.com/emedina/hexagonal-spring-ref-app.git
(Java 25, Spring Boot 4.0.1; reference uses Maven — mapped here to Gradle
Groovy per constitution stack pin, with a `postgres-adapter` added as the
REQUIRED production persistence adapter). `<pkg>` is the base package
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

### Adapters & Assembly (api, postgres, in-memory, external, wiring)

```text
api-adapter/                           # DRIVING adapter (REST in)
├── build.gradle                       # Spring Web ONLY here; depends on input-ports + shared-kernel
└── src/
    ├── main/java/<pkg>/api/
    │   ├── ArticleController.java     # @RestController; depends ONLY on ports/in interfaces
    │   ├── ArticleApi.java            # API contract (routes, OpenAPI)
    │   ├── ApiRequest.java            # inbound DTOs
    │   ├── ApiResponse.java           # outbound DTOs
    │   ├── ApiMapper.java             # request/response <-> command/query/DTO
    │   ├── ApiErrorHandler.java
    │   ├── ApiGlobalExceptionHandler.java
    │   └── ApiResultUtils.java        # Either<Error, T> -> ResponseEntity mapping
    └── test/java/<pkg>/api/           # controller slice tests, mocked use cases

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

author-external-adapter/               # DRIVEN adapter template slot (external clients)
├── build.gradle                       # REST/messaging client deps ONLY here
└── src/
    ├── main/java/<pkg>/external/
    │   └── AuthorExternalAPIAdapter.java  # implements AuthorOutputPort
    └── test/java/<pkg>/external/

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
  `in-memory-repositories`, `author-external-adapter`) MAY depend on
  `input-ports`/`output-ports`/`domain` and `shared-kernel`; they MUST NEVER
  depend on each other.
- `application` MAY depend on `domain`, `input-ports`, `output-ports`, and
  `shared-kernel`.
- `input-ports` and `output-ports` MAY depend on `domain` and
  `shared-kernel` ONLY.
- `domain` MAY depend on `shared-kernel` ONLY (plus plain-Java stdlib).
- `shared-kernel` MUST NOT depend on any sibling module.
- Every module MUST mirror `src/main` with `src/test` using the same
  package split.
- A new adapter (REST client, queue consumer, scheduler) MUST be a new
  top-level module following the `*-adapter` shape above; adding I/O inside
  `application-core` is FORBIDDEN.

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
  MUST verify adherence to Principles I-IV and record any boundary or DI
  decisions.

**Version**: 1.1.0 | **Ratified**: 2026-09-23 | **Last Amended**: 2026-09-23
