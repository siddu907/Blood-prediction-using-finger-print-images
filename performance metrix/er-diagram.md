# Entity-Relationship Diagram

This diagram reflects the SQLAlchemy models and database foreign keys currently
implemented in the platform. Mermaid-capable Markdown viewers render the diagram
below.

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
        int token_version
        boolean is_active
        datetime created_at
    }

    WAREHOUSE_USERS {
        int warehouse_id PK, FK
        int user_id PK, FK
    }

    CATEGORIES {
        int id PK
        string name UK
        string description
        boolean is_active
        datetime created_at
    }

    PRODUCTS {
        int id PK
        string sku UK
        string name
        string description
        int category_id FK
        decimal price
        decimal cost_price
        boolean is_active
        datetime created_at
        datetime updated_at
    }

    WAREHOUSES {
        int id PK
        string name UK
        string location
        int capacity
        boolean is_active
        datetime created_at
        datetime updated_at
    }

    INVENTORY {
        int id PK
        int product_id FK
        int warehouse_id FK
        int available_quantity
        int reserved_quantity
        int damaged_quantity
        int reorder_level
        datetime updated_at
    }

    INVENTORY_TRANSACTIONS {
        int id PK
        int product_id FK
        int warehouse_id FK
        string transaction_type
        int quantity
        string reference_entity
        string reference_id
        int user_id FK
        json metadata
        datetime created_at
    }

    ORDERS {
        int id PK
        string order_number UK
        int customer_id FK
        int warehouse_id FK
        decimal total_amount
        string status
        string payment_status
        datetime expected_delivery_at
        datetime created_at
        datetime updated_at
    }

    ORDER_ITEMS {
        int id PK
        int order_id FK
        int product_id FK
        int quantity
        decimal unit_price
        decimal discount
        decimal tax
        decimal tax_rate
        decimal total
    }

    ORDER_STATUS_HISTORY {
        int id PK
        int order_id FK
        string from_status
        string to_status
        int user_id FK
        datetime created_at
    }

    ORDER_ASSIGNMENTS {
        int id PK
        int order_id FK
        int agent_id FK
        int assigned_by_id FK
        datetime created_at
    }

    RETURNS {
        int id PK
        int order_id FK
        int customer_id FK
        string status
        string reason
        datetime created_at
    }

    RETURN_ITEMS {
        int id PK
        int return_id FK
        int order_item_id FK
        int quantity
    }

    NOTIFICATIONS {
        int id PK
        int user_id FK
        string event_type
        string message
        string entity_type
        string entity_id
        boolean is_read
        datetime created_at
    }

    AUDIT_LOGS {
        int id PK
        int user_id FK
        string action
        string entity
        string entity_id
        string ip_address
        json metadata
        datetime created_at
    }

    REFRESH_TOKENS {
        int id PK
        int user_id FK
        string token_hash UK
        datetime expires_at
        datetime revoked_at
        datetime created_at
    }

    IDEMPOTENCY_KEYS {
        int id PK
        int user_id FK
        string endpoint
        string key
        string request_hash
        int resource_id
        json response_json
        datetime created_at
    }

    TAX_CONFIGURATION {
        int id PK
        decimal rate_percent
        datetime updated_at
    }

    ROLES ||--o{ USERS : grants
    USERS ||--o{ WAREHOUSE_USERS : assigned
    WAREHOUSES ||--o{ WAREHOUSE_USERS : has_members
    CATEGORIES o|--o{ PRODUCTS : classifies
    PRODUCTS ||--o{ INVENTORY : stocked_as
    WAREHOUSES ||--o{ INVENTORY : stores
    PRODUCTS ||--o{ INVENTORY_TRANSACTIONS : movement_for
    WAREHOUSES ||--o{ INVENTORY_TRANSACTIONS : movement_at
    USERS o|--o{ INVENTORY_TRANSACTIONS : performs
    USERS ||--o{ ORDERS : places
    WAREHOUSES ||--o{ ORDERS : fulfills
    ORDERS ||--|{ ORDER_ITEMS : contains
    PRODUCTS ||--o{ ORDER_ITEMS : ordered_as
    ORDERS ||--o{ ORDER_STATUS_HISTORY : records
    USERS o|--o{ ORDER_STATUS_HISTORY : changes
    ORDERS ||--o{ ORDER_ASSIGNMENTS : assigned_over_time
    USERS ||--o{ ORDER_ASSIGNMENTS : assigned_agent
    USERS ||--o{ ORDER_ASSIGNMENTS : assigned_by
    ORDERS ||--o{ RETURNS : may_have
    USERS ||--o{ RETURNS : requests
    RETURNS ||--|{ RETURN_ITEMS : includes
    ORDER_ITEMS ||--o{ RETURN_ITEMS : returned_line
    USERS ||--o{ NOTIFICATIONS : receives
    USERS o|--o{ AUDIT_LOGS : performs
    USERS ||--o{ REFRESH_TOKENS : owns
    USERS ||--o{ IDEMPOTENCY_KEYS : owns
```

## Relationship and integrity notes

- `warehouse_users` is the many-to-many join table between users and warehouses.
  Its `(warehouse_id, user_id)` composite primary key prevents duplicate
  membership rows.
- `inventory` stores one balance row per product and warehouse. The
  `(product_id, warehouse_id)` pair is unique, and each quantity field must be
  non-negative.
- `inventory_transactions` is the inventory movement ledger. `reference_entity`
  and `reference_id` are descriptive polymorphic reference fields, not foreign
  keys. The API does not expose ledger mutation routes.
- `order_items` preserve unit price, discount, tax rate, tax, and total as
  snapshots. An order must contain one or more items when created through the
  API.
- `order_assignments` preserves assignment history; the latest assignment row
  identifies the currently assigned agent.
- `return_items` link requested return quantities to purchased `order_items`.
  The `(return_id, order_item_id)` pair is unique.
- `notifications.entity_type`/`entity_id` and
  `audit_logs.entity`/`entity_id` are polymorphic references and are not
  database foreign keys.
- Payment is currently a status on `orders` (`payment_status`); there is no
  separate payment table or external payment-provider entity.
- `tax_configuration` is a singleton configuration table constrained to `id=1`.
- Foreign-key deletion behavior is defined in the SQLAlchemy models. For
  example, deleting a user cascades their refresh tokens, notifications, and
  idempotency keys; referenced business records generally restrict deletion.

## Entity coverage

The diagram includes identity and role data, warehouse membership, catalog,
inventory balances and ledger transactions, orders and lifecycle history,
fulfillment assignments, return requests and lines, notifications, audit logs,
refresh tokens, idempotency keys, and tax configuration.
