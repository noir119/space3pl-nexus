# NEXUS Admin Panel — Complete Project Guide
## From Demo HTML → Production Next.js 15 + Supabase Application

> **Stack:** Next.js 15 (App Router) · TypeScript (strict) · Supabase (PostgreSQL + Auth + Storage) · Tailwind CSS · shadcn/ui · Drizzle ORM · Stripe · Vercel
> **Based on:** Admin Panel v3 Final — Orders, Returns, Exchanges, Partial Fulfillment, Inventory (SOH, Bundles, Cycle Count, Movement), Logistics (Dispatch, Route Builder, Driver Management, Last-Mile Tracking + POD)

---

## Table of Contents

1. [Project Architecture Overview](#1-project-architecture-overview)
2. [Environment Setup](#2-environment-setup)
3. [Supabase Database Schema](#3-supabase-database-schema)
4. [TypeScript Type Definitions](#4-typescript-type-definitions)
5. [Drizzle ORM Schema](#5-drizzle-orm-schema)
6. [Server Actions & API Logic](#6-server-actions--api-logic)
7. [Row Level Security (RLS) Policies](#7-row-level-security-rls-policies)
8. [File & Folder Structure](#8-file--folder-structure)
9. [TRAE AI Prompt Library](#9-trae-ai-prompt-library)
10. [Deployment Checklist](#10-deployment-checklist)

---

## 1. Project Architecture Overview

```
Browser (Admin Panel)
      │
      ▼
Next.js 15 App Router  ──►  Supabase Auth (session)
      │                            │
      ├── Server Components         ├── PostgreSQL (all data)
      ├── Server Actions            ├── Storage (POD photos)
      └── Client Components         └── Realtime (live tracking)
                │
                ▼
         Stripe (payments/refunds)
         Resend (email reports)
         Vercel (deployment + edge)
```

### Core Modules → Database Mapping

| Admin Module | Primary Tables |
|---|---|
| Orders | `orders`, `order_items`, `order_status_history` |
| Returns / RMA | `returns`, `return_items`, `return_history` |
| Exchanges | `exchanges` |
| Partial Fulfillment | `fulfillments`, `fulfillment_items` |
| Inventory / SOH | `products`, `inventory_levels`, `stock_locations` |
| Bundles | `bundles`, `bundle_items` |
| Cycle Count | `cycle_counts`, `cycle_count_items` |
| Movement Tracker | `inventory_movements` |
| Dispatch / Logistics | `routes`, `route_stops` |
| Driver Management | `drivers` |
| Last-Mile / POD | `route_stops`, `pod_records` |
| Vehicles / Fleet | `vehicles` |
| Customers | `customers` |
| Reports | Generated at query-time (no storage table needed) |

---

## 2. Environment Setup

### 2.1 Install & Initialize

```bash
# Create Next.js 15 project
npx create-next-app@latest nexus-admin \
  --typescript \
  --tailwind \
  --app \
  --src-dir \
  --import-alias "@/*"

cd nexus-admin

# Install core dependencies
npm install @supabase/supabase-js @supabase/ssr
npm install drizzle-orm drizzle-kit postgres
npm install stripe @stripe/stripe-js
npm install resend
npm install zod react-hook-form @hookform/resolvers
npm install recharts
npm install @tanstack/react-table
npm install date-fns

# shadcn/ui
npx shadcn@latest init
npx shadcn@latest add button card table badge dialog sheet select input textarea form toast

# Dev dependencies
npm install -D drizzle-kit tsx dotenv-cli
```

### 2.2 Environment Variables (`.env.local`)

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIs...
SUPABASE_SERVICE_ROLE_KEY=eyJhbGciOiJIUzI1NiIs...
DATABASE_URL=postgresql://postgres:[password]@db.xxxx.supabase.co:5432/postgres

# Stripe
STRIPE_SECRET_KEY=sk_live_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_live_...

# Resend (email reports)
RESEND_API_KEY=re_...
REPORT_EMAIL_FROM=reports@nexus.com
ADMIN_EMAIL=admin@nexus.com

# App
NEXT_PUBLIC_APP_URL=https://admin.nexus.com
NODE_ENV=production
```

### 2.3 TypeScript Config (`tsconfig.json` additions)

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "paths": {
      "@/*": ["./src/*"],
      "@db/*": ["./src/db/*"],
      "@types/*": ["./src/types/*"],
      "@actions/*": ["./src/actions/*"]
    }
  }
}
```

### 2.4 Drizzle Config (`drizzle.config.ts`)

```typescript
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/db/schema/*',
  out: './supabase/migrations',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
  verbose: true,
  strict: true,
} satisfies Config;
```

---

## 3. Supabase Database Schema

> Copy each block into **Supabase → SQL Editor → New Query** and run in order.
> Run them in the numbered sequence to respect foreign key dependencies.

### 3.1 — Extensions & Helpers

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Updated-at trigger function (reused by all tables)
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Helper: apply updated_at trigger to any table
CREATE OR REPLACE FUNCTION apply_updated_at(target_table TEXT)
RETURNS VOID AS $$
BEGIN
  EXECUTE format(
    'CREATE TRIGGER trg_updated_at BEFORE UPDATE ON %I
     FOR EACH ROW EXECUTE FUNCTION update_updated_at()',
    target_table
  );
END;
$$ LANGUAGE plpgsql;
```

---

### 3.2 — Users / Admins

```sql
-- Admin roles enum
CREATE TYPE admin_role AS ENUM ('super_admin', 'admin', 'manager', 'warehouse', 'logistics', 'finance');

-- Admin profiles (linked to Supabase Auth users)
CREATE TABLE admin_profiles (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id       UUID NOT NULL UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name     TEXT NOT NULL,
  email         TEXT NOT NULL UNIQUE,
  role          admin_role NOT NULL DEFAULT 'admin',
  avatar_url    TEXT,
  phone         TEXT,
  is_active     BOOLEAN NOT NULL DEFAULT TRUE,
  last_login_at TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('admin_profiles');

-- Index
CREATE INDEX idx_admin_profiles_user_id ON admin_profiles(user_id);
CREATE INDEX idx_admin_profiles_role    ON admin_profiles(role);
```

---

### 3.3 — Customers

```sql
CREATE TABLE customers (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email         TEXT NOT NULL UNIQUE,
  full_name     TEXT NOT NULL,
  phone         TEXT,
  country       TEXT,
  city          TEXT,
  address_line1 TEXT,
  address_line2 TEXT,
  postal_code   TEXT,
  notes         TEXT,
  total_orders  INTEGER NOT NULL DEFAULT 0,
  total_spent   NUMERIC(12,2) NOT NULL DEFAULT 0,
  is_active     BOOLEAN NOT NULL DEFAULT TRUE,
  joined_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('customers');

CREATE INDEX idx_customers_email   ON customers(email);
CREATE INDEX idx_customers_country ON customers(country);
```

---

### 3.4 — Products & Inventory

```sql
-- Product category enum
CREATE TYPE product_category AS ENUM (
  'electronics', 'apparel', 'home', 'accessories', 'bundle', 'other'
);

CREATE TYPE product_status AS ENUM ('active', 'draft', 'archived');

-- Core products table
CREATE TABLE products (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name             TEXT NOT NULL,
  sku              TEXT NOT NULL UNIQUE,
  description      TEXT,
  category         product_category NOT NULL DEFAULT 'other',
  status           product_status NOT NULL DEFAULT 'draft',
  price            NUMERIC(10,2) NOT NULL,
  compare_at_price NUMERIC(10,2),
  cost_price       NUMERIC(10,2),
  weight_grams     INTEGER,
  reorder_point    INTEGER NOT NULL DEFAULT 10,
  tags             TEXT[],
  images           TEXT[],         -- Supabase Storage URLs
  is_bundle        BOOLEAN NOT NULL DEFAULT FALSE,
  total_sold       INTEGER NOT NULL DEFAULT 0,
  created_by       UUID REFERENCES admin_profiles(id),
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('products');

CREATE INDEX idx_products_sku      ON products(sku);
CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_status   ON products(status);

-- Stock locations (warehouses)
CREATE TABLE stock_locations (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name        TEXT NOT NULL,
  code        TEXT NOT NULL UNIQUE,   -- e.g. 'WH-A'
  address     TEXT,
  city        TEXT,
  country     TEXT,
  is_active   BOOLEAN NOT NULL DEFAULT TRUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('stock_locations');

-- Inventory levels per product per location
CREATE TABLE inventory_levels (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  product_id       UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  location_id      UUID NOT NULL REFERENCES stock_locations(id) ON DELETE CASCADE,
  quantity_on_hand INTEGER NOT NULL DEFAULT 0,
  quantity_reserved INTEGER NOT NULL DEFAULT 0,
  quantity_available INTEGER GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved) STORED,
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(product_id, location_id)
);

CREATE INDEX idx_inv_levels_product  ON inventory_levels(product_id);
CREATE INDEX idx_inv_levels_location ON inventory_levels(location_id);

-- Inventory movement types
CREATE TYPE movement_type AS ENUM (
  'sale', 'return', 'purchase', 'adjustment',
  'transfer_in', 'transfer_out', 'cycle_count', 'damage', 'write_off'
);

-- Full inventory movement ledger
CREATE TABLE inventory_movements (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  product_id       UUID NOT NULL REFERENCES products(id),
  location_id      UUID NOT NULL REFERENCES stock_locations(id),
  movement_type    movement_type NOT NULL,
  quantity_change  INTEGER NOT NULL,               -- positive = in, negative = out
  quantity_before  INTEGER NOT NULL,
  quantity_after   INTEGER NOT NULL,
  reference_type   TEXT,                           -- 'order', 'return', 'po', 'adjustment', 'transfer'
  reference_id     UUID,                           -- FK to relevant table
  reference_code   TEXT,                           -- Human-readable e.g. '#ORD-8821'
  notes            TEXT,
  performed_by     UUID REFERENCES admin_profiles(id),
  performed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_inv_mov_product   ON inventory_movements(product_id);
CREATE INDEX idx_inv_mov_type      ON inventory_movements(movement_type);
CREATE INDEX idx_inv_mov_ref       ON inventory_movements(reference_type, reference_id);
CREATE INDEX idx_inv_mov_date      ON inventory_movements(performed_at DESC);
```

---

### 3.5 — Bundles

```sql
CREATE TABLE bundles (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  product_id   UUID NOT NULL UNIQUE REFERENCES products(id) ON DELETE CASCADE,
  bundle_price NUMERIC(10,2) NOT NULL,
  discount_pct NUMERIC(5,2),   -- auto-calculated but stored for display
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('bundles');

CREATE TABLE bundle_items (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  bundle_id    UUID NOT NULL REFERENCES bundles(id) ON DELETE CASCADE,
  product_id   UUID NOT NULL REFERENCES products(id),
  quantity     INTEGER NOT NULL DEFAULT 1,
  unit_price   NUMERIC(10,2) NOT NULL,   -- price at time of bundle creation
  UNIQUE(bundle_id, product_id)
);

CREATE INDEX idx_bundle_items_bundle  ON bundle_items(bundle_id);
CREATE INDEX idx_bundle_items_product ON bundle_items(product_id);
```

---

### 3.6 — Cycle Counts

```sql
CREATE TYPE cycle_count_method AS ENUM (
  'abc_analysis', 'location_based', 'random_sample', 'full_physical'
);

CREATE TYPE cycle_count_status AS ENUM (
  'scheduled', 'in_progress', 'completed', 'cancelled'
);

CREATE TABLE cycle_counts (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name         TEXT NOT NULL,
  method       cycle_count_method NOT NULL,
  scope        TEXT NOT NULL,
  location_id  UUID REFERENCES stock_locations(id),
  assigned_to  TEXT,                               -- team name or admin id
  scheduled_at DATE NOT NULL,
  started_at   TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  status       cycle_count_status NOT NULL DEFAULT 'scheduled',
  notes        TEXT,
  created_by   UUID REFERENCES admin_profiles(id),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('cycle_counts');

CREATE TABLE cycle_count_items (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  cycle_count_id    UUID NOT NULL REFERENCES cycle_counts(id) ON DELETE CASCADE,
  product_id        UUID NOT NULL REFERENCES products(id),
  location_id       UUID NOT NULL REFERENCES stock_locations(id),
  expected_quantity INTEGER NOT NULL,
  counted_quantity  INTEGER,
  variance          INTEGER GENERATED ALWAYS AS (
                      COALESCE(counted_quantity, 0) - expected_quantity
                    ) STORED,
  is_counted        BOOLEAN NOT NULL DEFAULT FALSE,
  counted_by        UUID REFERENCES admin_profiles(id),
  counted_at        TIMESTAMPTZ,
  notes             TEXT,
  UNIQUE(cycle_count_id, product_id, location_id)
);

CREATE INDEX idx_cc_items_count   ON cycle_count_items(cycle_count_id);
CREATE INDEX idx_cc_items_product ON cycle_count_items(product_id);
```

---

### 3.7 — Orders

```sql
CREATE TYPE payment_status    AS ENUM ('pending', 'paid', 'failed', 'refunded', 'partially_refunded');
CREATE TYPE fulfillment_status AS ENUM ('unfulfilled', 'processing', 'partially_fulfilled', 'shipped', 'delivered', 'cancelled');
CREATE TYPE order_priority     AS ENUM ('normal', 'express', 'urgent');

CREATE TABLE orders (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  order_number        TEXT NOT NULL UNIQUE,          -- e.g. 'ORD-8821'
  customer_id         UUID NOT NULL REFERENCES customers(id),
  payment_status      payment_status NOT NULL DEFAULT 'pending',
  fulfillment_status  fulfillment_status NOT NULL DEFAULT 'unfulfilled',
  priority            order_priority NOT NULL DEFAULT 'normal',
  subtotal            NUMERIC(12,2) NOT NULL,
  shipping_cost       NUMERIC(10,2) NOT NULL DEFAULT 0,
  tax_amount          NUMERIC(10,2) NOT NULL DEFAULT 0,
  discount_amount     NUMERIC(10,2) NOT NULL DEFAULT 0,
  total               NUMERIC(12,2) NOT NULL,
  currency            CHAR(3) NOT NULL DEFAULT 'USD',
  shipping_name       TEXT,
  shipping_address1   TEXT,
  shipping_address2   TEXT,
  shipping_city       TEXT,
  shipping_state      TEXT,
  shipping_postal     TEXT,
  shipping_country    TEXT,
  carrier             TEXT,
  tracking_number     TEXT,
  stripe_payment_id   TEXT,
  internal_notes      TEXT,
  tags                TEXT[],
  placed_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('orders');

CREATE INDEX idx_orders_number      ON orders(order_number);
CREATE INDEX idx_orders_customer    ON orders(customer_id);
CREATE INDEX idx_orders_payment     ON orders(payment_status);
CREATE INDEX idx_orders_fulfillment ON orders(fulfillment_status);
CREATE INDEX idx_orders_placed_at   ON orders(placed_at DESC);

-- Order line items
CREATE TABLE order_items (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  order_id     UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id   UUID NOT NULL REFERENCES products(id),
  product_name TEXT NOT NULL,   -- snapshot at time of order
  sku          TEXT NOT NULL,
  quantity     INTEGER NOT NULL,
  unit_price   NUMERIC(10,2) NOT NULL,
  line_total   NUMERIC(12,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
  fulfilled_qty INTEGER NOT NULL DEFAULT 0,
  returned_qty  INTEGER NOT NULL DEFAULT 0
);

CREATE INDEX idx_order_items_order   ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);

-- Order status history (full audit trail)
CREATE TABLE order_status_history (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  order_id     UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  from_status  TEXT,
  to_status    TEXT NOT NULL,
  changed_by   UUID REFERENCES admin_profiles(id),
  notes        TEXT,
  changed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ord_hist_order ON order_status_history(order_id);
CREATE INDEX idx_ord_hist_date  ON order_status_history(changed_at DESC);
```

---

### 3.8 — Returns (RMA)

```sql
CREATE TYPE return_status     AS ENUM ('requested', 'approved', 'rejected', 'in_transit', 'received', 'inspected', 'resolved', 'cancelled');
CREATE TYPE return_resolution AS ENUM ('refund', 'store_credit', 'exchange', 'repair', 'no_action');
CREATE TYPE item_condition    AS ENUM ('unopened', 'like_new', 'used_good', 'used_damaged', 'not_received');
CREATE TYPE return_initiator  AS ENUM ('customer', 'admin', 'system');

CREATE TABLE returns (
  id                UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  rma_number        TEXT NOT NULL UNIQUE,             -- e.g. 'RMA-2601'
  order_id          UUID NOT NULL REFERENCES orders(id),
  customer_id       UUID NOT NULL REFERENCES customers(id),
  status            return_status NOT NULL DEFAULT 'requested',
  resolution        return_resolution,
  initiated_by      return_initiator NOT NULL DEFAULT 'customer',
  refund_amount     NUMERIC(10,2),
  store_credit_amt  NUMERIC(10,2),
  return_tracking   TEXT,
  return_carrier    TEXT,
  shipping_paid_by  TEXT NOT NULL DEFAULT 'customer',  -- 'customer' | 'store'
  warehouse_notes   TEXT,
  resolution_notes  TEXT,
  requested_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  approved_at       TIMESTAMPTZ,
  received_at       TIMESTAMPTZ,
  resolved_at       TIMESTAMPTZ,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('returns');

CREATE INDEX idx_returns_rma_number ON returns(rma_number);
CREATE INDEX idx_returns_order      ON returns(order_id);
CREATE INDEX idx_returns_customer   ON returns(customer_id);
CREATE INDEX idx_returns_status     ON returns(status);

-- Individual items within a return (one order can have multiple returns,
-- each return can cover multiple items)
CREATE TABLE return_items (
  id             UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  return_id      UUID NOT NULL REFERENCES returns(id) ON DELETE CASCADE,
  order_item_id  UUID NOT NULL REFERENCES order_items(id),
  product_id     UUID NOT NULL REFERENCES products(id),
  product_name   TEXT NOT NULL,
  sku            TEXT NOT NULL,
  quantity       INTEGER NOT NULL,
  reason         TEXT NOT NULL,
  condition      item_condition NOT NULL DEFAULT 'used_good',
  refund_amount  NUMERIC(10,2),
  is_restockable BOOLEAN NOT NULL DEFAULT FALSE,
  qc_notes       TEXT
);

CREATE INDEX idx_return_items_return  ON return_items(return_id);
CREATE INDEX idx_return_items_product ON return_items(product_id);

-- Return event history (full timeline)
CREATE TABLE return_history (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  return_id   UUID NOT NULL REFERENCES returns(id) ON DELETE CASCADE,
  event       TEXT NOT NULL,
  notes       TEXT,
  performed_by UUID REFERENCES admin_profiles(id),
  performed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ret_hist_return ON return_history(return_id);
CREATE INDEX idx_ret_hist_date   ON return_history(performed_at DESC);
```

---

### 3.9 — Exchanges

```sql
CREATE TYPE exchange_status AS ENUM ('pending', 'approved', 'return_received', 'shipped', 'completed', 'cancelled');

CREATE TABLE exchanges (
  id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  exchange_number     TEXT NOT NULL UNIQUE,            -- e.g. 'EXC-001'
  return_id           UUID REFERENCES returns(id),     -- linked RMA if exists
  order_id            UUID NOT NULL REFERENCES orders(id),
  customer_id         UUID NOT NULL REFERENCES customers(id),
  return_product_id   UUID NOT NULL REFERENCES products(id),
  return_product_name TEXT NOT NULL,
  return_sku          TEXT NOT NULL,
  return_qty          INTEGER NOT NULL DEFAULT 1,
  return_reason       TEXT,
  return_condition    item_condition,
  new_product_id      UUID NOT NULL REFERENCES products(id),
  new_product_name    TEXT NOT NULL,
  new_sku             TEXT NOT NULL,
  new_qty             INTEGER NOT NULL DEFAULT 1,
  price_difference    NUMERIC(10,2) NOT NULL DEFAULT 0, -- positive = charge customer
  charge_method       TEXT,                             -- 'charge' | 'credit' | 'none'
  status              exchange_status NOT NULL DEFAULT 'pending',
  new_order_id        UUID REFERENCES orders(id),       -- new order created for replacement
  notes               TEXT,
  created_by          UUID REFERENCES admin_profiles(id),
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('exchanges');

CREATE INDEX idx_exchanges_order    ON exchanges(order_id);
CREATE INDEX idx_exchanges_customer ON exchanges(customer_id);
CREATE INDEX idx_exchanges_status   ON exchanges(status);
```

---

### 3.10 — Fulfillments (Partial Fulfillment)

```sql
CREATE TYPE fulfillment_item_status AS ENUM (
  'pending', 'processing', 'shipped', 'delivered', 'cancelled', 'backordered'
);

CREATE TABLE fulfillments (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  order_id        UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  location_id     UUID REFERENCES stock_locations(id),
  carrier         TEXT,
  tracking_number TEXT,
  shipped_at      TIMESTAMPTZ,
  delivered_at    TIMESTAMPTZ,
  notes           TEXT,
  created_by      UUID REFERENCES admin_profiles(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('fulfillments');

CREATE INDEX idx_fulfillments_order ON fulfillments(order_id);

CREATE TABLE fulfillment_items (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  fulfillment_id  UUID NOT NULL REFERENCES fulfillments(id) ON DELETE CASCADE,
  order_item_id   UUID NOT NULL REFERENCES order_items(id),
  product_id      UUID NOT NULL REFERENCES products(id),
  quantity        INTEGER NOT NULL,
  status          fulfillment_item_status NOT NULL DEFAULT 'pending',
  notes           TEXT
);

CREATE INDEX idx_fulfil_items_fulfillment ON fulfillment_items(fulfillment_id);
CREATE INDEX idx_fulfil_items_order_item  ON fulfillment_items(order_item_id);
```

---

### 3.11 — Drivers & Vehicles

```sql
CREATE TYPE driver_status  AS ENUM ('active', 'on_route', 'off_duty', 'suspended');
CREATE TYPE vehicle_type   AS ENUM ('motorcycle', 'pickup', 'van', 'light_truck', 'heavy_truck');
CREATE TYPE vehicle_status AS ENUM ('in_service', 'idle', 'maintenance', 'retired');

CREATE TABLE vehicles (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  vehicle_code    TEXT NOT NULL UNIQUE,   -- e.g. 'NX-001'
  license_plate   TEXT NOT NULL UNIQUE,
  type            vehicle_type NOT NULL,
  make            TEXT,
  model           TEXT,
  year            SMALLINT,
  capacity_kg     INTEGER NOT NULL,
  status          vehicle_status NOT NULL DEFAULT 'idle',
  current_load_kg INTEGER NOT NULL DEFAULT 0,
  next_service_at DATE,
  notes           TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('vehicles');

CREATE TABLE drivers (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  full_name       TEXT NOT NULL,
  phone           TEXT NOT NULL,
  email           TEXT,
  license_number  TEXT,
  vehicle_id      UUID REFERENCES vehicles(id),
  shift           TEXT,                  -- 'morning' | 'afternoon' | 'night'
  zone_coverage   TEXT,
  status          driver_status NOT NULL DEFAULT 'active',
  rating          NUMERIC(3,2) NOT NULL DEFAULT 5.0,
  total_deliveries INTEGER NOT NULL DEFAULT 0,
  is_active       BOOLEAN NOT NULL DEFAULT TRUE,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('drivers');

CREATE INDEX idx_drivers_status   ON drivers(status);
CREATE INDEX idx_drivers_vehicle  ON drivers(vehicle_id);
```

---

### 3.12 — Routes & Last-Mile Tracking

```sql
CREATE TYPE route_status AS ENUM ('draft', 'dispatched', 'in_progress', 'completed', 'cancelled');
CREATE TYPE stop_status  AS ENUM ('pending', 'in_transit', 'arrived', 'delivered', 'failed', 'skipped');
CREATE TYPE pod_type     AS ENUM ('signature', 'photo', 'signature_and_photo', 'none');
CREATE TYPE stop_type    AS ENUM ('delivery', 'collection', 'exchange');
CREATE TYPE delivery_window AS ENUM ('asap', 'morning', 'afternoon', 'evening', 'specific');
CREATE TYPE failure_reason  AS ENUM (
  'customer_not_home', 'wrong_address', 'refused_delivery',
  'access_issue', 'package_damaged', 'other'
);

CREATE TABLE routes (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  route_code      TEXT NOT NULL UNIQUE,   -- e.g. 'RT-2601'
  driver_id       UUID NOT NULL REFERENCES drivers(id),
  vehicle_id      UUID NOT NULL REFERENCES vehicles(id),
  status          route_status NOT NULL DEFAULT 'draft',
  start_location  TEXT NOT NULL,          -- warehouse or custom
  end_location    TEXT,
  optimization    TEXT NOT NULL DEFAULT 'shortest_distance',
  route_date      DATE NOT NULL,
  dispatched_at   TIMESTAMPTZ,
  started_at      TIMESTAMPTZ,
  completed_at    TIMESTAMPTZ,
  estimated_km    NUMERIC(8,2),
  actual_km       NUMERIC(8,2),
  estimated_mins  INTEGER,
  notes           TEXT,
  created_by      UUID REFERENCES admin_profiles(id),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('routes');

CREATE INDEX idx_routes_driver ON routes(driver_id);
CREATE INDEX idx_routes_date   ON routes(route_date DESC);
CREATE INDEX idx_routes_status ON routes(status);

-- Each stop on a route
CREATE TABLE route_stops (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  route_id         UUID NOT NULL REFERENCES routes(id) ON DELETE CASCADE,
  order_id         UUID NOT NULL REFERENCES orders(id),
  stop_sequence    SMALLINT NOT NULL,      -- position in route (1, 2, 3…)
  stop_type        stop_type NOT NULL DEFAULT 'delivery',
  status           stop_status NOT NULL DEFAULT 'pending',
  delivery_window  delivery_window NOT NULL DEFAULT 'asap',
  window_from      TIME,
  window_to        TIME,
  address          TEXT NOT NULL,
  contact_name     TEXT,
  contact_phone    TEXT,
  special_instructions TEXT,
  pod_required     pod_type NOT NULL DEFAULT 'signature',
  contact_on_arrival TEXT,                -- 'call' | 'sms' | 'none'
  arrived_at       TIMESTAMPTZ,
  completed_at     TIMESTAMPTZ,
  failure_reason   failure_reason,
  failure_notes    TEXT,
  reattempt_at     TIMESTAMPTZ,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

SELECT apply_updated_at('route_stops');

CREATE INDEX idx_route_stops_route    ON route_stops(route_id);
CREATE INDEX idx_route_stops_order    ON route_stops(order_id);
CREATE INDEX idx_route_stops_sequence ON route_stops(route_id, stop_sequence);

-- Proof of delivery records
CREATE TABLE pod_records (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  route_stop_id    UUID NOT NULL REFERENCES route_stops(id) ON DELETE CASCADE,
  order_id         UUID NOT NULL REFERENCES orders(id),
  driver_id        UUID NOT NULL REFERENCES drivers(id),
  pod_type         pod_type NOT NULL,
  received_by      TEXT,
  signature_url    TEXT,                  -- Supabase Storage URL
  photo_url        TEXT,                  -- Supabase Storage URL
  delivery_notes   TEXT,
  recorded_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  lat              NUMERIC(10,7),         -- GPS lat at capture
  lng              NUMERIC(10,7)          -- GPS lng at capture
);

CREATE INDEX idx_pod_stop    ON pod_records(route_stop_id);
CREATE INDEX idx_pod_order   ON pod_records(order_id);
CREATE INDEX idx_pod_driver  ON pod_records(driver_id);
CREATE INDEX idx_pod_date    ON pod_records(recorded_at DESC);
```

---

### 3.13 — Useful Views

```sql
-- Orders with customer name and item count
CREATE VIEW v_orders_summary AS
SELECT
  o.id,
  o.order_number,
  c.full_name        AS customer_name,
  c.email            AS customer_email,
  o.payment_status,
  o.fulfillment_status,
  o.priority,
  o.total,
  o.currency,
  o.carrier,
  o.tracking_number,
  o.placed_at,
  COUNT(oi.id)       AS item_count
FROM orders o
JOIN customers c  ON c.id = o.customer_id
LEFT JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, c.full_name, c.email;

-- Inventory SOH across all locations
CREATE VIEW v_inventory_soh AS
SELECT
  p.id            AS product_id,
  p.name          AS product_name,
  p.sku,
  p.category,
  p.price,
  p.reorder_point,
  sl.name         AS location_name,
  sl.code         AS location_code,
  il.quantity_on_hand,
  il.quantity_reserved,
  il.quantity_available,
  (p.price * il.quantity_available) AS stock_value,
  CASE
    WHEN il.quantity_available = 0        THEN 'out_of_stock'
    WHEN il.quantity_available <= p.reorder_point THEN 'low_stock'
    ELSE 'in_stock'
  END             AS stock_status
FROM products p
JOIN inventory_levels il ON il.product_id = p.id
JOIN stock_locations sl  ON sl.id = il.location_id
WHERE p.status != 'archived';

-- Active routes with driver info
CREATE VIEW v_active_routes AS
SELECT
  r.id,
  r.route_code,
  d.full_name     AS driver_name,
  d.phone         AS driver_phone,
  v.vehicle_code,
  v.type          AS vehicle_type,
  r.status,
  r.route_date,
  r.dispatched_at,
  r.started_at,
  COUNT(rs.id)                                           AS total_stops,
  COUNT(rs.id) FILTER (WHERE rs.status = 'delivered')   AS stops_delivered,
  COUNT(rs.id) FILTER (WHERE rs.status = 'failed')      AS stops_failed,
  COUNT(rs.id) FILTER (WHERE rs.status = 'in_transit')  AS stops_in_transit
FROM routes r
JOIN drivers d  ON d.id = r.driver_id
JOIN vehicles v ON v.id = r.vehicle_id
LEFT JOIN route_stops rs ON rs.route_id = r.id
WHERE r.status IN ('dispatched', 'in_progress')
GROUP BY r.id, d.full_name, d.phone, v.vehicle_code, v.type;
```

---

### 3.14 — Seed Data (Development)

```sql
-- Insert default stock locations
INSERT INTO stock_locations (name, code, address, city, country) VALUES
  ('Warehouse A — Dhaka',      'WH-A', 'Mirpur Industrial Area', 'Dhaka',      'BD'),
  ('Warehouse B — Chittagong', 'WH-B', 'CEPZ, Chittagong',       'Chittagong', 'BD');

-- Insert sample admin (password managed by Supabase Auth)
-- Run AFTER creating the auth user manually in Supabase Dashboard
INSERT INTO admin_profiles (user_id, full_name, email, role)
VALUES ('YOUR-AUTH-USER-UUID', 'Alex Rahman', 'admin@nexus.com', 'super_admin');
```

---

## 4. TypeScript Type Definitions

> Place in `src/types/` — one file per domain.

### `src/types/database.ts` — Auto-generated from Supabase

```bash
# Generate types automatically from your Supabase project
npx supabase gen types typescript --project-id YOUR_PROJECT_ID > src/types/database.ts
```

### `src/types/orders.ts`

```typescript
import type { Database } from './database';

export type Order = Database['public']['Tables']['orders']['Row'];
export type OrderInsert = Database['public']['Tables']['orders']['Insert'];
export type OrderUpdate = Database['public']['Tables']['orders']['Update'];

export type OrderItem = Database['public']['Tables']['order_items']['Row'];
export type OrderStatusHistory = Database['public']['Tables']['order_status_history']['Row'];

export type PaymentStatus    = Database['public']['Enums']['payment_status'];
export type FulfillmentStatus = Database['public']['Enums']['fulfillment_status'];
export type OrderPriority    = Database['public']['Enums']['order_priority'];

// Extended type with joins
export interface OrderWithDetails extends Order {
  customer: {
    id: string;
    full_name: string;
    email: string;
    phone: string | null;
  };
  items: (OrderItem & {
    product: {
      name: string;
      sku: string;
      images: string[];
    };
  })[];
  status_history: OrderStatusHistory[];
  fulfillment_status_label: string;
  item_count: number;
}

export interface OrderFilters {
  search?: string;
  payment_status?: PaymentStatus;
  fulfillment_status?: FulfillmentStatus;
  priority?: OrderPriority;
  date_from?: string;
  date_to?: string;
  customer_id?: string;
}

export interface OrdersResponse {
  data: OrderWithDetails[];
  total: number;
  page: number;
  per_page: number;
  total_pages: number;
}
```

### `src/types/returns.ts`

```typescript
import type { Database } from './database';

export type Return = Database['public']['Tables']['returns']['Row'];
export type ReturnInsert = Database['public']['Tables']['returns']['Insert'];
export type ReturnItem = Database['public']['Tables']['return_items']['Row'];
export type ReturnHistory = Database['public']['Tables']['return_history']['Row'];

export type ReturnStatus     = Database['public']['Enums']['return_status'];
export type ReturnResolution = Database['public']['Enums']['return_resolution'];
export type ItemCondition    = Database['public']['Enums']['item_condition'];

export interface ReturnWithDetails extends Return {
  order: { order_number: string };
  customer: { full_name: string; email: string };
  items: (ReturnItem & { product: { name: string; sku: string } })[];
  history: ReturnHistory[];
}

// Form data for creating a new return
export interface CreateReturnInput {
  order_id: string;
  items: {
    order_item_id: string;
    product_id: string;
    quantity: number;
    reason: string;
    condition: ItemCondition;
  }[];
  resolution: ReturnResolution;
  shipping_paid_by: 'customer' | 'store';
  warehouse_notes?: string;
}
```

### `src/types/inventory.ts`

```typescript
import type { Database } from './database';

export type Product         = Database['public']['Tables']['products']['Row'];
export type ProductInsert   = Database['public']['Tables']['products']['Insert'];
export type InventoryLevel  = Database['public']['Tables']['inventory_levels']['Row'];
export type InventoryMovement = Database['public']['Tables']['inventory_movements']['Row'];
export type CycleCount      = Database['public']['Tables']['cycle_counts']['Row'];
export type CycleCountItem  = Database['public']['Tables']['cycle_count_items']['Row'];
export type Bundle          = Database['public']['Tables']['bundles']['Row'];
export type BundleItem      = Database['public']['Tables']['bundle_items']['Row'];
export type StockLocation   = Database['public']['Tables']['stock_locations']['Row'];

export type MovementType    = Database['public']['Enums']['movement_type'];
export type ProductCategory = Database['public']['Enums']['product_category'];
export type ProductStatus   = Database['public']['Enums']['product_status'];
export type CycleCountStatus = Database['public']['Enums']['cycle_count_status'];

// SOH view type
export interface SOHRecord {
  product_id: string;
  product_name: string;
  sku: string;
  category: ProductCategory;
  price: number;
  reorder_point: number;
  location_name: string;
  location_code: string;
  quantity_on_hand: number;
  quantity_reserved: number;
  quantity_available: number;
  stock_value: number;
  stock_status: 'in_stock' | 'low_stock' | 'out_of_stock';
}

export interface AdjustStockInput {
  product_id: string;
  location_id: string;
  adjustment_type: 'add' | 'remove' | 'set';
  quantity: number;
  reason: string;
  reference_type?: string;
  reference_id?: string;
}

export interface CreateProductInput {
  name: string;
  sku: string;
  description?: string;
  category: ProductCategory;
  status: ProductStatus;
  price: number;
  compare_at_price?: number;
  cost_price?: number;
  weight_grams?: number;
  reorder_point?: number;
  tags?: string[];
  initial_stock?: number;
  location_id?: string;
}

export interface BundleWithItems extends Bundle {
  product: Product;
  items: (BundleItem & { product: Product })[];
}
```

### `src/types/logistics.ts`

```typescript
import type { Database } from './database';

export type Driver      = Database['public']['Tables']['drivers']['Row'];
export type Vehicle     = Database['public']['Tables']['vehicles']['Row'];
export type Route       = Database['public']['Tables']['routes']['Row'];
export type RouteStop   = Database['public']['Tables']['route_stops']['Row'];
export type PODRecord   = Database['public']['Tables']['pod_records']['Row'];

export type DriverStatus  = Database['public']['Enums']['driver_status'];
export type VehicleType   = Database['public']['Enums']['vehicle_type'];
export type RouteStatus   = Database['public']['Enums']['route_status'];
export type StopStatus    = Database['public']['Enums']['stop_status'];
export type PODType       = Database['public']['Enums']['pod_type'];
export type FailureReason = Database['public']['Enums']['failure_reason'];

export interface RouteWithDetails extends Route {
  driver: Pick<Driver, 'id' | 'full_name' | 'phone' | 'rating'>;
  vehicle: Pick<Vehicle, 'id' | 'vehicle_code' | 'type' | 'capacity_kg'>;
  stops: (RouteStop & {
    order: { order_number: string };
    customer: { full_name: string };
    pod?: PODRecord;
  })[];
  stats: {
    total: number;
    delivered: number;
    failed: number;
    in_transit: number;
    completion_pct: number;
  };
}

export interface CreateRouteInput {
  driver_id: string;
  vehicle_id: string;
  route_date: string;
  start_location: string;
  end_location?: string;
  optimization: 'shortest_distance' | 'fastest_time' | 'priority_first';
  stops: {
    order_id: string;
    stop_type: 'delivery' | 'collection' | 'exchange';
    stop_sequence: number;
    delivery_window: string;
    pod_required: PODType;
    contact_on_arrival: string;
    special_instructions?: string;
  }[];
}

export interface RecordPODInput {
  route_stop_id: string;
  order_id: string;
  driver_id: string;
  pod_type: PODType;
  received_by?: string;
  signature_url?: string;
  photo_url?: string;
  delivery_notes?: string;
  lat?: number;
  lng?: number;
}

export interface FailDeliveryInput {
  route_stop_id: string;
  failure_reason: FailureReason;
  failure_notes?: string;
  reattempt_at?: string;
}
```

---

## 5. Drizzle ORM Schema

> Place in `src/db/schema/` — mirrors SQL schema for type-safe queries.

### `src/db/schema/orders.ts`

```typescript
import {
  pgTable, uuid, text, numeric, integer, boolean,
  timestamp, pgEnum, index, uniqueIndex
} from 'drizzle-orm/pg-core';
import { customers } from './customers';
import { adminProfiles } from './admins';

export const paymentStatusEnum = pgEnum('payment_status', [
  'pending', 'paid', 'failed', 'refunded', 'partially_refunded'
]);

export const fulfillmentStatusEnum = pgEnum('fulfillment_status', [
  'unfulfilled', 'processing', 'partially_fulfilled',
  'shipped', 'delivered', 'cancelled'
]);

export const orders = pgTable('orders', {
  id:                uuid('id').primaryKey().defaultRandom(),
  orderNumber:       text('order_number').notNull().unique(),
  customerId:        uuid('customer_id').notNull().references(() => customers.id),
  paymentStatus:     paymentStatusEnum('payment_status').notNull().default('pending'),
  fulfillmentStatus: fulfillmentStatusEnum('fulfillment_status').notNull().default('unfulfilled'),
  subtotal:          numeric('subtotal', { precision: 12, scale: 2 }).notNull(),
  shippingCost:      numeric('shipping_cost', { precision: 10, scale: 2 }).notNull().default('0'),
  taxAmount:         numeric('tax_amount', { precision: 10, scale: 2 }).notNull().default('0'),
  discountAmount:    numeric('discount_amount', { precision: 10, scale: 2 }).notNull().default('0'),
  total:             numeric('total', { precision: 12, scale: 2 }).notNull(),
  currency:          text('currency').notNull().default('USD'),
  shippingAddress1:  text('shipping_address1'),
  shippingCity:      text('shipping_city'),
  shippingCountry:   text('shipping_country'),
  carrier:           text('carrier'),
  trackingNumber:    text('tracking_number'),
  stripePaymentId:   text('stripe_payment_id'),
  internalNotes:     text('internal_notes'),
  placedAt:          timestamp('placed_at', { withTimezone: true }).notNull().defaultNow(),
  createdAt:         timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt:         timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  customerIdx:    index('idx_orders_customer').on(t.customerId),
  paymentIdx:     index('idx_orders_payment').on(t.paymentStatus),
  fulfillmentIdx: index('idx_orders_fulfillment').on(t.fulfillmentStatus),
  placedAtIdx:    index('idx_orders_placed_at').on(t.placedAt),
}));

export const orderItems = pgTable('order_items', {
  id:          uuid('id').primaryKey().defaultRandom(),
  orderId:     uuid('order_id').notNull().references(() => orders.id, { onDelete: 'cascade' }),
  productName: text('product_name').notNull(),
  sku:         text('sku').notNull(),
  quantity:    integer('quantity').notNull(),
  unitPrice:   numeric('unit_price', { precision: 10, scale: 2 }).notNull(),
  fulfilledQty: integer('fulfilled_qty').notNull().default(0),
  returnedQty:  integer('returned_qty').notNull().default(0),
}, (t) => ({
  orderIdx: index('idx_order_items_order').on(t.orderId),
}));
```

### `src/db/index.ts`

```typescript
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as ordersSchema    from './schema/orders';
import * as inventorySchema from './schema/inventory';
import * as logisticsSchema from './schema/logistics';
import * as returnsSchema   from './schema/returns';
import * as customersSchema from './schema/customers';

const connectionString = process.env.DATABASE_URL!;
const client = postgres(connectionString);

export const db = drizzle(client, {
  schema: {
    ...ordersSchema,
    ...inventorySchema,
    ...logisticsSchema,
    ...returnsSchema,
    ...customersSchema,
  },
});

export type DB = typeof db;
```

---

## 6. Server Actions & API Logic

> Place in `src/actions/` — all use Next.js 15 Server Actions.

### `src/actions/orders.ts`

```typescript
'use server';

import { db } from '@db/index';
import { orders, orderItems, orderStatusHistory } from '@db/schema/orders';
import { eq, desc, and, like, sql, count } from 'drizzle-orm';
import { createClient } from '@/lib/supabase/server';
import { revalidatePath } from 'next/cache';
import type { OrderFilters, OrderUpdate } from '@types/orders';
import { z } from 'zod';

const updateOrderSchema = z.object({
  payment_status:     z.enum(['pending','paid','failed','refunded','partially_refunded']).optional(),
  fulfillment_status: z.enum(['unfulfilled','processing','partially_fulfilled','shipped','delivered','cancelled']).optional(),
  carrier:            z.string().optional(),
  tracking_number:    z.string().optional(),
  internal_notes:     z.string().optional(),
  shipping_address1:  z.string().optional(),
  shipping_city:      z.string().optional(),
  shipping_country:   z.string().optional(),
});

export async function getOrders(
  filters: OrderFilters = {},
  page = 1,
  perPage = 10
) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  let query = supabase
    .from('v_orders_summary')
    .select('*', { count: 'exact' });

  if (filters.search) {
    query = query.or(
      `order_number.ilike.%${filters.search}%,customer_name.ilike.%${filters.search}%,customer_email.ilike.%${filters.search}%`
    );
  }
  if (filters.payment_status)    query = query.eq('payment_status', filters.payment_status);
  if (filters.fulfillment_status) query = query.eq('fulfillment_status', filters.fulfillment_status);
  if (filters.date_from)         query = query.gte('placed_at', filters.date_from);
  if (filters.date_to)           query = query.lte('placed_at', filters.date_to);

  const from = (page - 1) * perPage;
  query = query.order('placed_at', { ascending: false }).range(from, from + perPage - 1);

  const { data, error, count: total } = await query;
  if (error) throw error;

  return {
    data: data ?? [],
    total: total ?? 0,
    page,
    per_page: perPage,
    total_pages: Math.ceil((total ?? 0) / perPage),
  };
}

export async function getOrderById(orderId: string) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  const { data, error } = await supabase
    .from('orders')
    .select(`
      *,
      customer:customers(id, full_name, email, phone, address_line1, city, country),
      items:order_items(
        *,
        product:products(name, sku, images)
      ),
      status_history:order_status_history(* )
    `)
    .eq('id', orderId)
    .single();

  if (error) throw error;
  return data;
}

export async function updateOrder(orderId: string, updates: OrderUpdate) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  const validated = updateOrderSchema.parse(updates);

  // Get current order for status history
  const { data: current } = await supabase
    .from('orders')
    .select('fulfillment_status, payment_status')
    .eq('id', orderId)
    .single();

  const { data, error } = await supabase
    .from('orders')
    .update({ ...validated, updated_at: new Date().toISOString() })
    .eq('id', orderId)
    .select()
    .single();

  if (error) throw error;

  // Record status change in history
  if (
    validated.fulfillment_status &&
    validated.fulfillment_status !== current?.fulfillment_status
  ) {
    await supabase.from('order_status_history').insert({
      order_id:   orderId,
      from_status: current?.fulfillment_status,
      to_status:   validated.fulfillment_status,
      changed_by:  user.id,
    });
  }

  revalidatePath('/orders');
  revalidatePath(`/orders/${orderId}`);
  return data;
}

export async function cancelOrder(orderId: string, reason?: string) {
  return updateOrder(orderId, {
    fulfillment_status: 'cancelled',
    payment_status: 'failed',
    internal_notes: reason,
  } as OrderUpdate);
}
```

### `src/actions/returns.ts`

```typescript
'use server';

import { createClient } from '@/lib/supabase/server';
import { revalidatePath } from 'next/cache';
import type { CreateReturnInput } from '@types/returns';
import { z } from 'zod';

const createReturnSchema = z.object({
  order_id:        z.string().uuid(),
  items:           z.array(z.object({
    order_item_id: z.string().uuid(),
    product_id:    z.string().uuid(),
    quantity:      z.number().int().positive(),
    reason:        z.string().min(1),
    condition:     z.enum(['unopened','like_new','used_good','used_damaged','not_received']),
  })).min(1),
  resolution:       z.enum(['refund','store_credit','exchange','repair','no_action']),
  shipping_paid_by: z.enum(['customer','store']),
  warehouse_notes:  z.string().optional(),
});

function generateRMANumber(): string {
  const year = new Date().getFullYear().toString().slice(-2);
  const random = Math.floor(1000 + Math.random() * 9000);
  return `RMA-${year}${random}`;
}

export async function createReturn(input: CreateReturnInput) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  const validated = createReturnSchema.parse(input);

  // Get order to link customer_id
  const { data: order } = await supabase
    .from('orders')
    .select('customer_id, order_number')
    .eq('id', validated.order_id)
    .single();

  if (!order) throw new Error('Order not found');

  // Create the return record
  const { data: returnRecord, error: returnError } = await supabase
    .from('returns')
    .insert({
      rma_number:       generateRMANumber(),
      order_id:         validated.order_id,
      customer_id:      order.customer_id,
      status:           'approved',           // auto-approve
      resolution:       validated.resolution,
      shipping_paid_by: validated.shipping_paid_by,
      warehouse_notes:  validated.warehouse_notes,
      approved_at:      new Date().toISOString(),
    })
    .select()
    .single();

  if (returnError) throw returnError;

  // Insert return items
  const { error: itemsError } = await supabase
    .from('return_items')
    .insert(
      validated.items.map(item => ({
        return_id:     returnRecord.id,
        order_item_id: item.order_item_id,
        product_id:    item.product_id,
        product_name:  '',          // will be joined on read
        sku:           '',
        quantity:      item.quantity,
        reason:        item.reason,
        condition:     item.condition,
      }))
    );

  if (itemsError) throw itemsError;

  // Create initial history entry
  await supabase.from('return_history').insert([
    {
      return_id:    returnRecord.id,
      event:        `Return requested for order ${order.order_number}`,
      performed_by: user.id,
    },
    {
      return_id:    returnRecord.id,
      event:        'RMA approved — awaiting return shipment',
      performed_by: user.id,
    },
  ]);

  revalidatePath('/returns');
  return returnRecord;
}

export async function updateReturnStatus(
  returnId: string,
  status: string,
  notes?: string
) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  const updates: Record<string, unknown> = {
    status,
    updated_at: new Date().toISOString(),
  };

  if (status === 'received')  updates.received_at  = new Date().toISOString();
  if (status === 'resolved')  updates.resolved_at  = new Date().toISOString();

  const { error } = await supabase
    .from('returns')
    .update(updates)
    .eq('id', returnId);

  if (error) throw error;

  // Log to history
  await supabase.from('return_history').insert({
    return_id:    returnId,
    event:        `Status updated to: ${status}`,
    notes,
    performed_by: user.id,
  });

  revalidatePath('/returns');
}
```

### `src/actions/inventory.ts`

```typescript
'use server';

import { createClient } from '@/lib/supabase/server';
import { revalidatePath } from 'next/cache';
import type { AdjustStockInput, CreateProductInput } from '@types/inventory';

export async function getSOH(locationId?: string) {
  const supabase = await createClient();
  let query = supabase.from('v_inventory_soh').select('*');
  if (locationId) query = query.eq('location_code', locationId);
  const { data, error } = await query.order('stock_status');
  if (error) throw error;
  return data;
}

export async function adjustStock(input: AdjustStockInput) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  // Get current level
  const { data: current } = await supabase
    .from('inventory_levels')
    .select('quantity_on_hand')
    .eq('product_id', input.product_id)
    .eq('location_id', input.location_id)
    .single();

  const before = current?.quantity_on_hand ?? 0;
  let after: number;

  switch (input.adjustment_type) {
    case 'add':    after = before + input.quantity; break;
    case 'remove': after = Math.max(0, before - input.quantity); break;
    case 'set':    after = input.quantity; break;
  }

  // Upsert inventory level
  const { error: levelError } = await supabase
    .from('inventory_levels')
    .upsert({
      product_id:        input.product_id,
      location_id:       input.location_id,
      quantity_on_hand:  after,
      updated_at:        new Date().toISOString(),
    }, { onConflict: 'product_id,location_id' });

  if (levelError) throw levelError;

  // Record movement
  await supabase.from('inventory_movements').insert({
    product_id:       input.product_id,
    location_id:      input.location_id,
    movement_type:    'adjustment',
    quantity_change:  after - before,
    quantity_before:  before,
    quantity_after:   after,
    reference_type:   input.reference_type,
    reference_id:     input.reference_id,
    notes:            input.reason,
    performed_by:     user.id,
  });

  revalidatePath('/inventory');
  return { before, after };
}

export async function createProduct(input: CreateProductInput) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  const { data: product, error } = await supabase
    .from('products')
    .insert({
      name:             input.name,
      sku:              input.sku,
      description:      input.description,
      category:         input.category,
      status:           input.status,
      price:            input.price,
      compare_at_price: input.compare_at_price,
      cost_price:       input.cost_price,
      weight_grams:     input.weight_grams,
      reorder_point:    input.reorder_point ?? 10,
      tags:             input.tags ?? [],
      created_by:       user.id,
    })
    .select()
    .single();

  if (error) throw error;

  // Set initial stock if provided
  if (input.initial_stock && input.location_id && product) {
    await adjustStock({
      product_id:       product.id,
      location_id:      input.location_id,
      adjustment_type:  'set',
      quantity:         input.initial_stock,
      reason:           'Initial stock on product creation',
    });
  }

  revalidatePath('/inventory');
  return product;
}

export async function getInventoryMovements(
  productId?: string,
  limit = 50,
  offset = 0
) {
  const supabase = await createClient();
  let query = supabase
    .from('inventory_movements')
    .select(`
      *,
      product:products(name, sku),
      location:stock_locations(name, code),
      performed_by_admin:admin_profiles(full_name)
    `, { count: 'exact' })
    .order('performed_at', { ascending: false })
    .range(offset, offset + limit - 1);

  if (productId) query = query.eq('product_id', productId);

  const { data, error, count } = await query;
  if (error) throw error;
  return { data: data ?? [], total: count ?? 0 };
}
```

### `src/actions/logistics.ts`

```typescript
'use server';

import { createClient } from '@/lib/supabase/server';
import { revalidatePath } from 'next/cache';
import type { CreateRouteInput, RecordPODInput, FailDeliveryInput } from '@types/logistics';

function generateRouteCode(): string {
  const ts = Date.now().toString().slice(-4);
  return `RT-${new Date().getFullYear().toString().slice(-2)}${ts}`;
}

export async function createRoute(input: CreateRouteInput) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  const { data: route, error: routeError } = await supabase
    .from('routes')
    .insert({
      route_code:      generateRouteCode(),
      driver_id:       input.driver_id,
      vehicle_id:      input.vehicle_id,
      route_date:      input.route_date,
      start_location:  input.start_location,
      end_location:    input.end_location,
      optimization:    input.optimization,
      status:          'dispatched',
      dispatched_at:   new Date().toISOString(),
      created_by:      user.id,
    })
    .select()
    .single();

  if (routeError) throw routeError;

  // Insert all stops
  const stopsToInsert = input.stops.map(stop => ({
    route_id:             route.id,
    order_id:             stop.order_id,
    stop_type:            stop.stop_type,
    stop_sequence:        stop.stop_sequence,
    delivery_window:      stop.delivery_window,
    pod_required:         stop.pod_required,
    contact_on_arrival:   stop.contact_on_arrival,
    special_instructions: stop.special_instructions,
    status:               'pending' as const,
    address:              '',    // populated from order join
  }));

  const { error: stopsError } = await supabase
    .from('route_stops')
    .insert(stopsToInsert);

  if (stopsError) throw stopsError;

  // Update driver status to on_route
  await supabase
    .from('drivers')
    .update({ status: 'on_route' })
    .eq('id', input.driver_id);

  // Update vehicle load (simplistic — add total order weights)
  await supabase
    .from('vehicles')
    .update({ status: 'in_service' })
    .eq('id', input.vehicle_id);

  revalidatePath('/logistics/dispatch');
  return route;
}

export async function recordPOD(input: RecordPODInput) {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();
  if (!user) throw new Error('Unauthorized');

  // Insert POD record
  const { error: podError } = await supabase
    .from('pod_records')
    .insert({
      route_stop_id: input.route_stop_id,
      order_id:      input.order_id,
      driver_id:     input.driver_id,
      pod_type:      input.pod_type,
      received_by:   input.received_by,
      signature_url: input.signature_url,
      photo_url:     input.photo_url,
      delivery_notes: input.delivery_notes,
      lat:           input.lat,
      lng:           input.lng,
    });

  if (podError) throw podError;

  // Update stop status to delivered
  await supabase
    .from('route_stops')
    .update({
      status:       'delivered',
      completed_at: new Date().toISOString(),
    })
    .eq('id', input.route_stop_id);

  // Update order fulfillment status
  await supabase
    .from('orders')
    .update({
      fulfillment_status: 'delivered',
      updated_at:         new Date().toISOString(),
    })
    .eq('id', input.order_id);

  revalidatePath('/logistics/utl');
  return { success: true };
}

export async function failDelivery(input: FailDeliveryInput) {
  const supabase = await createClient();

  const { error } = await supabase
    .from('route_stops')
    .update({
      status:         'failed',
      failure_reason: input.failure_reason,
      failure_notes:  input.failure_notes,
      reattempt_at:   input.reattempt_at,
      completed_at:   new Date().toISOString(),
    })
    .eq('id', input.route_stop_id);

  if (error) throw error;
  revalidatePath('/logistics/utl');
  return { success: true };
}

export async function getActiveRoutes() {
  const supabase = await createClient();
  const { data, error } = await supabase
    .from('v_active_routes')
    .select('*')
    .order('dispatched_at', { ascending: false });
  if (error) throw error;
  return data;
}

export async function getPODLog(limit = 50, offset = 0) {
  const supabase = await createClient();
  const { data, error, count } = await supabase
    .from('pod_records')
    .select(`
      *,
      order:orders(order_number),
      driver:drivers(full_name),
      stop:route_stops(address)
    `, { count: 'exact' })
    .order('recorded_at', { ascending: false })
    .range(offset, offset + limit - 1);

  if (error) throw error;
  return { data: data ?? [], total: count ?? 0 };
}
```

---

## 7. Row Level Security (RLS) Policies

```sql
-- Enable RLS on all tables
ALTER TABLE admin_profiles     ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders             ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_items        ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_status_history ENABLE ROW LEVEL SECURITY;
ALTER TABLE returns            ENABLE ROW LEVEL SECURITY;
ALTER TABLE return_items       ENABLE ROW LEVEL SECURITY;
ALTER TABLE return_history     ENABLE ROW LEVEL SECURITY;
ALTER TABLE exchanges          ENABLE ROW LEVEL SECURITY;
ALTER TABLE products           ENABLE ROW LEVEL SECURITY;
ALTER TABLE inventory_levels   ENABLE ROW LEVEL SECURITY;
ALTER TABLE inventory_movements ENABLE ROW LEVEL SECURITY;
ALTER TABLE cycle_counts       ENABLE ROW LEVEL SECURITY;
ALTER TABLE routes             ENABLE ROW LEVEL SECURITY;
ALTER TABLE route_stops        ENABLE ROW LEVEL SECURITY;
ALTER TABLE pod_records        ENABLE ROW LEVEL SECURITY;
ALTER TABLE drivers            ENABLE ROW LEVEL SECURITY;
ALTER TABLE vehicles           ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers          ENABLE ROW LEVEL SECURITY;

-- Helper: is the current user an active admin?
CREATE OR REPLACE FUNCTION is_admin()
RETURNS BOOLEAN AS $$
  SELECT EXISTS (
    SELECT 1 FROM admin_profiles
    WHERE user_id = auth.uid() AND is_active = TRUE
  );
$$ LANGUAGE sql SECURITY DEFINER;

-- Helper: get admin role
CREATE OR REPLACE FUNCTION admin_role()
RETURNS TEXT AS $$
  SELECT role::TEXT FROM admin_profiles
  WHERE user_id = auth.uid() AND is_active = TRUE
  LIMIT 1;
$$ LANGUAGE sql SECURITY DEFINER;

-- READ: all active admins can read everything
CREATE POLICY "admins_read_all" ON orders             FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON order_items        FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON customers          FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON products           FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON inventory_levels   FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON inventory_movements FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON returns            FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON return_items       FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON return_history     FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON exchanges          FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON routes             FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON route_stops        FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON pod_records        FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON drivers            FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON vehicles           FOR SELECT USING (is_admin());
CREATE POLICY "admins_read_all" ON cycle_counts       FOR SELECT USING (is_admin());

-- WRITE: only super_admin, admin, manager can modify orders
CREATE POLICY "senior_admins_write_orders" ON orders
  FOR ALL USING (admin_role() IN ('super_admin','admin','manager'));

-- WRITE: warehouse team can adjust inventory
CREATE POLICY "warehouse_write_inventory" ON inventory_levels
  FOR ALL USING (admin_role() IN ('super_admin','admin','manager','warehouse'));

CREATE POLICY "warehouse_write_movements" ON inventory_movements
  FOR ALL USING (admin_role() IN ('super_admin','admin','manager','warehouse'));

-- WRITE: logistics team can manage routes, stops, POD
CREATE POLICY "logistics_write_routes" ON routes
  FOR ALL USING (admin_role() IN ('super_admin','admin','manager','logistics'));

CREATE POLICY "logistics_write_stops" ON route_stops
  FOR ALL USING (admin_role() IN ('super_admin','admin','manager','logistics'));

CREATE POLICY "logistics_write_pod" ON pod_records
  FOR ALL USING (admin_role() IN ('super_admin','admin','manager','logistics'));

-- WRITE: returns can be created/updated by admin, manager
CREATE POLICY "admins_write_returns" ON returns
  FOR ALL USING (admin_role() IN ('super_admin','admin','manager'));

-- Only super_admin can delete
CREATE POLICY "super_admin_delete_orders" ON orders
  FOR DELETE USING (admin_role() = 'super_admin');
```

---

## 8. File & Folder Structure

```
nexus-admin/
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   │   └── page.tsx
│   │   │   └── layout.tsx
│   │   ├── (admin)/
│   │   │   ├── layout.tsx                ← Sidebar + Topbar wrapper
│   │   │   ├── dashboard/page.tsx
│   │   │   ├── orders/
│   │   │   │   ├── page.tsx              ← All Orders
│   │   │   │   ├── [id]/page.tsx         ← Order detail
│   │   │   │   ├── returns/page.tsx
│   │   │   │   ├── exchanges/page.tsx
│   │   │   │   └── partial/page.tsx
│   │   │   ├── inventory/
│   │   │   │   ├── page.tsx              ← SOH
│   │   │   │   ├── new/page.tsx
│   │   │   │   ├── bundles/page.tsx
│   │   │   │   ├── products/page.tsx
│   │   │   │   ├── cycle-count/page.tsx
│   │   │   │   └── movement/page.tsx
│   │   │   ├── logistics/
│   │   │   │   ├── dispatch/page.tsx
│   │   │   │   ├── drivers/page.tsx
│   │   │   │   └── tracking/page.tsx     ← Last-Mile + POD
│   │   │   ├── analytics/page.tsx
│   │   │   ├── reports/page.tsx
│   │   │   └── settings/page.tsx
│   │   ├── api/
│   │   │   ├── stripe/webhook/route.ts
│   │   │   └── reports/export/route.ts
│   │   ├── globals.css
│   │   └── layout.tsx
│   ├── actions/
│   │   ├── orders.ts
│   │   ├── returns.ts
│   │   ├── exchanges.ts
│   │   ├── inventory.ts
│   │   ├── logistics.ts
│   │   ├── reports.ts
│   │   └── auth.ts
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Topbar.tsx
│   │   │   └── MobileMenu.tsx
│   │   ├── orders/
│   │   │   ├── OrdersTable.tsx
│   │   │   ├── OrderViewModal.tsx
│   │   │   ├── OrderEditModal.tsx
│   │   │   └── OrderTimeline.tsx
│   │   ├── returns/
│   │   │   ├── ReturnCard.tsx
│   │   │   ├── CreateReturnModal.tsx
│   │   │   └── ReturnHistory.tsx
│   │   ├── inventory/
│   │   │   ├── SOHTable.tsx
│   │   │   ├── MovementTable.tsx
│   │   │   ├── CycleCountCard.tsx
│   │   │   └── BundleBuilder.tsx
│   │   ├── logistics/
│   │   │   ├── RouteCard.tsx
│   │   │   ├── RouteBuilder.tsx
│   │   │   ├── DriverCard.tsx
│   │   │   ├── VehicleCard.tsx
│   │   │   └── PODCaptureModal.tsx
│   │   └── ui/                           ← shadcn/ui components
│   ├── db/
│   │   ├── index.ts
│   │   └── schema/
│   │       ├── orders.ts
│   │       ├── inventory.ts
│   │       ├── logistics.ts
│   │       ├── returns.ts
│   │       └── customers.ts
│   ├── lib/
│   │   ├── supabase/
│   │   │   ├── client.ts
│   │   │   ├── server.ts
│   │   │   └── middleware.ts
│   │   ├── stripe.ts
│   │   ├── resend.ts
│   │   ├── utils.ts
│   │   └── export.ts                     ← CSV/PDF export helpers
│   ├── types/
│   │   ├── database.ts                   ← Generated from Supabase CLI
│   │   ├── orders.ts
│   │   ├── returns.ts
│   │   ├── inventory.ts
│   │   └── logistics.ts
│   └── middleware.ts                     ← Auth protection
├── supabase/
│   ├── migrations/                       ← Drizzle generated SQL
│   └── config.toml
├── public/
├── .env.local
├── drizzle.config.ts
├── next.config.ts
├── tailwind.config.ts
└── package.json
```

---

## 9. TRAE AI Prompt Library

> All prompts written for **TRAE AI** with explicit tool requirements and context.
> Use them in sequence — each builds on the previous.

---

### PHASE 1 — PROJECT SCAFFOLD

#### Prompt 1.1 — Initialize Project
```
Create a Next.js 15 App Router project with TypeScript strict mode for a logistics admin panel called NEXUS.
Install: @supabase/supabase-js @supabase/ssr drizzle-orm drizzle-kit postgres zod
react-hook-form @hookform/resolvers recharts @tanstack/react-table date-fns resend stripe.
Initialize shadcn/ui. Create folder structure exactly matching:
src/app/(auth)/login, src/app/(admin)/[all modules], src/actions/, src/components/,
src/db/schema/, src/lib/supabase/, src/types/.
Create .env.local with all required variables for Supabase, Stripe, Resend, and App URL.
Create drizzle.config.ts pointing to src/db/schema/*.
```
**Tools:** Next.js 15, TypeScript, Supabase, Drizzle ORM, shadcn/ui

---

#### Prompt 1.2 — Auth Middleware
```
Create Next.js 15 middleware.ts that protects all /app/(admin)/* routes using Supabase SSR auth.
If user is not authenticated, redirect to /login.
Create src/lib/supabase/server.ts and client.ts using @supabase/ssr createServerClient and
createBrowserClient. Read NEXT_PUBLIC_SUPABASE_URL and NEXT_PUBLIC_SUPABASE_ANON_KEY from env.
Create src/lib/supabase/middleware.ts that refreshes the session cookie on every request.
```
**Tools:** Next.js 15 middleware, Supabase SSR, `@supabase/ssr`

---

#### Prompt 1.3 — Login Page
```
Create a full-screen login page at src/app/(auth)/login/page.tsx using Next.js 15 App Router
and Server Actions. UI: centered card, NEXUS logo, email + password inputs, "Sign In" button.
Server action calls supabase.auth.signInWithPassword(). On success redirect to /dashboard.
On failure show inline error message. Use react-hook-form with zod validation schema.
Style with Tailwind CSS, dark navy background gradient, white card with shadow.
```
**Tools:** Next.js 15 Server Actions, Supabase Auth, react-hook-form, zod, Tailwind CSS

---

### PHASE 2 — LAYOUT & NAVIGATION

#### Prompt 2.1 — Admin Layout with Sidebar
```
Create src/app/(admin)/layout.tsx as a Server Component that fetches the current admin profile
from Supabase (table: admin_profiles, joined on auth.uid()).
Create src/components/layout/Sidebar.tsx as a Client Component with:
- Logo at top
- Collapsible sub-menus for: Orders (All Orders, Returns, Exchanges, Partial Fulfillment),
  Inventory (SOH, New Product, Bundles, Products, Cycle Count, Movement),
  Logistics (Dispatch, Drivers, Last-Mile Tracking)
- Analytics and Reports links
- Admin profile at bottom with logout button
- Mobile: hamburger toggle, slide-in overlay sidebar
- Active route highlighted using usePathname from next/navigation
Use Tailwind CSS with dark sidebar (#0f172a) and sky-blue (#0ea5e9) accents.
```
**Tools:** Next.js 15 App Router, Supabase, `usePathname`, Tailwind CSS

---

### PHASE 3 — ORDERS MODULE

#### Prompt 3.1 — Orders Table Page
```
Create src/app/(admin)/orders/page.tsx as a Server Component.
Fetch orders using the getOrders() server action with searchParams as filters.
Accept URL searchParams: search, payment_status, fulfillment_status, page, per_page.
Pass data to a Client Component <OrdersTable />.
OrdersTable features:
- Checkbox column for bulk selection (bulk bar with Export CSV, Update Status, Delete)
- Sortable column headers (click = sort asc/desc, indicator shows ↑ or ↓)
- Status badges with colors: paid=green, failed=red, pending=amber, delivered=green, etc.
- Action column: View (eye icon), Edit (pencil icon), Return (↩ icon), Cancel (trash icon)
- Pagination: rows-per-page selector (5/10/20/50), page buttons with ellipsis
- Inline URL filter synced: changing filters updates URL params without full reload
Type the component with OrdersResponse from src/types/orders.ts.
```
**Tools:** Next.js 15 (Server + Client Components, searchParams, useSearchParams, useRouter), TanStack Table, Tailwind CSS, TypeScript

---

#### Prompt 3.2 — Order View Modal (Full Detail + Timeline)
```
Create src/components/orders/OrderViewModal.tsx as a Client Component.
Props: orderId string, open boolean, onClose callback.
Inside: call getOrderById(orderId) on mount (use React useEffect + useState or SWR).
Render:
1. Header: Order number + fulfillment badge + Print Invoice button + Edit button
2. Two-column info grid: Customer card (name, email, phone, address) | Shipment card (date, payment badge, carrier, tracking number in blue)
3. Order Timeline: 5 steps (Placed → Payment → Fulfillment Created → Shipped → Delivered), each step has a dot (green=done, blue=active, red=failed), label, and timestamp
4. Line items table: product emoji/image, name, SKU, qty, unit price, line total
5. Invoice totals: subtotal, shipping, tax, discount, grand total in bold
6. Internal notes block (amber background) if present
Print button opens a browser print window with styled invoice.
```
**Tools:** Next.js 15 Client Component, Supabase, Tailwind CSS, TypeScript (OrderWithDetails type)

---

#### Prompt 3.3 — Order Edit Modal
```
Create src/components/orders/OrderEditModal.tsx as a Client Component.
Props: order (OrderWithDetails), open boolean, onClose callback, onSave callback.
Form fields (react-hook-form + zod):
- Customer name, email (read-only display, not editable)
- Shipping address (editable)
- Payment status dropdown
- Fulfillment status dropdown
- Carrier dropdown (DHL, FedEx, UPS, local carriers)
- Tracking number input
- Internal notes textarea
Footer buttons: Cancel | Cancel Order (red, confirm dialog) | Save Changes
On save: call updateOrder() Server Action, show success toast, call onSave() to refresh table.
Cancel Order button: confirm dialog → calls cancelOrder() Server Action.
```
**Tools:** Next.js 15 Server Actions, react-hook-form, zod, shadcn/ui Dialog, Tailwind CSS

---

### PHASE 4 — RETURNS / RMA MODULE

#### Prompt 4.1 — Returns Page with RMA Cards
```
Create src/app/(admin)/orders/returns/page.tsx.
Fetch all returns with items, history, and customer from Supabase (ordered by requested_at DESC).
Render a KPI strip: Total Returns, Pending Approval, In Transit, Refunded MTD, Exchanges.
For each return render a <ReturnCard /> component showing:
- RMA number + status badge + resolution badge + refund amount
- Linked order number and customer name + date
- Items list: each item as a small card with emoji, product name, reason, condition
- Return history timeline: date | event description | performed by
- Return tracking number if present
- Action buttons: Approve, Mark Received, Resolve, View Detail
Include "Create Return" button in toolbar → opens <CreateReturnModal />.
```
**Tools:** Next.js 15 Server Component, Supabase, Tailwind CSS, TypeScript (ReturnWithDetails)

---

#### Prompt 4.2 — Create Return Modal (Multi-item, per-item reason)
```
Create src/components/returns/CreateReturnModal.tsx as a Client Component.
Step 1: Order ID input → on blur, fetch order line items from Supabase and display them as checkboxes.
Step 2: Per item, show: checkbox to select, product name, SKU, qty input (1 to max ordered),
return reason select (7 options), condition select (5 options).
Step 3: Resolution select (Refund to Stripe / Store Credit / Exchange / Repair).
Step 4: Shipping paid by (Customer / Store prepaid label).
Step 5: Warehouse notes textarea.
Submit calls createReturn() Server Action with typed CreateReturnInput.
Show success toast with generated RMA number.
Validate that at least one item is selected.
```
**Tools:** Next.js 15 Server Actions, react-hook-form, zod, Supabase, Tailwind CSS

---

### PHASE 5 — INVENTORY MODULE

#### Prompt 5.1 — SOH Table Page
```
Create src/app/(admin)/inventory/page.tsx (Stock on Hand).
Server Component: fetch SOH from v_inventory_soh view grouped by product.
KPI strip: Total SKUs, In Stock count, Low Stock count (available ≤ reorder_point), Out of Stock, Total Inventory Value.
SOH Table columns: Product (with emoji), SKU, Location, On Hand, Reserved, Available (color-coded: red=0, amber=≤reorder, green=healthy), Stock Level progress bar, Status badge, Actions (Adjust, Movement, Details).
Filter: search by name/SKU, filter by location dropdown.
Export CSV button downloads all inventory data.
Row action "Adjust" → opens AdjustStockModal.
Row action "Movement" → links to /inventory/movement?product_id=xxx.
```
**Tools:** Next.js 15, Supabase (v_inventory_soh view), Tailwind CSS, TypeScript (SOHRecord)

---

#### Prompt 5.2 — Inventory Movement Tracker
```
Create src/app/(admin)/inventory/movement/page.tsx.
Server Component: fetch inventory_movements with product, location, performed_by joins.
Support URL searchParams: product_id, type, date_from, date_to, page.
Table columns: Date/Time, Product, SKU, Movement Type (colored badge + icon: 🛒sale ↩return 📥purchase ✏️adjustment 🔀transfer), Qty Change (green + or red −), Before, After, Reference (linked order/PO/RMA), Location, User.
Export to CSV button.
Each product's SOH row in /inventory has a direct link here with product_id pre-filtered.
```
**Tools:** Next.js 15, Supabase, TypeScript (InventoryMovement), Tailwind CSS

---

#### Prompt 5.3 — Cycle Count Module
```
Create src/app/(admin)/inventory/cycle-count/page.tsx.
Server Component: fetch cycle_counts with item counts and variance sums.
KPI: Active, Completed (30 days), Variances Found, Next Scheduled.
Each cycle count renders as a card: name, method badge, scope, assigned team, scheduled date.
Progress bar: items_counted / total_items.
Status badge: scheduled (amber), in_progress (blue), completed (green), cancelled (gray).
Action buttons per card: Start Count (for scheduled), Continue (for in_progress), Export Results.
"New Cycle Count" button opens a modal: name, method (4 options), scope (5 options), location, schedule date, assigned team.
"Continue Count" opens a counting interface: list all items in scope, each row shows expected qty, input for counted qty, notes field, save per row.
On save: update cycle_count_items.counted_quantity, compute variance, mark is_counted=true.
```
**Tools:** Next.js 15, Supabase, react-hook-form, Tailwind CSS, TypeScript

---

### PHASE 6 — LOGISTICS MODULE

#### Prompt 6.1 — Route Builder Page
```
Create src/app/(admin)/logistics/dispatch/page.tsx as a Client Component (needs interactivity).
Tabs: Unassigned Orders | Route Builder | Active Routes.

Tab 1 — Unassigned: fetch orders where fulfillment_status='processing' and no active route_stop.
Table with: checkbox, order ID, customer, address (truncated), weight, priority badge, suggested driver (nearest available with capacity), Assign Driver button, Add to Route Builder button.
"Bulk Assign" button: modal with auto-optimize or manual mode.
"Auto-Route All" button: groups by zone, assigns to drivers.

Tab 2 — Route Builder:
Left panel: driver selector, date, start/end location, optimization radio (3 options).
Below: draggable stop list (shows order ID, customer, address, type, window, POD requirement).
Reorder buttons (↑↓) and remove per stop.
Live calculated est. distance and est. time based on stop count.
Optimize button: reorders stops for shortest distance.
Dispatch button: calls createRoute() Server Action.
Right panel: available unassigned orders to quick-add with + button.

Tab 3 — Active Routes: fetch active routes from v_active_routes view.
Each route: expandable card showing all stops with status, delivery time, POD type.
POD button per in-transit stop.
```
**Tools:** Next.js 15 Client Component, Supabase, Server Actions (createRoute), Tailwind CSS, TypeScript (CreateRouteInput, RouteWithDetails)

---

#### Prompt 6.2 — Last-Mile Tracking & POD Page
```
Create src/app/(admin)/logistics/tracking/page.tsx.
Tabs: Live Routes | POD Log | Fleet | Delivery Analytics.

Tab 1 — Live Routes:
Fetch from v_active_routes, render each as a card.
Per card: driver, vehicle, started time, ETA, progress bar (stops done / total).
Per stop inside card: stop number circle, order ID, customer, address, status icon, delivery time.
For in-transit stops: "📸 POD" button.
For failed stops: "Retry" button + failure reason.
"View All Stops" button: opens route detail modal with full stop table.
"📞 Call Driver" button.

Tab 2 — POD Log:
Table: Order, Driver, Customer, Address, Time, POD Type badge, Status badge, View button.
Filters: search, status, date range.
Export CSV button.

Tab 3 — Fleet:
Vehicle cards: vehicle code, plate, type, driver, capacity, current load bar, utilization %, active trips, next service date.
⚠️ warning on overdue service. Activate / Edit buttons.

Tab 4 — Analytics:
7-day delivery success bar chart (Delivered vs Failed, stacked).
Average time per stop per driver (horizontal bar chart).
Failure reason breakdown (5 colored stat boxes).
```
**Tools:** Next.js 15, Supabase (v_active_routes, pod_records), Recharts, Tailwind CSS, TypeScript

---

#### Prompt 6.3 — POD Capture Modal
```
Create src/components/logistics/PODCaptureModal.tsx as a Client Component.
Props: routeStopId, orderId, driverId, customerName, address, onComplete callback.
3 delivery status buttons: ✅ Delivered | ❌ Failed Attempt | ⚠️ Partial.
For Delivered: receiver name input, signature capture panel (simulated — shows captured state on click), photo capture panel (simulated), delivery notes input.
For Failed: failure reason dropdown (6 options), reattempt schedule dropdown.
On submit: call recordPOD() or failDelivery() Server Action depending on status.
Success: show toast "POD Recorded", call onComplete() to refresh route card.
All inputs validated with zod before submission.
```
**Tools:** Next.js 15 Server Actions (recordPOD, failDelivery), react-hook-form, zod, Tailwind CSS, TypeScript (RecordPODInput, FailDeliveryInput)

---

### PHASE 7 — REPORTS & EXPORT

#### Prompt 7.1 — Reports Page with Export
```
Create src/app/(admin)/reports/page.tsx.
6 report cards: Revenue, Orders, Returns, Inventory, Logistics, Email Report.
Each card: icon, title, description, export buttons (CSV, PDF, Excel, Print).

CSV export: Server Action that queries Supabase and returns data as CSV string.
Response as file download using Response with Content-Disposition: attachment.

PDF export: use @react-pdf/renderer to generate a styled PDF.
Excel export: use xlsx library to generate .xlsx.
Print: opens a new browser window with styled print-only HTML.

Email Report modal: recipient, report type, date range, format, schedule (once/daily/weekly/monthly).
On submit: call Resend API via Server Action to send the report as attachment.

All exports require admin authentication via Supabase session check.
```
**Tools:** Next.js 15 Server Actions, Supabase, `xlsx` library, `@react-pdf/renderer`, Resend, Tailwind CSS

---

### PHASE 8 — ANALYTICS & DASHBOARD

#### Prompt 8.1 — Dashboard with Live KPIs
```
Create src/app/(admin)/dashboard/page.tsx as a Server Component.
Fetch in parallel using Promise.all:
- Total orders + revenue MTD from orders table (aggregate query)
- Open returns count from returns table where status NOT IN ('resolved','cancelled')
- Active drivers count from drivers where status = 'on_route'
- Low stock items count from v_inventory_soh where stock_status = 'low_stock'

Pass to Client Components for charts (Recharts).
Revenue Chart: line chart, last 14 days revenue (query orders grouped by date).
Orders by Status: doughnut chart (grouped count by fulfillment_status).
Recent Orders table: last 5 orders with customer name, total, status badge, clickable to detail.
Low Stock Alerts table: products below reorder point, with available qty highlighted red/amber.
All data refresh via revalidatePath on server actions.
```
**Tools:** Next.js 15 Server Component, Supabase (parallel queries), Recharts (line + doughnut), Tailwind CSS, TypeScript

---

## 10. Deployment Checklist

### Supabase Setup
- [ ] Create Supabase project
- [ ] Run all SQL scripts in order (3.1 → 3.14)
- [ ] Enable Realtime on `routes` and `route_stops` tables
- [ ] Create Storage bucket `pod-evidence` (private)
- [ ] Set Storage policy: admins can upload/read
- [ ] Generate TypeScript types: `npx supabase gen types typescript --project-id YOUR_ID > src/types/database.ts`
- [ ] Create first admin user via Supabase Auth Dashboard
- [ ] Run seed SQL (3.14) with the new user's UUID

### Stripe Setup
- [ ] Create Stripe account and get API keys
- [ ] Configure webhook endpoint: `https://admin.nexus.com/api/stripe/webhook`
- [ ] Subscribe to events: `payment_intent.succeeded`, `payment_intent.failed`, `refund.created`
- [ ] Add `STRIPE_WEBHOOK_SECRET` to env

### Resend Setup
- [ ] Create Resend account
- [ ] Verify sending domain
- [ ] Add `RESEND_API_KEY` to env

### Vercel Deployment
- [ ] Push project to GitHub
- [ ] Import to Vercel
- [ ] Add all env variables in Vercel dashboard
- [ ] Set `DATABASE_URL` to Supabase pooled connection string (port 6543, not 5432)
- [ ] Add `NODE_ENV=production`
- [ ] Deploy — Vercel auto-detects Next.js 15

### Post-Deployment
- [ ] Test login flow
- [ ] Test order view, edit, cancel
- [ ] Test create return
- [ ] Test route creation and POD capture
- [ ] Test CSV export from reports page
- [ ] Verify RLS policies block unauthenticated requests
- [ ] Set up cron job for scheduled email reports (Vercel Cron or Supabase pg_cron)

---

*Generated for NEXUS Admin Panel v3 Final | Stack: Next.js 15 + Supabase + TypeScript + Drizzle ORM*
*Document version: 1.0 | March 2026*
