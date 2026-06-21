# MySQL E-Commerce Database Project

A learning-grade **relational database schema** for a realistic e-commerce
application, built entirely in MySQL. The project models the seven core tables
that almost every online store needs — users, products, carts, orders,
inventory, warehouses, and wishlists — with proper primary keys, foreign keys,
generated columns, ENUMs, indexes, and seed data.

It is delivered as a set of **MySQL Shell Notebooks** (`.mysql-notebook` files),
one per table, so you can open each one in **MySQL Workbench / MySQL Shell for
VS Code** and execute the cells block-by-block to see the schema come to life.

---

## Table of contents

- [What is in this project](#what-is-in-this-project)
- [Prerequisites](#prerequisites)
- [Setup — from zero to a running database](#setup--from-zero-to-a-running-database)
- [How to run the notebooks](#how-to-run-the-notebooks)
- [Recommended load order](#recommended-load-order)
- [The schema in detail](#the-schema-in-detail)
- [Entity relationships](#entity-relationships)
- [Sample queries you can try](#sample-queries-you-can-try)
- [Resetting the database](#resetting-the-database)
- [Troubleshooting](#troubleshooting)

---

## What is in this project

```
MySql Ecommerce Project/
├── UsersTable.mysql-notebook        # users
├── ProductTable.mysql-notebook      # products (FK -> users as vendors)
├── CartTable.mysql-notebook         # carts + cart_items
├── OrderTable.mysql-notebook        # orders + order_items
├── WarehouseTable.mysql-notebook    # warehouses (placeholder for warehouse DDL)
├── InventoryTable.mysql-notebook    # inventory (M:N bridge: products x warehouses)
└── WishlistTable.mysql-notebook     # wishlists (M:N bridge: users x products)
```

Every notebook contains, in order:

1. A `CREATE TABLE` statement for the table(s) it owns
2. `DESC` / `SHOW TABLES` checks so you can verify the schema
3. A `START TRANSACTION;` block followed by bulk `INSERT` statements with
   realistic seed data (hundreds to thousands of rows)
4. Sanity-check `SELECT` queries

Each table uses `CHAR(36)` UUIDs as primary keys, soft-delete columns where
appropriate, ENUMs for fixed-value status fields, and tight foreign-key
constraints with explicit `ON DELETE` behavior.

---

## Prerequisites

| Tool | Version | Why |
|------|---------|-----|
| **MySQL Server** | 8.0+ | Schema uses `GENERATED ALWAYS AS ... STORED`, `CHECK` constraints, `JSON` columns, and `DEFAULT (UUID())` — all of which require 8.0+ |
| **MySQL Shell** or **MySQL Workbench** | latest | Required to open `.mysql-notebook` files. Plain `mysql` CLI cannot open notebooks. |
| (optional) **VS Code + MySQL Shell for VS Code** extension | latest | Lets you open the notebooks directly inside VS Code |

> Why MySQL 8 specifically: tables use `JSON` (`orders.shipping_address`),
> stored generated columns (`cart_items.line_total`, `inventory.quantity_available`,
> `carts.is_active_cart`), and `CHECK` constraints — none of which work
> reliably on MySQL 5.7 or earlier.

---

## Setup — from zero to a running database

### 1. Install MySQL 8

- **Windows:** download the MySQL Installer from
  <https://dev.mysql.com/downloads/installer/> and pick "Server + Workbench +
  Shell". Set a root password you will remember.
- **macOS:** `brew install mysql` then `brew services start mysql`.
- **Linux:** `sudo apt install mysql-server` (Debian/Ubuntu) or equivalent.

Verify the server is up:

```bash
mysql --version
mysql -u root -p -e "SELECT VERSION();"
```

You should see `8.0.x` or `8.4.x`.

### 2. Create the database

The first notebook (`UsersTable.mysql-notebook`) does this for you with:

```sql
CREATE DATABASE Ecommerce;
USE ecommerce;
```

If you would rather pre-create it from the CLI:

```sql
CREATE DATABASE IF NOT EXISTS ecommerce
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_0900_ai_ci;
```

### 3. Open the project folder

In MySQL Workbench:
**File → Open SQL Script…** does not work for notebooks. Instead use
**File → Open MySQL Shell Notebook…** and pick a `.mysql-notebook` file.

In VS Code with the *MySQL Shell for VS Code* extension installed:
just open the project folder — the notebooks render as interactive cells.

### 4. Connect to your MySQL server

Inside Workbench / VS Code, create a connection:

- Host: `127.0.0.1`
- Port: `3306`
- User: `root` (or a dedicated user you created)
- Default schema: `ecommerce`

---

## How to run the notebooks

Each `.mysql-notebook` file is a sequence of SQL cells. Inside MySQL Shell /
Workbench:

- Click the first cell.
- Press **Ctrl + Enter** (or the ▶ button) to execute that cell.
- Move to the next cell and repeat.

You can also select **Run All** to execute every cell in the notebook in order.

---

## Recommended load order

Foreign keys force a strict order. Run the notebooks **in this sequence**, top
to bottom, the first time you set the database up:

| # | Notebook | Creates | Depends on |
|---|----------|---------|------------|
| 1 | `UsersTable.mysql-notebook` | `users` | — |
| 2 | `ProductTable.mysql-notebook` | `products` | `users` (vendor FK) |
| 3 | `WarehouseTable.mysql-notebook` | `warehouses` | — |
| 4 | `InventoryTable.mysql-notebook` | `inventory` | `products`, `warehouses` |
| 5 | `CartTable.mysql-notebook` | `carts`, `cart_items` | `users`, `products` |
| 6 | `OrderTable.mysql-notebook` | `orders`, `order_items` | `users`, `products`, `carts` |
| 7 | `WishlistTable.mysql-notebook` | `wishlists` | `users`, `products` |

The very last cell in `CartTable.mysql-notebook` adds an `ALTER TABLE` that
links `carts.converted_order_id` back to `orders.id`. Run it **after** the
orders notebook so the referenced table exists.

> The `WarehouseTable.mysql-notebook` is currently a placeholder (just
> `\about`). If you do not have a `warehouses` table yet, create a minimal one
> before running the inventory notebook — see
> [Troubleshooting](#troubleshooting).

---

## The schema in detail

### `users`

The identity table for everyone in the system — customers, admins, vendors,
and support staff.

| Column | Type | Notes |
|--------|------|-------|
| `id` | `CHAR(36)` PK | UUID, default `(UUID())` |
| `email`, `phone_number` | `VARCHAR` | Unique |
| `password_hash` | `VARCHAR(255)` | Bcrypt-shaped placeholder |
| `role` | `ENUM('customer','admin','vendor','support')` | Drives authorization |
| `status` | `ENUM('active','inactive','suspended','banned')` | Account lifecycle |
| `is_email_verified`, `is_phone_verified`, `two_factor_enabled` | `TINYINT(1)` | Booleans |
| `email_verification_token`, `password_reset_token`, `password_reset_expires` | — | Auth flow state |
| `loyalty_points`, `marketing_opt_in`, `preferred_locale`, `currency`, `avatar_url`, `date_of_birth` | — | Commerce / preferences |
| `created_at`, `updated_at`, `deleted_at` | `DATETIME` | Soft-delete pattern |

Indexes: unique on `email` and `phone_number`; secondary on `status`, `role`,
`deleted_at`.

### `products`

The product catalog. Each product can be owned by a vendor (a row in `users`
with `role = 'vendor'`).

Notable columns: `sku` and `slug` (both unique), `price`, `compare_at_price`
(the "was" price for sale displays), `cost_price` (margin tracking),
`stock_quantity` + `low_stock_threshold` (denormalized stock summary; the
real per-warehouse stock lives in `inventory`), `status` ENUM
(`draft/active/inactive/out_of_stock/archived`), and a soft-delete `deleted_at`.

FK: `vendor_id → users(id) ON DELETE SET NULL`.

### `carts` and `cart_items`

A shopping cart with line items.

- `carts.status` is `active / converted / abandoned / merged`.
- A user can only have **one active cart at a time**. This is enforced
  elegantly with a generated column:

  ```sql
  is_active_cart TINYINT GENERATED ALWAYS AS
      (CASE WHEN status = 'active' THEN 1 ELSE NULL END) STORED,
  UNIQUE KEY uq_user_active_cart (user_id, is_active_cart)
  ```

  Because non-active carts have `is_active_cart = NULL`, they are exempt
  from the unique constraint — only the active ones are constrained.

- `cart_items` has a unique `(cart_id, product_id)` pair, so re-adding the
  same product bumps `quantity` instead of duplicating rows.
- `cart_items.line_total` is a stored generated column = `unit_price * quantity`.

### `orders` and `order_items`

Orders are immutable snapshots — once placed, prices and addresses are
captured so changes to products or user addresses do not retroactively
rewrite history.

- `order_number` is a human-facing identifier (e.g. `ORD-2026-000123`)
  generated by your app.
- `status` ENUM covers the full lifecycle:
  `pending / confirmed / processing / shipped / delivered / cancelled / refunded / returned`.
- `payment_status` is tracked separately from order `status` so payment
  state (`authorized`, `paid`, `failed`, `refunded`, etc.) can evolve
  independently.
- Money is split into `subtotal / discount_amount / tax_amount /
  shipping_amount / total_amount`, each `DECIMAL(12,2)`, all guarded by a
  `CHECK` constraint that none can go negative.
- `shipping_address` is a `JSON` column — the full address is snapshotted
  into the order row.
- `order_items.line_total` is a stored generated column =
  `unit_price * quantity - discount_amount + tax_amount`.
- `order_items.product_id` uses `ON DELETE SET NULL` so the order history
  survives a product deletion. The `product_name` / `product_sku` snapshot
  columns keep the line readable even after the FK goes null.

### `warehouses`

Physical (or virtual) stock locations. The notebook ships as a placeholder —
add columns that match your business (name, code, city, country, address,
is_active, etc.). At minimum you need:

```sql
CREATE TABLE warehouses (
    id   CHAR(36) NOT NULL DEFAULT (UUID()),
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50)  NOT NULL UNIQUE,
    PRIMARY KEY (id)
);
```

…before `InventoryTable.mysql-notebook` will run.

### `inventory`

The many-to-many bridge between `products` and `warehouses` that **carries
the stock numbers**.

- One row per `(warehouse_id, product_id)` — enforced by a unique key.
- `quantity_on_hand` and `quantity_reserved` are the inputs.
- `quantity_available` is a stored generated column =
  `quantity_on_hand - quantity_reserved`.
- `reorder_point` and `reorder_quantity` drive automated restocking logic.
- `bin_location` is a free-form aisle/shelf code (e.g. `A-04-2`).
- FKs cascade-delete: removing a warehouse or product wipes the
  corresponding inventory rows.

### `wishlists`

The many-to-many bridge between `users` and `products` for "save for later".

- Unique on `(user_id, product_id)` — a user can't add the same product twice.
- `priority` ENUM (`low / medium / high`), optional `note`, and two
  notification flags: `notify_on_restock`, `notify_on_price_drop`.
- Both FKs cascade-delete.

---

## Entity relationships

```
                    +----------+
                    |  users   |
                    +----------+
                    /  | | |  \
                   /   | | |   \
     (vendor_id)  /    | | |    \  (user_id)
                 /     | | |     \
        +----------+   | | |   +----------+
        | products |   | | |   |  carts   |
        +----------+   | | |   +----------+
           |  |        | | |        |
           |  |        | | |        | (cart_id)
           |  |        | | |        v
           |  |        | | |   +-------------+
           |  |        | | |   | cart_items  |
           |  |        | | |   +-------------+
           |  |        | | |
           |  |        | | +-- wishlists -----+
           |  |        | |     (user_id,      |
           |  |        | |      product_id)   |
           |  |        | |                    |
           |  |        | +-- orders -----+    |
           |  |        |    (user_id,    |    |
           |  |        |     cart_id)    |    |
           |  |        |                 v    |
           |  |        |          +-------------+
           |  |        |          | order_items |
           |  |        |          +-------------+
           |  |        |                ^
           |  |        |                |
           |  +--------+----------------+
           |          (product_id everywhere above)
           |
           v
     +-----------+        +-------------+
     | inventory | <----- | warehouses  |
     +-----------+        +-------------+
```

Key relationships:

- `users 1—N products` (a vendor owns many products)
- `users 1—N carts`, with at most **one active** per user
- `carts 1—N cart_items`, `cart_items N—1 products`
- `users 1—N orders`, `orders 1—N order_items`
- `orders 1—1 carts` via `carts.converted_order_id` (set at checkout)
- `users M—N products` via `wishlists`
- `warehouses M—N products` via `inventory`, carrying stock numbers

---

## Sample queries you can try

Once everything is loaded, paste these into a query window:

**Top 10 customers by lifetime spend**
```sql
SELECT u.id, u.email,
       SUM(o.total_amount) AS lifetime_spend,
       COUNT(*)            AS orders_count
FROM   users  u
JOIN   orders o ON o.user_id = u.id
WHERE  o.status IN ('delivered','shipped','processing','confirmed')
GROUP  BY u.id, u.email
ORDER  BY lifetime_spend DESC
LIMIT  10;
```

**Products that are out of stock everywhere**
```sql
SELECT p.id, p.sku, p.name
FROM   products p
LEFT JOIN inventory i
       ON i.product_id = p.id AND i.quantity_available > 0
WHERE  i.id IS NULL;
```

**Active carts that have been sitting for more than 7 days (abandoned cart candidates)**
```sql
SELECT c.id, c.user_id, c.updated_at,
       COUNT(ci.id) AS items, SUM(ci.line_total) AS cart_value
FROM   carts c
JOIN   cart_items ci ON ci.cart_id = c.id
WHERE  c.status = 'active'
   AND c.updated_at < (NOW() - INTERVAL 7 DAY)
GROUP  BY c.id, c.user_id, c.updated_at
ORDER  BY cart_value DESC;
```

**Inventory items at or below their reorder point**
```sql
SELECT w.id   AS warehouse_id,
       p.sku, p.name,
       i.quantity_available, i.reorder_point, i.reorder_quantity
FROM   inventory  i
JOIN   products   p ON p.id = i.product_id
JOIN   warehouses w ON w.id = i.warehouse_id
WHERE  i.quantity_available <= i.reorder_point
ORDER  BY i.quantity_available ASC;
```

**Wishlist demand — which products are most-wished-for and how many of those wishers want a restock alert?**
```sql
SELECT p.sku, p.name,
       COUNT(*)                                AS wished_by,
       SUM(w.notify_on_restock)                AS wants_restock_ping,
       SUM(w.notify_on_price_drop)             AS wants_price_drop_ping
FROM   wishlists w
JOIN   products  p ON p.id = w.product_id
GROUP  BY p.id, p.sku, p.name
ORDER  BY wished_by DESC
LIMIT  20;
```

---

## Resetting the database

To start over from scratch:

```sql
DROP DATABASE ecommerce;
CREATE DATABASE ecommerce;
USE ecommerce;
```

Then re-run the notebooks in the [recommended order](#recommended-load-order).

If you only need to wipe the data but keep the schema:

```sql
SET FOREIGN_KEY_CHECKS = 0;
TRUNCATE TABLE order_items;
TRUNCATE TABLE orders;
TRUNCATE TABLE cart_items;
TRUNCATE TABLE carts;
TRUNCATE TABLE wishlists;
TRUNCATE TABLE inventory;
TRUNCATE TABLE warehouses;
TRUNCATE TABLE products;
TRUNCATE TABLE users;
SET FOREIGN_KEY_CHECKS = 1;
```

---

## Troubleshooting

**`ERROR 1822: Failed to add the foreign key constraint. Missing index for constraint`**
You ran the notebooks out of order — the referenced table does not exist yet.
Follow the [load order](#recommended-load-order).

**`ERROR 3819: Check constraint is violated`**
The seed `INSERT` includes a row that would make a monetary column negative.
Either fix the row or temporarily disable the check (not recommended in
production):
```sql
SET @@SESSION.check_constraint_checks = 0;
```

**`ERROR 1452: Cannot add or update a child row: a foreign key constraint fails`**
A row references a `user_id` / `product_id` / `warehouse_id` that has not been
inserted yet. Re-run the parent table's notebook first.

**`Table 'ecommerce.warehouses' doesn't exist` while loading inventory**
The `WarehouseTable.mysql-notebook` is a placeholder. Run the minimal
`CREATE TABLE warehouses (...)` snippet shown in the [warehouses](#warehouses)
section, then re-run the inventory notebook.

**MySQL Workbench shows `Statement is unsafe because it uses a system function that may return a different value on the replication slave.`**
Harmless warning on `DEFAULT (UUID())` when binary logging is on with
statement-based replication. Either ignore, or switch to row-based:
```sql
SET GLOBAL binlog_format = 'ROW';
```

**ENUM values not accepted**
ENUM values are case-sensitive in storage but case-insensitive in comparison.
Use the lowercase value spelled exactly as in the schema (e.g. `'customer'`,
not `'Customer'`).

**`Cannot delete or update a parent row: a foreign key constraint fails`**
Some FKs are `ON DELETE RESTRICT` on purpose (e.g. `orders.user_id`). Soft-delete
the user by setting `users.deleted_at = NOW()` instead of `DELETE`-ing.

---

## License

Educational project — the schema, seed data, and notebooks are free to copy,
fork, and modify.
