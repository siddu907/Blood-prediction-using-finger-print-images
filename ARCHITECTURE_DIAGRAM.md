# Architecture Diagram

```mermaid
flowchart LR
    Client[Client / Postman / Frontend]

    subgraph API[FastAPI Application]
        Routes[Routers\nAuth, Users, Tickets, Comments, Attachments, Notifications, Dashboard, Audit Logs]
        Services[Services\nAuth, Ticket, User, Category, SLA, Notification, Audit]
        Core[Core\nSecurity, Permissions, Dependencies, Exception Handlers]
    end

    subgraph Data[Data Layer]
        Repos[Repositories\nUser, Ticket, Category, Notification, Refresh Token]
        Models[SQLAlchemy Models\nUser, Role, Ticket, Category, Comment, Attachment, Notification, Audit Log]
        DB[(PostgreSQL Database)]
    end

    subgraph Background[SLA Monitoring]
        Worker[SLA Worker]
        Breach[SLA Breach Notifications]
    end

    Client -->|HTTP Request| Routes
    Routes --> Services
    Services --> Repos
    Repos --> Models
    Models --> DB

    Services --> Core
    Worker -->|checks overdue tickets| Repos
    Worker -->|creates| Breach
    Breach --> DB

    Notifications[Notifications] --> Client
    Dashboard[Dashboard Metrics] --> Client
```

## Architecture overview

The backend follows a layered architecture:

1. Client layer
   - Frontend app, Postman, or API consumers.

2. API layer
   - FastAPI routers expose endpoints grouped by domain.

3. Service layer
   - Business logic for authentication, ticket workflows, notifications, SLA checks, and user management.

4. Repository/data layer
   - SQLAlchemy repositories interact with database models.

5. Persistence layer
   - PostgreSQL stores users, roles, tickets, categories, comments, attachments, notifications, and audit records.

6. Background processing layer
   - SLA worker checks overdue tickets and creates breach notifications on a scheduled interval.

## Request flow example

```mermaid
sequenceDiagram
    participant C as Client
    participant R as Router
    participant S as Service
    participant DB as Database

    C->>R: API request with bearer token
    R->>S: Validate request and business rules
    S->>DB: Read or update ticket/user data
    DB-->>S: Result
    S-->>R: Response payload
    R-->>C: JSON response
```

This shows the standard flow used across the project: request → router → service → database → return response.
