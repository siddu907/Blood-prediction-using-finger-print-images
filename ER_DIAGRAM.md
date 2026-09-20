# ER Diagram

```mermaid
erDiagram
    ROLE ||--o{ USER : has
    USER ||--o{ TICKET : creates
    USER ||--o{ TICKET_ASSIGNMENT : assigned_to
    USER ||--o{ TICKET_COMMENT : writes
    USER ||--o{ TICKET_ATTACHMENT : uploads
    USER ||--o{ NOTIFICATION : receives
    USER ||--o{ AUDIT_LOG : performs
    USER ||--o{ REFRESH_TOKEN : owns

    TICKET_CATEGORY ||--o{ TICKET : categorizes
    TICKET ||--o{ TICKET_COMMENT : contains
    TICKET ||--o{ TICKET_ATTACHMENT : contains
    TICKET ||--o{ NOTIFICATION : triggers
    TICKET ||--o{ TICKET_STATUS_HISTORY : updates

    ROLE {
        int id PK
        string name
    }

    USER {
        int id PK
        string name
        string email
        string hashed_password
        int role_id FK
        bool is_active
        datetime created_at
        datetime updated_at
    }

    TICKET_CATEGORY {
        int id PK
        string name
        string description
        bool is_active
        datetime created_at
        datetime updated_at
    }

    TICKET {
        int id PK
        string ticket_number
        string title
        string description
        int customer_id FK
        int agent_id FK
        int category_id FK
        string priority
        string status
        datetime due_date
        datetime created_at
        datetime updated_at
    }

    TICKET_COMMENT {
        int id PK
        int ticket_id FK
        int user_id FK
        string content
        datetime created_at
        datetime updated_at
    }

    TICKET_ATTACHMENT {
        int id PK
        int ticket_id FK
        int user_id FK
        string file_name
        string file_path
        datetime uploaded_at
    }

    NOTIFICATION {
        int id PK
        int user_id FK
        int ticket_id FK
        string notification_type
        string title
        string message
        bool is_read
        datetime created_at
    }

    AUDIT_LOG {
        int id PK
        int user_id FK
        string action
        string entity_type
        int entity_id
        json payload
        string ip_address
        datetime created_at
    }

    REFRESH_TOKEN {
        int id PK
        int user_id FK
        string token
        bool is_revoked
        datetime expires_at
        datetime created_at
    }

    TICKET_STATUS_HISTORY {
        int id PK
        int ticket_id FK
        int changed_by FK
        string old_status
        string new_status
        datetime created_at
    }
```

## Entity description

- Users belong to a role and can act as customers, support agents, or admins.
- A ticket belongs to one customer, may be assigned to one agent, and belongs to one category.
- A ticket can have many comments, attachments, notifications, and status history entries.
- Notifications are sent to users related to ticket activity and SLA breaches.
- Audit logs track important actions for security and traceability.
- Refresh tokens are stored per user so sessions can be revoked safely.
