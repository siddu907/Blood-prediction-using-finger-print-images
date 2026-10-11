# System Architecture

This is the architecture of the implemented Order Fulfillment & Inventory
Management Platform. It is a modular monolithic REST API: the modules run in
one FastAPI application and share one SQLAlchemy session and relational
database. Mermaid-capable Markdown viewers render the diagrams.

## Component architecture

```mermaid
flowchart TB
    Customer[Customer client]
    WarehouseStaff[Warehouse manager / fulfillment agent client]
    Administrator[Admin client]
    Postman[Postman]
    Swagger[Swagger UI / OpenAPI]

    subgraph API["FastAPI application"]
        Main["app/main.py<br/>Application setup and router registration"]
        Routes["Routers<br/>auth, users/catalog, inventory, orders,<br/>fulfillment, payments, returns,<br/>notifications, dashboards, audit, settings"]
        Schemas["Pydantic schemas<br/>Request validation and data contracts"]
        Auth["Authentication dependency<br/>JWT verification, active user, token version"]
        Access["Authorization dependencies<br/>Role checks and warehouse membership"]
        Services["Domain services<br/>orders, inventory, payments,<br/>idempotency, audit, notifications"]
        Repositories["Repositories<br/>Shared order and inventory queries"]
        Models["SQLAlchemy models<br/>Entities, foreign keys and constraints"]
        Session["Database session<br/>SQLAlchemy engine and request-scoped session"]
        Errors["Exception handlers<br/>Consistent HTTP error responses"]
    end

    DB[(PostgreSQL<br/>business data and transactional records)]
    Migrations["Alembic migrations"]

    Customer --> Swagger
    WarehouseStaff --> Swagger
    Administrator --> Swagger
    Customer --> Postman
    WarehouseStaff --> Postman
    Administrator --> Postman

    Swagger --> Main
    Postman --> Main
    Main --> Routes
    Routes --> Schemas
    Routes --> Auth
    Auth --> Access
    Routes --> Services
    Services --> Repositories
    Services --> Models
    Repositories --> Models
    Models --> Session
    Session <--> DB
    Routes -. errors .-> Errors
    Services -. errors .-> Errors
    Migrations -. schema changes .-> DB
```

### Component responsibilities

- **FastAPI routers** expose the HTTP endpoints and coordinate validation,
  authorization, domain operations, and commits.
- **Pydantic schemas** validate incoming request bodies and define typed API
  inputs and outputs.
- **Authentication** decodes the access JWT and checks that its user still
  exists, is active, and has the current token version.
- **Authorization** applies role rules and checks warehouse membership before
  warehouse-scoped actions.
- **Domain services** implement order and return state machines, inventory
  reservation and transfer, pricing/tax snapshots, assignment, and payment
  demo behavior. Shared helpers write audit and notification rows.
- **Repositories** hold reusable database queries. SQLAlchemy models define the
  relational entities and constraints.
- **Database sessions** are created per request. PostgreSQL commits persist
  business changes and associated history, inventory ledger, audit, and
  notification records together.
- **Alembic** applies schema migrations independently of API request handling.

## Request and transaction flow

```mermaid
sequenceDiagram
    actor Client
    participant API as FastAPI router
    participant Guard as Authentication and authorization
    participant Service as Domain service
    participant DB as PostgreSQL

    Client->>API: HTTP request with JSON and bearer token
    API->>API: Validate path, query, headers, and body
    API->>Guard: Resolve user and check role/resource scope
    Guard->>DB: Read user and warehouse membership
    DB-->>Guard: User / membership
    Guard-->>API: Authorized user
    API->>Service: Execute requested business operation
    Service->>DB: Read/write within request database session
    Note over Service,DB: Inventory reservations use conditional updates;<br/>state-changing paths lock relevant rows.
    Service->>DB: Add ledger, order history, audit, and in-app notifications
    API->>DB: Commit transaction
    DB-->>API: Committed result
    API-->>Client: JSON response
```

If a domain operation raises an error before commit, the request session closes
without committing its pending changes. The registered exception handlers map
domain, validation, HTTP, integrity, and unexpected errors to JSON responses.

## Order and inventory transaction

```mermaid
sequenceDiagram
    actor Customer
    participant Orders as Orders router / service
    participant Inventory as Inventory service
    participant DB as PostgreSQL

    Customer->>Orders: POST /orders (product IDs and quantities)
    Orders->>DB: Select active warehouse and lock/check inventory rows
    Orders->>DB: Insert order and order lines with price/tax snapshots
    loop Each order line
        Orders->>Inventory: Reserve requested quantity
        Inventory->>DB: UPDATE balance only if available_quantity >= requested
        alt Reservation succeeds
            Inventory->>DB: Insert reservation ledger and audit rows
        else Insufficient stock
            Inventory-->>Orders: Return domain conflict
            Note over Orders,DB: Request transaction rolls back order and prior line changes
        end
    end
    Orders->>DB: Insert order history, notification, optional idempotency record
    Orders->>DB: Commit one transaction
    Orders-->>Customer: Created order and item snapshots
```

PostgreSQL serializes conflicting updates to the same inventory row. The
conditional update re-evaluates stock after waiting, so simultaneous requests
cannot reserve more units than are available. Database constraints also
prohibit negative balance quantities and enforce one inventory row per
product/warehouse pair.

## Deployment and integration boundaries

- Run the FastAPI application with an ASGI server such as Uvicorn and connect it
  to PostgreSQL using `DATABASE_URL`.
- The API, domain services, and persistence layer are in the same application
  process; there is no separate service-to-service network layer.
- Audit entries, inventory ledger rows, and in-app notifications are persisted
  in the database as part of business transactions.
- The current implementation does not configure an external message broker,
  background worker, email/SMS/push provider, carrier integration, or payment
  gateway. Payment confirmation and refund status are local demo behavior.
- SQLite is used by default in fast tests; PostgreSQL is needed to validate
  production row-locking behavior.

See the [ER diagram](./er-diagram.md) for the database entities and
relationships, the [API reference](./api.md) for endpoints, and the
[project README](../README.md) for setup and testing.
