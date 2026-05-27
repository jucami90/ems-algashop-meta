# AlgaShop - Event-Driven Microservices Example - DDD, Spring Boot, Java 21

AlgaShop is an e-commerce backend built as an **event-driven microservices system**. It handles the full purchase lifecycle: customer registration, shopping cart management, order placement, shipping cost calculation, invoice generation, and payment processing.

The project was designed as a learning platform for **Domain-Driven Design (DDD)** and microservices architecture patterns using the Spring ecosystem.

---

## Services

| Service | Description | Port |
|---------|-------------|------|
| `ordering` | Customers, shopping carts, orders, checkout, shipping | `8080` |
| `billing` | Invoices and payment processing | `8082` |
| `wiremock` | Mock for the RapiDex shipping carrier API | `8780` |

Each service lives in its own Git repository and is included here as a **git submodule**.

---

## Tech Stack

- **Java 21**
- **Spring Boot 3.x** (Web, Data JPA, AOP)
- **Gradle** (per-service build)
- **H2** — file-based for development, in-memory for tests
- **WireMock** — Docker-based mock of the external shipping API
- **Lombok** — boilerplate reduction
- **ModelMapper** — DTO mapping
- **Mockito** — unit testing with inline mock agent

---

## Architecture Overview

The system follows **Domain-Driven Design** principles. Each microservice is internally structured in three layers: **Domain**, **Application**, and **Infrastructure**. The domain layer is completely isolated from framework concerns.

```
┌─────────────────────────────────────────────────────────────────────┐
│                           AlgaShop System                           │
│                                                                     │
│  ┌──────────────────────────────┐  ┌──────────────────────────────┐ │
│  │      ordering-service        │  │      billing-service         │ │
│  │          :8080               │  │          :8082               │ │
│  │                              │  │                              │ │
│  │  ┌────────────────────────┐  │  │  ┌────────────────────────┐ │ │
│  │  │   Application Layer    │  │  │  │   Application Layer    │ │ │
│  │  │  (Application          │  │  │  │  (InvoiceManagement    │ │ │
│  │  │   Services, DTOs)      │  │  │  │   ApplicationService)  │ │ │
│  │  └──────────┬─────────────┘  │  │  └──────────┬────────────┘ │ │
│  │             │                │  │             │               │ │
│  │  ┌──────────▼─────────────┐  │  │  ┌──────────▼────────────┐ │ │
│  │  │     Domain Layer       │  │  │  │     Domain Layer      │ │ │
│  │  │  Customer, Order,      │  │  │  │  Invoice, Payment,    │ │ │
│  │  │  ShoppingCart,         │  │  │  │  CreditCard           │ │ │
│  │  │  Domain Services,      │  │  │  │  Domain Events        │ │ │
│  │  │  Domain Events         │  │  │  └──────────┬────────────┘ │ │
│  │  └──────────┬─────────────┘  │  │             │               │ │
│  │             │                │  │  ┌──────────▼────────────┐ │ │
│  │  ┌──────────▼─────────────┐  │  │  │  Infrastructure Layer │ │ │
│  │  │  Infrastructure Layer  │  │  │  │  (JPA, Listeners)    │ │ │
│  │  │  Persistence Providers │  │  │  └───────────────────────┘ │ │
│  │  │  Assemblers/           │  │  │                              │ │
│  │  │  Disassemblers         │  │  │         H2 Database          │ │
│  │  │  HTTP Clients          │  │  └──────────────────────────────┘ │
│  │  │  Event Listeners       │  │                                   │
│  │  └──────────┬─────────────┘  │  ┌──────────────────────────────┐ │
│  │             │                │  │      WireMock :8780           │ │
│  │     H2 Database              │  │  (RapiDex Shipping API Mock)  │ │
│  └──────────────────────────────┘  └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Domain Model

### Ordering Service

The ordering service owns three aggregate roots:

#### `Customer`
Represents a registered buyer. Supports archiving (data anonymization), loyalty points accumulation, and preference management (promotion notifications). Status transitions: active → archived (irreversible).

#### `ShoppingCart`
Belongs to one customer. Items can be added, removed, or have their quantities adjusted. Checkout converts the cart into an `Order` (cart is then emptied). The `BuyNow` flow creates an order directly without a cart.

#### `Order`
Lifecycle governed by a strict state machine:

```
DRAFT ──► PLACED ──► PAID ──► READY
  │          │         │
  └──────────┴─────────┴──► CANCELED
```

An order requires shipping info, billing info, a payment method, and at least one item before it can be placed. Shipping cost is calculated via the `ShippingCostService` (backed by RapiDex or a fake implementation).

---

### Billing Service

#### `Invoice`
Generated from a placed order. Lifecycle:

```
UNPAID ──► PAID
  │
  └──► CANCELED
```

Payment is processed through a `PaymentGatewayService` (currently a fake implementation). If payment capture fails, the invoice is automatically canceled. An invoice can reference a `CreditCard` as the payment source.

---

## Persistence Design

### Ordering Service — Decoupled Persistence

The ordering service domain model contains **no JPA annotations**. It is pure Java. Persistence is handled by an explicit mapping layer:

```
Domain Aggregate  ◄──────────────────────────────────────────►  Database
(plain Java)       Assembler / Disassembler / PersistenceEntity   (H2/JPA)

  Customer         CustomerPersistenceEntityAssembler             customer table
                   CustomerPersistenceEntityDisassembler
                   CustomerPersistenceEntity (@Entity)

  Order            OrderPersistenceEntityAssembler                order table
                   OrderPersistenceEntityDisassembler             order_item table
                   OrderPersistenceEntity (@Entity)

  ShoppingCart     ShoppingCartPersistenceEntityAssembler         shopping_cart table
                   ShoppingCartPersistenceEntityDisassembler      shopping_cart_item table
                   ShoppingCartPersistenceEntity (@Entity)
```

**Persistence Providers** (e.g., `CustomersPersistenceProvider`) implement domain repository interfaces (e.g., `Customers`) and coordinate the assemblers, disassemblers, and Spring Data repositories.

**Optimistic locking** is implemented via `@Version` on every JPA entity. Because the domain aggregate has no public version setter, the version is synced back using reflection after each save.

**Domain events** flow from the domain aggregate → are copied onto the JPA `PersistenceEntity` (which extends Spring's `AbstractAggregateRoot`) → and are dispatched by Spring after the transaction commits, consumed by `@EventListener` components in the infrastructure layer.

### Billing Service — Direct JPA Mapping

The billing service takes a simpler approach: `Invoice` is annotated directly as a JPA `@Entity`. It extends `AbstractAuditableAggregateRoot`, which provides both auditing fields and Spring's domain event publishing mechanism. Domain events are registered inline via `registerEvent(...)` and dispatched by Spring after the transaction.

---

## External Integration — Shipping (RapiDex API)

The `ShippingCostService` domain interface has two implementations:

| Implementation | Used when |
|---------------|-----------|
| `ShippingCostServiceRapidexImpl` | `algashop.integrations.shipping.provider=RAPIDEX` |
| `ShippingCostServiceFakeImpl` | `algashop.integrations.shipping.provider=FAKE` (or any other value) |

The HTTP client (`RapiDexAPIClient`) is a Spring 6 declarative `HttpExchange` interface. WireMock, started via Docker Compose, mocks the endpoint `POST /api/delivery-cost` on port `8780`.

---

## Running Locally

**1. Start the WireMock shipping mock (required for the Rapidex integration):**

```bash
docker compose up -d
```

**2. Run each microservice from its directory:**

```bash
cd microservices/ordering
./gradlew bootRun

# in another terminal
cd microservices/billing
./gradlew bootRun
```

**H2 Console** is available at:
- Ordering: `http://localhost:8080/h2-console` (JDBC URL: `jdbc:h2:file:~/ordering`)
- Billing: `http://localhost:8082/h2-console` (JDBC URL: `jdbc:h2:file:~/billing`)

---

## Running Tests

From within a microservice directory:

```bash
# Unit tests only
./gradlew test

# Integration tests only
./gradlew integrationTest

# Both
./gradlew check

# Single class
./gradlew test --tests "com.algaworks.algashop.ordering.domain.model.order.OrderCancelTest"
```

Unit tests (`*Test`) run without a Spring context. Integration tests (`*IT`) start a full Spring Boot context against an in-memory H2 database.

---

## Project Structure

```
algashop/                         ← meta-repository (this repo)
├── microservices/
│   ├── ordering/                 ← git submodule (ems-algashop-ordering)
│   │   └── src/
│   │       ├── main/java/.../ordering/
│   │       │   ├── application/  ← application services & DTOs
│   │       │   ├── domain/model/ ← aggregates, value objects, domain services
│   │       │   └── infrastructure/ ← persistence, listeners, HTTP clients
│   │       └── test/
│   └── billing/                  ← git submodule (ems-algashop-billing)
│       └── src/
│           ├── main/java/.../billing/
│           │   ├── domain/model/ ← Invoice aggregate + application services
│           │   └── infraestructure/ ← persistence, payment gateway impl
│           └── test/
├── docs/                         ← git submodule (ems-algashop-docs, domain diagrams)
├── etc/wiremock/                 ← WireMock stub mappings for RapiDex API
└── docker-compose.yml            ← starts WireMock
```