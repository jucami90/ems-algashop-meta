# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository structure

This is a **meta-repository** using git submodules. Each microservice lives in its own repo:

- `microservices/ordering` → `ems-algashop-ordering` (Spring Boot, port 8080)
- `microservices/billing` → `ems-algashop-billing` (Spring Boot, port 8082)
- `docs/` → `ems-algashop-docs`

Both microservices share the same stack: **Java 21, Spring Boot 3.x, Gradle, H2 (file-based dev / in-memory test)**.

## Commands

All Gradle commands are run from within the microservice directory (e.g., `microservices/ordering/` or `microservices/billing/`).

```bash
# Build
./gradlew build

# Run unit tests only (classes matching *Test, excluding *IT)
./gradlew test

# Run integration tests only (classes matching *IT, excluding *Test)
./gradlew integrationTest

# Run all tests (both unit and integration)
./gradlew check

# Run a single test class
./gradlew test --tests "com.algaworks.algashop.ordering.domain.model.order.OrderCancelTest"

# Run the application
./gradlew bootRun
```

### External dependency for integration tests

The ordering service's shipping integration talks to a RapiDex API mock. Start WireMock before running integration tests that exercise the HTTP client:

```bash
# From repository root
docker compose up -d
```

WireMock loads stubs from `etc/wiremock/` and listens at `http://localhost:8780`.

The shipping provider can be switched between `RAPIDEX` (real HTTP client) and the fake in-memory implementation via `algashop.integrations.shipping.provider` in `application.yaml`.

## Architecture

Both microservices follow **Domain-Driven Design** with a strict layered structure:

### Layers

```
domain/model/         ← Pure Java domain model (aggregates, value objects, domain services, events)
application/          ← Application services (use case orchestration, input/output DTOs)
infrastructure/       ← Spring/JPA adapters (persistence, listeners, HTTP clients, fakes)
```

### Domain layer conventions

- **Aggregate roots** implement `AggregateRoot<ID>` and extend `AbstractEventSourceEntity`.
- Domain events are registered with `publishDomainEvent(event)` inside aggregate methods and dispatched by Spring's `@EventListener` in `infrastructure/listener/`.
- Domain objects have **no JPA annotations** in the ordering service — they are plain Java. (The billing service's `Invoice` is an exception: it is a `@Entity` directly.)
- Value objects (e.g., `Money`, `Quantity`, `Email`, `CustomerId`) enforce their own invariants in constructors.
- Repository interfaces are defined in the domain layer (e.g., `Customers`, `Orders`, `ShoppingCarts`) and implemented in infrastructure.

### Persistence pattern (ordering service)

The ordering service uses a double-mapping pattern to keep the domain model clean:

1. **`*PersistenceEntity`** — JPA `@Entity` classes, kept in `infrastructure/persistence/`.
2. **`*PersistenceEntityAssembler`** — converts domain object → persistence entity.
3. **`*PersistenceEntityDisassembler`** — converts persistence entity → domain object.
4. **`*PersistenceProvider`** implements the domain repository interface and coordinates the above.

Optimistic locking: the `version` field on aggregates is written back via reflection (`ReflectionUtils.setField`) after save, since the field has no public setter by design.

### Test conventions

| Suffix | Type | Spring context |
|--------|------|----------------|
| `*Test` | Unit test (Mockito, no Spring) | No |
| `*IT` | Integration test | Yes (`@SpringBootTest`) |

Test data builders (`*TestDataBuilder`) provide pre-configured objects for tests — use them instead of constructing objects manually.