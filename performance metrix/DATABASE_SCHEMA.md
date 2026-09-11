# Database Schema

The Task Management System uses six relational tables. Authentication tokens are not stored in the database; JWT access tokens are stateless.

## Entity-relationship diagram

```mermaid
erDiagram
    USERS ||--o{ TASKS : creates
    USERS o|--o{ TASKS : receives
    USERS ||--o{ COMMENTS : writes
    TASKS ||--o{ COMMENTS : contains
    TASKS ||--o{ ATTACHMENTS : contains
    USERS ||--o{ NOTIFICATIONS : receives
    TASKS o|--o{ NOTIFICATIONS : triggers
    USERS o|--o{ AUDIT_LOGS : performs

    USERS {
        int id PK
        varchar name
        varchar email UK
        varchar phone nullable
        varchar password_hash
        varchar role
        boolean is_active
        datetime created_at
    }

    TASKS {
        int id PK
        varchar title
        text description nullable
        int assigned_to_id FK nullable
        int created_by_id FK
        varchar priority
        varchar status
        datetime due_date nullable
        datetime created_at
        datetime updated_at
        datetime completed_at nullable
    }

    COMMENTS {
        int id PK
        text content
        int task_id FK
        int user_id FK
        datetime created_at
        datetime updated_at
    }

    ATTACHMENTS {
        int id PK
        int task_id FK
        varchar file_name
        varchar file_path
        varchar file_type
        int file_size
        datetime uploaded_at
    }

    NOTIFICATIONS {
        int id PK
        int user_id FK
        int task_id FK nullable
        text message
        varchar notification_type
        boolean is_read
        datetime created_at
    }

    AUDIT_LOGS {
        int id PK
        int user_id FK nullable
        varchar action
        varchar entity_type
        int entity_id nullable
        datetime timestamp
    }
```

## Tables and relationships

### `users`

Stores registered users and administrators.

- `email` is unique.
- `role` contains `user` or `admin`.
- `is_active` controls whether the user can authenticate or receive tasks.
- One user can create many tasks through `tasks.created_by_id`.
- One user can receive many tasks through `tasks.assigned_to_id`.
- One user can write many comments, receive many notifications, and create many audit-log entries.

### `tasks`

Stores task details and workflow state.

- `created_by_id` is required and identifies the task creator.
- `assigned_to_id` is optional and identifies the current assignee.
- `priority` values are `low`, `medium`, `high`, and `critical`.
- `status` values are `todo`, `in_progress`, `completed`, and `cancelled`.
- A task can have many comments, attachments, and notifications.

### `comments`

Stores comments made by users on tasks.

- Each comment belongs to one task.
- Each comment belongs to one user.
- Comment content is limited by the API to 5,000 characters.

### `attachments`

Stores metadata for files uploaded to tasks.

- Each attachment belongs to one task.
- File contents are stored in the configured upload directory.
- The database stores the original name, storage path, type, and size.

### `notifications`

Stores user-specific task notifications.

- Each notification belongs to one user.
- A notification can optionally reference a task.
- Notifications are generated for assignment, reassignment, status changes, comments, and completion.
- `is_read` tracks whether the user has read the notification.

### `audit_logs`

Stores administrative and user action history.

- `user_id` is nullable so an audit record can exist without a linked user.
- `entity_type` and `entity_id` identify the affected resource.
- Audit records are available through admin-only endpoints.

## Foreign-key summary

| Foreign key | References | Meaning |
|---|---|---|
| `tasks.created_by_id` | `users.id` | User who created the task |
| `tasks.assigned_to_id` | `users.id` | Current task assignee; nullable |
| `comments.task_id` | `tasks.id` | Comment's task |
| `comments.user_id` | `users.id` | Comment author |
| `attachments.task_id` | `tasks.id` | Attachment's task |
| `notifications.user_id` | `users.id` | Notification recipient |
| `notifications.task_id` | `tasks.id` | Related task; nullable |
| `audit_logs.user_id` | `users.id` | Actor; nullable |

## Migration source

Alembic migrations are the source of truth for deployed database schemas. Apply them with:

```powershell
python -m alembic upgrade head
```
