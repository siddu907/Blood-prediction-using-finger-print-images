# Database Schema

The system uses PostgreSQL with SQLAlchemy ORM and Alembic migrations.

```mermaid
erDiagram
    USERS ||--o| CUSTOMERS : owns
    CUSTOMERS ||--o{ TICKETS : creates
    USERS ||--o{ TICKETS : assigned_to
    CATEGORIES ||--o{ TICKETS : classifies
    TICKETS ||--o{ COMMENTS : contains
    USERS ||--o{ COMMENTS : writes
    TICKETS ||--o{ ATTACHMENTS : contains
    USERS ||--o{ ATTACHMENTS : uploads
    USERS ||--o{ NOTIFICATIONS : receives
    TICKETS ||--o{ NOTIFICATIONS : references
    USERS ||--o{ AUDIT_LOGS : performs

    USERS {
        int id PK
        varchar full_name
        varchar email UK
        varchar password_hash
        varchar role
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }

    CUSTOMERS {
        int id PK
        int user_id FK_UK
        varchar phone_number
        varchar company
        varchar address
        varchar status
        timestamp created_at
        timestamp updated_at
    }

    CATEGORIES {
        int id PK
        varchar name UK
        varchar description
        boolean is_active
        timestamp created_at
        timestamp updated_at
    }

    TICKETS {
        int id PK
        varchar ticket_number UK
        int customer_id FK
        int assigned_agent_id FK
        int category_id FK
        varchar subject
        varchar description
        varchar status
        varchar priority
        timestamp sla_response_deadline
        timestamp sla_deadline
        timestamp first_response_at
        timestamp resolved_at
        timestamp closed_at
        timestamp cancelled_at
        varchar sla_status
        timestamp created_at
        timestamp updated_at
    }

    COMMENTS {
        int id PK
        int ticket_id FK
        int user_id FK
        varchar content
        timestamp created_at
        timestamp updated_at
    }

    ATTACHMENTS {
        int id PK
        int ticket_id FK
        int uploaded_by FK
        varchar file_name
        varchar file_path UK
        int file_size
        varchar content_type
        timestamp created_at
    }

    NOTIFICATIONS {
        int id PK
        int user_id FK
        int ticket_id FK
        varchar event_type
        varchar message
        boolean is_read
        timestamp created_at
    }

    SLAS {
        int id PK
        varchar priority UK
        int response_time_hours
        int resolution_time_hours
        timestamp created_at
        timestamp updated_at
    }

    AUDIT_LOGS {
        int id PK
        int user_id FK
        varchar action
        varchar entity_type
        int entity_id
        json previous_value
        json new_value
        varchar description
        timestamp created_at
    }
```

## Relationships

- One `USER` may own one `CUSTOMER` profile. The customer `user_id` is unique.
- One `CUSTOMER` can create many `TICKETS`.
- A `TICKET` may be assigned to one support-agent `USER`; inactive agents cannot receive assignments.
- One `CATEGORY` can classify many tickets.
- One ticket can contain many comments and attachments.
- Comments and attachments record the user who created or uploaded them.
- Notifications belong to a user and may reference a ticket.
- Audit logs record the actor, action, entity type, entity ID, previous value, new value, and description.
- `slas` is a priority-keyed policy table. Tickets store deadline snapshots, so later policy changes do not rewrite existing ticket deadlines.

## Constraints and indexes

- `users.email` is unique and indexed.
- `customers.user_id` is unique.
- `categories.name` is unique and indexed.
- `tickets.ticket_number` is unique and indexed.
- `attachments.file_path` is unique.
- SLA policy priority is unique.
- Common ticket indexes cover status/priority, assigned-agent/status, and customer/created-at queries.
- Customer status, company, user role/active state, and notification user ID are indexed.
- Audit logs are indexed by user, entity type, and entity ID.

## Delete behavior

- Deleting a user cascades to the customer profile, comments, attachments, and notifications where configured.
- Ticket-to-customer and ticket-to-category references use `RESTRICT` to preserve support history and prevent deleting referenced records.
- Assigned-agent deletion uses `SET NULL`, preserving the ticket as unassigned.
- Comments and attachments cascade from their ticket.
- Notification ticket references use `SET NULL` so notifications can remain after ticket removal.
- Customer deletion through the API is implemented as deactivation, preserving the customer and ticket history.
- Ticket cancellation changes status to `cancelled`; it does not remove the ticket.
