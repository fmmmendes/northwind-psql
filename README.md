# Northwind Traders — PostgreSQL

Standalone Dockerized PostgreSQL project hosting the classic **Northwind** sample database
(all objects created and fully populated on first container start).

## Source & provenance

1. **Northwind Traders sample database** — shipped with **Microsoft Access 2010**, a tutorial
   schema for a small business managing customers, orders, inventory, purchasing, suppliers,
   shipping and employees.
2. **[dalers/mywind](https://github.com/dalers/mywind)**
   ([README](https://github.com/dalers/mywind/blob/master/README.md)) — the popular open-source
   port of the Access Northwind sample (structure `northwind.sql` + data `northwind-data.sql`,
   using `lower_case_with_underscore` identifiers).
3. This repository — the PostgreSQL version of those scripts, shipped as `initdb/01-schema.sql`
   (schema `northwind`, 20 tables) + `initdb/02-data.sql` (full seed data).

## Contents

| File | Description |
|------|-------------|
| `docker-compose.yml` | Docker Compose service definitions (PostgreSQL 17 + pgAdmin 4) |
| `initdb/01-schema.sql` | Creates the `northwind` schema, 20 tables, indexes and FK constraints |
| `initdb/02-data.sql` | Inserts the full Northwind dataset into every table |
| `.env.example` | Template for overriding connection settings |
| `pgadmin/servers.json` | Pre-registers the Northwind server in pgAdmin |

## Quickstart

```bash
docker compose up -d
```

On the **first** startup only, the official `postgres` image automatically:

1. Creates the database named by `POSTGRES_DB` (default `northwind`),
2. Runs every script in `./initdb/` in alphabetical order against it.

No manual `psql` steps are needed — when the container is healthy, all tables exist and are populated.

## Connection details (defaults)

| Setting | Value |
|---------|-------|
| Host    | `localhost` |
| Port    | `5432` |
| Database| `northwind` |
| User    | `postgres` |
| Password| `postgres` |

```bash
# psql inside the container
docker exec -it northwind-postgres psql -U postgres -d northwind

# psql from the host
psql -h localhost -p 5432 -U postgres -d northwind

# any client (URI)
postgresql://postgres:postgres@localhost:5432/northwind
```

Override any default via environment variables or a `.env` file next to `docker-compose.yml`
(see `.env.example`): `POSTGRES_PORT`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`,
`PGADMIN_PORT`, `PGADMIN_DEFAULT_EMAIL`, `PGADMIN_DEFAULT_PASSWORD`.

## pgAdmin

pgAdmin 4 (pinned to `dpage/pgadmin4:9.19`) ships in the same compose project and is
available right after `docker compose up -d` (it waits for the `db` healthcheck).

| Setting | Value (defaults) |
|---------|------------------|
| URL     | `http://localhost:5050` |
| Login   | `admin@northwind.com` |
| Password| `postgres` |

The `Northwind (docker)` server entry (`db:5432`, db `northwind`, user `postgres`) is
pre-registered through `pgadmin/servers.json`; on first connect pgAdmin prompts for the
database password (default `postgres`).

## Database objects

Schema `northwind` — 20 tables:

`customers`, `employees`, `employee_privileges`, `privileges`,
`orders`, `order_details`, `order_details_status`, `orders_status`, `orders_tax_status`,
`invoices`, `products`, `suppliers`, `purchase_orders`, `purchase_order_details`,
`purchase_order_status`, `inventory_transactions`, `inventory_transaction_types`,
`shippers`, `sales_reports`, `strings`

Foreign keys link orders → customers/employees/shippers, order & purchase-order details → products,
and purchase details → inventory transactions, with search indexes created on frequently
queried columns (city, company, names, postal codes, FKs).

## Data model

Entity-relationship overview of the `northwind` schema — all 20 tables with attributes,
primary keys (PK), foreign keys (FK) and cardinality (generated from
`initdb/01-schema.sql`; `||` = mandatory, `o|` = optional:
e.g. `inventory_transactions.product_id` is `NOT NULL`, `orders.customer_id` is nullable):

```mermaid
erDiagram
%% Data model of schema "northwind" (from initdb/01-schema.sql)
    CUSTOMERS {
        INTEGER id PK
        VARCHAR company
        VARCHAR last_name
        VARCHAR first_name
        VARCHAR email_address
        VARCHAR job_title
        VARCHAR business_phone
        VARCHAR home_phone
        VARCHAR mobile_phone
        VARCHAR fax_number
        TEXT address
        VARCHAR city
        VARCHAR state_province
        VARCHAR zip_postal_code
        VARCHAR country_region
        TEXT web_page
        TEXT notes
        BYTEA attachments
    }
    EMPLOYEE_PRIVILEGES {
        INT employee_id PK
        INT privilege_id PK
    }
    EMPLOYEES {
        INTEGER id PK
        VARCHAR company
        VARCHAR last_name
        VARCHAR first_name
        VARCHAR email_address
        VARCHAR job_title
        VARCHAR business_phone
        VARCHAR home_phone
        VARCHAR mobile_phone
        VARCHAR fax_number
        TEXT address
        VARCHAR city
        VARCHAR state_province
        VARCHAR zip_postal_code
        VARCHAR country_region
        TEXT web_page
        TEXT notes
        BYTEA attachments
    }
    INVENTORY_TRANSACTION_TYPES {
        SMALLINT id PK
        VARCHAR type_name "NOT NULL"
    }
    INVENTORY_TRANSACTIONS {
        INTEGER id PK
        SMALLINT transaction_type FK "NOT NULL"
        TIMESTAMP transaction_created_date
        TIMESTAMP transaction_modified_date
        INT product_id FK "NOT NULL"
        INT quantity "NOT NULL"
        INT purchase_order_id FK
        INT customer_order_id FK
        VARCHAR comments
    }
    INVOICES {
        INTEGER id PK
        INT order_id FK
        TIMESTAMP invoice_date
        TIMESTAMP due_date
        NUMERIC tax
        NUMERIC shipping
        NUMERIC amount_due
    }
    ORDER_DETAILS {
        INTEGER id PK
        INT order_id FK "NOT NULL"
        INT product_id FK
        NUMERIC quantity "NOT NULL"
        NUMERIC unit_price
        DOUBLE_PRECISION discount "NOT NULL"
        INT status_id FK
        TIMESTAMP date_allocated
        INT purchase_order_id
        INT inventory_id
    }
    ORDER_DETAILS_STATUS {
        INT id PK
        VARCHAR status_name "NOT NULL"
    }
    ORDERS {
        INTEGER id PK
        INT employee_id FK
        INT customer_id FK
        TIMESTAMP order_date
        TIMESTAMP shipped_date
        INT shipper_id FK
        VARCHAR ship_name
        TEXT ship_address
        VARCHAR ship_city
        VARCHAR ship_state_province
        VARCHAR ship_zip_postal_code
        VARCHAR ship_country_region
        NUMERIC shipping_fee
        NUMERIC taxes
        VARCHAR payment_type
        TIMESTAMP paid_date
        TEXT notes
        DOUBLE_PRECISION tax_rate
        SMALLINT tax_status_id FK
        SMALLINT status_id FK
    }
    ORDERS_STATUS {
        SMALLINT id PK
        VARCHAR status_name "NOT NULL"
    }
    ORDERS_TAX_STATUS {
        SMALLINT id PK
        VARCHAR tax_status_name "NOT NULL"
    }
    PRIVILEGES {
        INTEGER id PK
        VARCHAR privilege_name
    }
    PRODUCTS {
        TEXT supplier_ids
        INTEGER id PK
        VARCHAR product_code
        VARCHAR product_name
        TEXT description
        NUMERIC standard_cost
        NUMERIC list_price "NOT NULL"
        INT reorder_level
        INT target_level
        VARCHAR quantity_per_unit
        BOOLEAN discontinued "NOT NULL"
        INT minimum_reorder_quantity
        VARCHAR category
        BYTEA attachments
    }
    PURCHASE_ORDER_DETAILS {
        INTEGER id PK
        INT purchase_order_id FK "NOT NULL"
        INT product_id FK
        NUMERIC quantity "NOT NULL"
        NUMERIC unit_cost "NOT NULL"
        TIMESTAMP date_received
        BOOLEAN posted_to_inventory "NOT NULL"
        INT inventory_id FK
    }
    PURCHASE_ORDER_STATUS {
        INT id PK
        VARCHAR status
    }
    PURCHASE_ORDERS {
        INTEGER id PK
        INT supplier_id FK
        INT created_by FK
        TIMESTAMP submitted_date
        TIMESTAMP creation_date
        INT status_id FK
        TIMESTAMP expected_date
        NUMERIC shipping_fee "NOT NULL"
        NUMERIC taxes "NOT NULL"
        TIMESTAMP payment_date
        NUMERIC payment_amount
        VARCHAR payment_method
        TEXT notes
        INT approved_by
        TIMESTAMP approved_date
        INT submitted_by
    }
    SALES_REPORTS {
        VARCHAR group_by "NOT NULL"
        VARCHAR display
        VARCHAR title
        TEXT filter_row_source
        BOOLEAN default "NOT NULL"
    }
    SHIPPERS {
        INTEGER id PK
        VARCHAR company
        VARCHAR last_name
        VARCHAR first_name
        VARCHAR email_address
        VARCHAR job_title
        VARCHAR business_phone
        VARCHAR home_phone
        VARCHAR mobile_phone
        VARCHAR fax_number
        TEXT address
        VARCHAR city
        VARCHAR state_province
        VARCHAR zip_postal_code
        VARCHAR country_region
        TEXT web_page
        TEXT notes
        BYTEA attachments
    }
    STRINGS {
        INTEGER string_id PK
        VARCHAR string_data
    }
    SUPPLIERS {
        INTEGER id PK
        VARCHAR company
        VARCHAR last_name
        VARCHAR first_name
        VARCHAR email_address
        VARCHAR job_title
        VARCHAR business_phone
        VARCHAR home_phone
        VARCHAR mobile_phone
        VARCHAR fax_number
        TEXT address
        VARCHAR city
        VARCHAR state_province
        VARCHAR zip_postal_code
        VARCHAR country_region
        TEXT web_page
        TEXT notes
        BYTEA attachments
    }
    EMPLOYEE_PRIVILEGES }o--|| EMPLOYEES : "employee_privileges.employee_id"
    EMPLOYEE_PRIVILEGES }o--|| PRIVILEGES : "employee_privileges.privilege_id"
    INVENTORY_TRANSACTIONS }o--o| ORDERS : "inventory_transactions.customer_order_id"
    INVENTORY_TRANSACTIONS }o--|| PRODUCTS : "inventory_transactions.product_id"
    INVENTORY_TRANSACTIONS }o--o| PURCHASE_ORDERS : "inventory_transactions.purchase_order_id"
    INVENTORY_TRANSACTIONS }o--|| INVENTORY_TRANSACTION_TYPES : "inventory_transactions.transaction_type"
    INVOICES }o--o| ORDERS : "invoices.order_id"
    ORDER_DETAILS }o--|| ORDERS : "order_details.order_id"
    ORDER_DETAILS }o--o| PRODUCTS : "order_details.product_id"
    ORDER_DETAILS }o--o| ORDER_DETAILS_STATUS : "order_details.status_id"
    ORDERS }o--o| CUSTOMERS : "orders.customer_id"
    ORDERS }o--o| EMPLOYEES : "orders.employee_id"
    ORDERS }o--o| SHIPPERS : "orders.shipper_id"
    ORDERS }o--o| ORDERS_STATUS : "orders.status_id"
    ORDERS }o--o| ORDERS_TAX_STATUS : "orders.tax_status_id"
    PURCHASE_ORDER_DETAILS }o--o| INVENTORY_TRANSACTIONS : "purchase_order_details.inventory_id"
    PURCHASE_ORDER_DETAILS }o--o| PRODUCTS : "purchase_order_details.product_id"
    PURCHASE_ORDER_DETAILS }o--|| PURCHASE_ORDERS : "purchase_order_details.purchase_order_id"
    PURCHASE_ORDERS }o--o| EMPLOYEES : "purchase_orders.created_by"
    PURCHASE_ORDERS }o--o| PURCHASE_ORDER_STATUS : "purchase_orders.status_id"
    PURCHASE_ORDERS }o--o| SUPPLIERS : "purchase_orders.supplier_id"
```

Useful row-count check:

```sql
SELECT relname AS table, n_live_tup AS rows
FROM pg_stat_user_tables
WHERE schemaname = 'northwind'
ORDER BY relname;
```

## Operations

```bash
docker compose up -d      # start
docker compose ps         # status / health
docker compose logs -f db # logs
docker compose down       # stop (data volume is preserved)

docker compose down -v    # stop and DESTROY data
docker compose up -d      # next start re-runs initdb/ scripts from scratch
```

The init scripts run only when the data volume is empty. `down` keeps the volume; to reset
the database and re-run initialization, `down -v` first.
