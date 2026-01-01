# DATABASE DESIGN & DATA MODELING
## E-commerce DevOps Platform

**Date:** January 1, 2026  
**Status:** Production-Grade Schema Design  
**Database:** PostgreSQL

---

## 🎯 DESIGN PRINCIPLES

### Core Rules (NON-NEGOTIABLE)
1. **Service Database Isolation:** Each microservice owns its own database/schema
2. **No Cross-Service Joins:** Services communicate via APIs only
3. **UUID for Public IDs:** Use UUID for all primary keys exposed externally
4. **Soft Delete Strategy:** Implement where audit trails are needed
5. **Optimistic Locking:** Use version columns for concurrency control

### Why This Matters
- ✅ Prevents data corruption
- ✅ Handles concurrent transactions
- ✅ Scales with traffic
- ✅ Enables independent service deployments

---

## 1️⃣ AUTH SERVICE DATABASE

### Schema: `auth_db`

### Table: `users`
```sql
CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(255) NOT NULL,
    password_hash       VARCHAR(255) NOT NULL,
    role                VARCHAR(50) NOT NULL DEFAULT 'CUSTOMER',
    is_active           BOOLEAN NOT NULL DEFAULT true,
    email_verified      BOOLEAN NOT NULL DEFAULT false,
    failed_login_count  INTEGER NOT NULL DEFAULT 0,
    last_login_at       TIMESTAMP,
    created_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at          TIMESTAMP NULL
);

-- Indexes
CREATE UNIQUE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_is_active ON users(is_active);

-- Constraints
ALTER TABLE users ADD CONSTRAINT chk_users_email_format 
    CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
ALTER TABLE users ADD CONSTRAINT chk_users_role 
    CHECK (role IN ('CUSTOMER', 'ADMIN', 'VENDOR'));
```

### Table: `refresh_tokens`
```sql
CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash      VARCHAR(255) NOT NULL,
    expires_at      TIMESTAMP NOT NULL,
    revoked_at      TIMESTAMP NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_expires_at ON refresh_tokens(expires_at);
```

### Why These Indexes?
- `idx_users_email`: Email lookup happens on **every login** (most frequent query)
- `idx_users_role`: Role-based access control queries
- `idx_refresh_tokens_user_id`: Token validation per user

---

## 2️⃣ PRODUCT SERVICE DATABASE

### Schema: `product_db`

### Table: `products`
```sql
CREATE TABLE products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    price           DECIMAL(10, 2) NOT NULL,
    category        VARCHAR(100) NOT NULL,
    sku             VARCHAR(100) NOT NULL,
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    deleted_at      TIMESTAMP NULL
);

-- Indexes
CREATE UNIQUE INDEX idx_products_sku ON products(sku) WHERE deleted_at IS NULL;
CREATE INDEX idx_products_category ON products(category) WHERE active = true;
CREATE INDEX idx_products_active ON products(active);
CREATE INDEX idx_products_price ON products(price) WHERE active = true;

-- Constraints
ALTER TABLE products ADD CONSTRAINT chk_products_price_positive 
    CHECK (price > 0);
ALTER TABLE products ADD CONSTRAINT chk_products_name_not_empty 
    CHECK (length(trim(name)) > 0);
```

### Table: `product_images`
```sql
CREATE TABLE product_images (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    image_url       VARCHAR(500) NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    display_order   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_product_images_product_id ON product_images(product_id);
```

### 🚨 CRITICAL: NO INVENTORY HERE
**Inventory is stored in a separate service to prevent race conditions and overselling.**

---

## 3️⃣ INVENTORY SERVICE DATABASE (MOST CRITICAL)

### Schema: `inventory_db`

### Table: `inventory`
```sql
CREATE TABLE inventory (
    product_id          UUID PRIMARY KEY,  -- No FK to products (different service!)
    available_quantity  INTEGER NOT NULL DEFAULT 0,
    reserved_quantity   INTEGER NOT NULL DEFAULT 0,
    version             INTEGER NOT NULL DEFAULT 0,  -- Optimistic locking
    updated_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Constraints (PREVENT NEGATIVE STOCK)
ALTER TABLE inventory ADD CONSTRAINT chk_inventory_available_positive 
    CHECK (available_quantity >= 0);
ALTER TABLE inventory ADD CONSTRAINT chk_inventory_reserved_positive 
    CHECK (reserved_quantity >= 0);

-- Indexes
CREATE INDEX idx_inventory_available ON inventory(available_quantity) 
    WHERE available_quantity > 0;
```

### Table: `inventory_transactions`
```sql
CREATE TABLE inventory_transactions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id          UUID NOT NULL,
    transaction_type    VARCHAR(50) NOT NULL,
    quantity_change     INTEGER NOT NULL,
    reference_id        UUID,  -- order_id or reservation_id
    notes               TEXT,
    created_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_inventory_txn_product_id ON inventory_transactions(product_id);
CREATE INDEX idx_inventory_txn_reference_id ON inventory_transactions(reference_id);
CREATE INDEX idx_inventory_txn_created_at ON inventory_transactions(created_at DESC);

-- Constraints
ALTER TABLE inventory_transactions ADD CONSTRAINT chk_txn_type 
    CHECK (transaction_type IN ('RESERVE', 'RELEASE', 'CONFIRM', 'RESTOCK', 'ADJUSTMENT'));
```

### Why Separate Inventory Service?
1. **Prevents Overselling:** Centralized control with atomic operations
2. **Handles Concurrency:** Optimistic locking with version field
3. **Audit Trail:** All stock movements tracked in `inventory_transactions`
4. **Performance:** Inventory queries don't slow down product searches

---

## 4️⃣ CART SERVICE DATABASE

### Schema: `cart_db`

### Table: `carts`
```sql
CREATE TABLE carts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,  -- No FK (different service)
    expires_at      TIMESTAMP NOT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE UNIQUE INDEX idx_carts_user_id ON carts(user_id) WHERE expires_at > CURRENT_TIMESTAMP;
CREATE INDEX idx_carts_expires_at ON carts(expires_at);
```

### Table: `cart_items`
```sql
CREATE TABLE cart_items (
    cart_id         UUID NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL,  -- No FK (different service)
    quantity        INTEGER NOT NULL,
    added_at        TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (cart_id, product_id)
);

-- Constraints
ALTER TABLE cart_items ADD CONSTRAINT chk_cart_items_quantity_positive 
    CHECK (quantity > 0);

-- Indexes
CREATE INDEX idx_cart_items_cart_id ON cart_items(cart_id);
```

### 📌 Cart Cleanup Strategy
```sql
-- Delete expired carts (run daily via cron job)
DELETE FROM carts WHERE expires_at < CURRENT_TIMESTAMP - INTERVAL '7 days';
```

### Future Optimization
**Move carts to Redis** for:
- Faster access (in-memory)
- Automatic TTL expiration
- Reduced database load

---

## 5️⃣ ORDER SERVICE DATABASE

### Schema: `order_db`

### Table: `orders`
```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,  -- No FK (different service)
    order_number    VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'CREATED',
    total_amount    DECIMAL(10, 2) NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'USD',
    shipping_address TEXT NOT NULL,
    billing_address TEXT NOT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE UNIQUE INDEX idx_orders_order_number ON orders(order_number);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Constraints
ALTER TABLE orders ADD CONSTRAINT chk_orders_total_positive 
    CHECK (total_amount > 0);
ALTER TABLE orders ADD CONSTRAINT chk_orders_status 
    CHECK (status IN ('CREATED', 'PAID', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED', 'REFUNDED'));
```

### Table: `order_items`
```sql
CREATE TABLE order_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id      UUID NOT NULL,  -- No FK (different service)
    product_name    VARCHAR(255) NOT NULL,  -- Snapshot at order time
    price           DECIMAL(10, 2) NOT NULL,  -- Price at order time
    quantity        INTEGER NOT NULL,
    subtotal        DECIMAL(10, 2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

-- Indexes
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Constraints
ALTER TABLE order_items ADD CONSTRAINT chk_order_items_price_positive 
    CHECK (price > 0);
ALTER TABLE order_items ADD CONSTRAINT chk_order_items_quantity_positive 
    CHECK (quantity > 0);
```

### Table: `order_status_history`
```sql
CREATE TABLE order_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    from_status     VARCHAR(50),
    to_status       VARCHAR(50) NOT NULL,
    notes           TEXT,
    changed_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_order_status_history_order_id ON order_status_history(order_id);
CREATE INDEX idx_order_status_history_changed_at ON order_status_history(changed_at DESC);
```

---

## 6️⃣ PAYMENT SERVICE DATABASE

### Schema: `payment_db`

### Table: `payments`
```sql
CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id            UUID NOT NULL,  -- No FK (different service)
    payment_intent_id   VARCHAR(255),  -- Stripe/PayPal reference
    amount              DECIMAL(10, 2) NOT NULL,
    currency            VARCHAR(3) NOT NULL DEFAULT 'USD',
    status              VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    provider            VARCHAR(50) NOT NULL,
    payment_method      VARCHAR(50),
    error_message       TEXT,
    created_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE UNIQUE INDEX idx_payments_order_id ON payments(order_id);
CREATE INDEX idx_payments_payment_intent_id ON payments(payment_intent_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_created_at ON payments(created_at DESC);

-- Constraints
ALTER TABLE payments ADD CONSTRAINT chk_payments_amount_positive 
    CHECK (amount > 0);
ALTER TABLE payments ADD CONSTRAINT chk_payments_status 
    CHECK (status IN ('PENDING', 'SUCCESS', 'FAILED', 'REFUNDED', 'CANCELLED'));
ALTER TABLE payments ADD CONSTRAINT chk_payments_provider 
    CHECK (provider IN ('STRIPE', 'PAYPAL', 'RAZORPAY', 'MANUAL'));
```

### Table: `payment_events`
```sql
CREATE TABLE payment_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL REFERENCES payments(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,
    event_data      JSONB NOT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_payment_events_payment_id ON payment_events(payment_id);
CREATE INDEX idx_payment_events_created_at ON payment_events(created_at DESC);
```

---

## 🔄 STATE MACHINES

### Order State Machine
```
CREATED → PAID → CONFIRMED → SHIPPED → DELIVERED
   ↓         ↓                    ↓
CANCELLED  REFUNDED           CANCELLED
```

**Valid Transitions:**
```sql
CREATE TABLE order_state_transitions (
    from_state VARCHAR(50),
    to_state VARCHAR(50) NOT NULL,
    PRIMARY KEY (from_state, to_state)
);

INSERT INTO order_state_transitions VALUES
    (NULL, 'CREATED'),
    ('CREATED', 'PAID'),
    ('CREATED', 'CANCELLED'),
    ('PAID', 'CONFIRMED'),
    ('PAID', 'REFUNDED'),
    ('CONFIRMED', 'SHIPPED'),
    ('CONFIRMED', 'CANCELLED'),
    ('SHIPPED', 'DELIVERED'),
    ('SHIPPED', 'CANCELLED');
```

### Payment State Machine
```
PENDING → SUCCESS
   ↓         ↓
FAILED   REFUNDED
   ↓
CANCELLED
```

**Valid Transitions:**
```sql
CREATE TABLE payment_state_transitions (
    from_state VARCHAR(50),
    to_state VARCHAR(50) NOT NULL,
    PRIMARY KEY (from_state, to_state)
);

INSERT INTO payment_state_transitions VALUES
    (NULL, 'PENDING'),
    ('PENDING', 'SUCCESS'),
    ('PENDING', 'FAILED'),
    ('PENDING', 'CANCELLED'),
    ('SUCCESS', 'REFUNDED');
```

---

## 🔐 CRITICAL TRANSACTION PATTERNS

### 1. Inventory Reservation (On Order Creation)
```sql
-- MUST be executed in a single transaction
BEGIN;

-- Check availability with row-level lock
SELECT available_quantity, reserved_quantity, version 
FROM inventory 
WHERE product_id = $1 
FOR UPDATE;

-- Reserve inventory
UPDATE inventory 
SET 
    available_quantity = available_quantity - $2,
    reserved_quantity = reserved_quantity + $2,
    version = version + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE 
    product_id = $1 
    AND available_quantity >= $2
    AND version = $3;  -- Optimistic locking

-- Log transaction
INSERT INTO inventory_transactions 
    (product_id, transaction_type, quantity_change, reference_id, notes)
VALUES 
    ($1, 'RESERVE', -$2, $4, 'Order reservation');

COMMIT;
```

**Why This Works:**
- ✅ `FOR UPDATE` prevents concurrent modifications
- ✅ `version` field detects conflicts
- ✅ CHECK constraint prevents negative stock
- ✅ All-or-nothing atomicity

### 2. Payment Failure (Release Reservation)
```sql
BEGIN;

-- Release reserved inventory
UPDATE inventory 
SET 
    available_quantity = available_quantity + $2,
    reserved_quantity = reserved_quantity - $2,
    version = version + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE 
    product_id = $1;

-- Log transaction
INSERT INTO inventory_transactions 
    (product_id, transaction_type, quantity_change, reference_id, notes)
VALUES 
    ($1, 'RELEASE', $2, $3, 'Payment failed - releasing reservation');

COMMIT;
```

### 3. Order Confirmation (Convert Reserved to Sold)
```sql
BEGIN;

-- Convert reserved to sold
UPDATE inventory 
SET 
    reserved_quantity = reserved_quantity - $2,
    version = version + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE 
    product_id = $1
    AND reserved_quantity >= $2;

-- Log transaction
INSERT INTO inventory_transactions 
    (product_id, transaction_type, quantity_change, reference_id, notes)
VALUES 
    ($1, 'CONFIRM', -$2, $3, 'Order confirmed - inventory sold');

COMMIT;
```

---

## 🛡️ CONCURRENCY CONTROL STRATEGIES

### Optimistic Locking (Preferred)
```sql
-- Read with version
SELECT * FROM inventory WHERE product_id = $1;

-- Update with version check
UPDATE inventory 
SET 
    available_quantity = available_quantity - $2,
    version = version + 1
WHERE 
    product_id = $1 
    AND version = $3;  -- Fails if version changed

-- Check affected rows
IF affected_rows = 0 THEN
    RAISE EXCEPTION 'Concurrent modification detected - retry';
END IF;
```

### Pessimistic Locking (For High Contention)
```sql
BEGIN;

-- Lock the row
SELECT * FROM inventory 
WHERE product_id = $1 
FOR UPDATE NOWAIT;  -- Fail immediately if locked

-- Perform update
UPDATE inventory 
SET available_quantity = available_quantity - $2
WHERE product_id = $1;

COMMIT;
```

---

## 📊 INDEX STRATEGY RATIONALE

### High-Frequency Query Indexes
1. **Email Lookup** (`users.email`): Every login attempt
2. **Order by User** (`orders.user_id`): User order history
3. **Product Category** (`products.category`): Catalog browsing
4. **Order Status** (`orders.status`): Admin dashboards

### Partial Indexes (Space Optimization)
```sql
-- Only index active products
CREATE INDEX idx_products_active_category 
ON products(category) WHERE active = true;

-- Only index non-expired carts
CREATE INDEX idx_carts_active 
ON carts(user_id) WHERE expires_at > CURRENT_TIMESTAMP;
```

### Covering Indexes (Performance)
```sql
-- Include commonly selected columns
CREATE INDEX idx_orders_user_status_created 
ON orders(user_id, status, created_at) 
INCLUDE (order_number, total_amount);
```

---

## 🔍 INTERVIEW QUESTIONS & ANSWERS

### Q1: How do you prevent overselling?
**Answer:**
1. Separate inventory service with centralized control
2. Row-level locking (`FOR UPDATE`) during reservation
3. CHECK constraints preventing negative stock
4. Two-phase commit: Reserve → Confirm
5. Optimistic locking with version field

### Q2: Why is inventory separate from products?
**Answer:**
1. **Concurrency:** Product reads don't lock inventory
2. **Scalability:** Inventory updates are write-heavy, products are read-heavy
3. **Responsibility:** Product service handles catalog, inventory service handles stock
4. **Transaction Isolation:** Inventory changes don't affect product searches

### Q3: How do you handle concurrent orders for the same product?
**Answer:**
1. **Database Level:** `FOR UPDATE` lock + version field
2. **Application Level:** Retry logic with exponential backoff
3. **Business Level:** Reserved quantity vs available quantity separation
4. **Monitoring:** Track failed reservation attempts

### Q4: What happens if payment fails after inventory reservation?
**Answer:**
1. Payment service publishes `PAYMENT_FAILED` event
2. Order service transitions order to `CANCELLED`
3. Inventory service releases reservation:
   - `reserved_quantity -= qty`
   - `available_quantity += qty`
4. All in compensating transaction with audit trail

### Q5: How do you ensure data consistency across services?
**Answer:**
1. **Saga Pattern:** Choreography with events
2. **Idempotency:** All operations have unique request IDs
3. **Audit Logs:** Every state change tracked
4. **Reconciliation Jobs:** Nightly consistency checks
5. **No Distributed Transactions:** Eventual consistency with compensation

---

## 🚀 MIGRATION SCRIPTS

### Create All Schemas
```sql
-- Run in order
CREATE DATABASE auth_db;
CREATE DATABASE product_db;
CREATE DATABASE inventory_db;
CREATE DATABASE cart_db;
CREATE DATABASE order_db;
CREATE DATABASE payment_db;

-- Enable UUID extension on all databases
\c auth_db
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

\c product_db
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

\c inventory_db
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

\c cart_db
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

\c order_db
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

\c payment_db
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

---

## 📈 SCALING CONSIDERATIONS

### Read Replicas
```yaml
master: write operations
replica-1: user-facing reads (orders, products)
replica-2: analytics queries
```

### Sharding Strategy (Future)
```yaml
# Shard by user_id hash
shard-1: user_id % 4 = 0
shard-2: user_id % 4 = 1
shard-3: user_id % 4 = 2
shard-4: user_id % 4 = 3
```

### Cache Strategy
```yaml
Redis:
  - Product catalog (1 hour TTL)
  - User sessions (30 min TTL)
  - Cart data (24 hour TTL)

Cache Invalidation:
  - Product update → Invalidate product cache
  - Price change → Invalidate category cache
```

---

## ✅ DAY 2 CHECKLIST

- [x] All table schemas defined with proper types
- [x] Indexes for high-frequency queries
- [x] CHECK constraints preventing invalid states
- [x] Foreign keys within service boundaries only
- [x] Optimistic locking with version fields
- [x] Audit tables for critical operations
- [x] State machines with valid transitions
- [x] Transaction patterns for inventory management
- [x] Concurrency control strategies
- [x] Interview question preparation

---

## 📚 NEXT STEPS (DAY 3-4)

1. Generate SQL migration scripts using Flyway/Liquibase
2. Implement database connection pools in Spring Boot
3. Add JPA entities with proper annotations
4. Create repository interfaces
5. Write integration tests for transaction patterns
6. Set up database monitoring (slow query logs)

**Status:** Ready for implementation ✅
