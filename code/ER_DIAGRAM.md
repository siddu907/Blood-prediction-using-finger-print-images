# Entity-Relationship Diagram

This diagram documents the current SQLAlchemy relational schema. `PK` is a primary key, `FK` is a database foreign key, and `UK` is a unique constraint. Mermaid renders the diagram in GitHub and compatible Markdown viewers.

```mermaid
erDiagram
    ROLES {
        string name PK
        string description
    }

    USERS {
        int id PK
        string email UK
        string full_name
        string password_hash
        string role FK
        boolean is_active
        int auth_token_version
        datetime created_at
    }

    REFRESH_TOKENS {
        int id PK
        int user_id FK
        string jti UK
        datetime expires_at
        datetime revoked_at
    }

    WALLETS {
        int id PK
        int owner_id FK
        string currency
        decimal available_balance
        decimal pending_balance
        string status
        datetime last_frozen_at
        datetime created_at
        datetime updated_at
    }

    DEPOSITS {
        int id PK
        int wallet_id FK
        int user_id FK
        decimal amount
        string currency
        string status
        string reference UK
        datetime created_at
        datetime completed_at
    }

    WITHDRAWALS {
        int id PK
        int wallet_id FK
        int user_id FK
        decimal amount
        string currency
        string status
        string reference UK
        datetime created_at
        datetime completed_at
    }

    TRANSFERS {
        int id PK
        int sender_wallet_id FK
        int receiver_wallet_id FK
        int user_id FK
        decimal amount
        string currency
        string status
        string reference UK
        string risk_status
        datetime created_at
        datetime completed_at
        datetime reversed_at
    }

    TRANSACTION_LEDGER {
        int id PK
        int wallet_id FK
        string transaction_type
        decimal amount
        decimal balance_before
        decimal balance_after
        string reference_entity
        int reference_id
        string description
        int created_by FK
        json metadata
        datetime created_at
    }

    REFUNDS {
        int id PK
        int ledger_entry_id FK
        int wallet_id FK
        int user_id FK
        decimal amount
        string status
        string reason
        datetime created_at
        int reviewed_by FK
        datetime reviewed_at
        datetime completed_at
    }

    SUSPICIOUS_TRANSACTIONS {
        int id PK
        int transfer_id FK, UK
        int user_id FK
        int wallet_id FK
        string operation_type
        int operation_id
        decimal amount
        string status
        string reason
        int reviewed_by FK
        datetime reviewed_at
        datetime created_at
    }

    IDEMPOTENCY_KEYS {
        int id PK
        int user_id FK
        string key
        string request_hash
        string entity_type
        int entity_id
        datetime created_at
    }

    NOTIFICATIONS {
        int id PK
        int user_id FK
        string event
        string message
        boolean is_read
        datetime created_at
    }

    AUDIT_LOGS {
        int id PK
        int user_id FK
        string action
        string entity
        int entity_id
        string ip_address
        json metadata
        datetime created_at
    }

    WALLET_DAILY_SUMMARIES {
        int id PK
        int wallet_id FK
        date summary_date
        decimal available_balance
        decimal pending_balance
        datetime created_at
    }

    TRANSACTION_LIMITS {
        int id PK
        decimal max_transaction_amount
        decimal daily_transfer_limit
        decimal daily_withdrawal_limit
        int max_transactions_per_day
        decimal suspicious_amount_threshold
    }

    DAILY_LIMIT_RESETS {
        date reset_date PK
        datetime completed_at
        string strategy
    }

    FINANCIAL_IDENTIFIERS {
        int id PK
    }

    OUTBOX_EVENTS {
        int id PK
        string event_type
        int aggregate_id
        json payload
        datetime created_at
        datetime processed_at
    }

    ROLES ||--o{ USERS : assigns
    USERS ||--o{ REFRESH_TOKENS : authenticates
    USERS ||--o{ WALLETS : owns
    USERS ||--o{ DEPOSITS : requests
    USERS ||--o{ WITHDRAWALS : requests
    USERS ||--o{ TRANSFERS : initiates
    USERS ||--o{ REFUNDS : requests
    USERS ||--o{ NOTIFICATIONS : receives
    USERS o|--o{ AUDIT_LOGS : acts_in
    USERS o|--o{ TRANSACTION_LEDGER : creates
    USERS o|--o{ REFUNDS : reviews
    USERS o|--o{ SUSPICIOUS_TRANSACTIONS : reviews
    USERS o|--o{ SUSPICIOUS_TRANSACTIONS : associated_user
    WALLETS ||--o{ DEPOSITS : credited_by
    WALLETS ||--o{ WITHDRAWALS : funds
    WALLETS ||--o{ TRANSFERS : sender
    WALLETS ||--o{ TRANSFERS : receiver
    WALLETS ||--o{ TRANSACTION_LEDGER : records
    WALLETS ||--o{ REFUNDS : sender_wallet
    WALLETS o|--o{ SUSPICIOUS_TRANSACTIONS : flagged_wallet
    WALLETS ||--o{ WALLET_DAILY_SUMMARIES : summarized_by
    TRANSACTION_LEDGER ||--o{ REFUNDS : refund_source
    TRANSFERS o|--o| SUSPICIOUS_TRANSACTIONS : transfer_review
```

## Relationship and integrity notes

- Each user has one role and may own multiple wallets. The database enforces one wallet per owner and currency.
- Idempotency keys are unique per user/key pair. Daily wallet summaries are unique per wallet/date pair.
- Deposits and withdrawals reference both the wallet affected and the user who initiated the operation. Transfers reference the initiating user and two wallets, with separate sender and receiver foreign keys.
- Every ledger row belongs to one wallet. `created_by` is nullable for system-generated entries. Ledger records are append-only; corrections are represented by compensating entries.
- A refund references the sender's original `TRANSFER_DEBIT` ledger row, the sender wallet, and the requesting user. `reviewed_by` is nullable until a staff decision is recorded.
- A suspicious transaction may refer to a transfer through `transfer_id`, or represent a generic financial operation through the `operation_type` / `operation_id` pair. A unique constraint prevents more than one case per operation.
- Several references are intentionally polymorphic and therefore are **not** database foreign keys: `TRANSACTION_LEDGER.reference_entity` / `reference_id`, `AUDIT_LOGS.entity` / `entity_id`, `IDEMPOTENCY_KEYS.entity_type` / `entity_id`, `SUSPICIOUS_TRANSACTIONS.operation_type` / `operation_id`, and `OUTBOX_EVENTS.aggregate_id`. Their target is identified by the accompanying entity/type field.
- Financial-operation IDs and ledger IDs are allocated from `FINANCIAL_IDENTIFIERS`; the allocator relationship is implemented in application/database logic and is not expressed as a foreign key.
- `TRANSACTION_LIMITS` stores system configuration and `DAILY_LIMIT_RESETS` stores idempotent scheduler checkpoints; neither has a direct relational foreign key to user or wallet data.
- Wallet balances are `NUMERIC(18,2)` and protected by nonnegative database checks. Financial amounts use decimal arithmetic. Enum-like values are shown as `string` in the diagram; their allowed values are enforced by SQLAlchemy enum columns and API schemas.

## Source of truth

The diagram reflects the ORM metadata loaded by `app.models`. If schema migrations or model definitions change, update this document to match and verify the resulting database schema with Alembic.
