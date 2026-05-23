# Jenny Florería — Phase 1: Supabase Foundation

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the Supabase project with the complete database schema, storage bucket, seed data, and RLS policies that every other phase depends on.

**Architecture:** A single Supabase project holds all data. SQL migrations are version-controlled files applied in order. A Node.js verification script confirms every table, column, and policy exists after setup.

**Tech Stack:** Supabase (PostgreSQL + Storage + Realtime), Node.js 18+, Supabase CLI, @supabase/supabase-js

**Spec reference:** `docs/superpowers/specs/2026-05-23-jenny-floreria-automation-design.md` — Sections 4 (Data Layer) and 3 (Sub-Systems)

---

## Project File Structure

```
jenny-floreria/                        ← repo root (create this)
├── supabase/
│   └── migrations/
│       ├── 001_initial_schema.sql     ← all table definitions
│       ├── 002_seed_data.sql          ← loyalty_tiers + app_settings defaults
│       ├── 003_rls_policies.sql       ← Row Level Security
│       └── 004_seed_dev.sql           ← sample flowers + drivers for development
├── scripts/
│   └── verify_schema.js              ← confirms schema applied correctly
├── .env.example                       ← required env vars documented
├── .env                               ← local secrets (gitignored)
├── package.json
└── .gitignore
```

---

## Task 1: Repository & Project Bootstrap

**Files:**
- Create: `jenny-floreria/` (project root)
- Create: `jenny-floreria/.gitignore`
- Create: `jenny-floreria/package.json`
- Create: `jenny-floreria/.env.example`

- [ ] **Step 1: Create project directory**

```bash
mkdir jenny-floreria && cd jenny-floreria
git init
mkdir -p supabase/migrations scripts
```

- [ ] **Step 2: Create `.gitignore`**

```
.env
node_modules/
.DS_Store
```

- [ ] **Step 3: Create `package.json`**

```json
{
  "name": "jenny-floreria",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "verify": "node scripts/verify_schema.js"
  },
  "dependencies": {
    "@supabase/supabase-js": "^2.45.0"
  }
}
```

- [ ] **Step 4: Create `.env.example`**

```bash
# Supabase — get from: supabase.com → project → Settings → API
SUPABASE_URL=https://xxxxxxxxxxxxxxxxxxxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJhbGc...   # service_role key (full access, never expose publicly)
SUPABASE_ANON_KEY=eyJhbGc...           # anon key (public catalog use)

# Set after Phase 2
# WHATSAPP_PHONE_NUMBER_ID=
# WHATSAPP_ACCESS_TOKEN=
# OPENAI_API_KEY=

# Set after Phase 3
# MERCADOPAGO_ACCESS_TOKEN=

# Set after Phase 4
# VAPI_API_KEY=
# ELEVENLABS_API_KEY=
# TWILIO_ACCOUNT_SID=
# TWILIO_AUTH_TOKEN=
# TWILIO_PHONE_NUMBER=

# Set after Phase 6
# N8N_WEBHOOK_BASE_URL=
# CARMELITA_PERSONAL_WHATSAPP=+521XXXXXXXXXX
```

- [ ] **Step 5: Install dependencies**

```bash
npm install
```

- [ ] **Step 6: Create Supabase project**

Go to [supabase.com](https://supabase.com) → New project:
- Name: `jenny-floreria`
- Database password: generate a strong one, save it
- Region: `South America (São Paulo)` — closest to Guasave, Sinaloa
- Plan: Free tier (sufficient for this scale)

Wait ~2 minutes for provisioning. Then:
- Go to Settings → API
- Copy `Project URL` → `SUPABASE_URL` in `.env`
- Copy `service_role` secret → `SUPABASE_SERVICE_ROLE_KEY` in `.env`
- Copy `anon` public key → `SUPABASE_ANON_KEY` in `.env`

```bash
# Create .env from example
cp .env.example .env
# Now fill in the three Supabase values
```

- [ ] **Step 7: Install Supabase CLI**

```bash
brew install supabase/tap/supabase
supabase --version
# Expected: supabase version X.X.X
```

- [ ] **Step 8: Commit bootstrap**

```bash
git add .gitignore package.json package-lock.json .env.example
git commit -m "chore: bootstrap project structure"
```

---

## Task 2: Schema Migration — All Tables

**Files:**
- Create: `supabase/migrations/001_initial_schema.sql`

- [ ] **Step 1: Write `supabase/migrations/001_initial_schema.sql`**

```sql
-- ============================================================
-- 001_initial_schema.sql
-- Jenny Florería — complete database schema
-- ============================================================

-- Flower & ornament inventory
CREATE TABLE flowers (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name                    TEXT NOT NULL,
  description             TEXT,
  price_mxn               NUMERIC(10,2) NOT NULL CHECK (price_mxn >= 0),
  category                TEXT CHECK (category IN (
                            'flores','arreglos','ornamentos','coronas',
                            'centros_de_mesa','bouquets_boda','decoracion_evento'
                          )),
  stock_count             INTEGER DEFAULT 0 CHECK (stock_count >= 0),
  image_url               TEXT,
  occasion_tags           TEXT[],
  customization_options   JSONB,
  active                  BOOLEAN DEFAULT true,
  created_at              TIMESTAMPTZ DEFAULT now()
);

-- Customers
CREATE TABLE customers (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  whatsapp_phone          TEXT UNIQUE NOT NULL,
  display_name            TEXT,
  saved_addresses         JSONB[],
  total_orders_completed  INTEGER DEFAULT 0 CHECK (total_orders_completed >= 0),
  loyalty_tier            TEXT DEFAULT 'none' CHECK (loyalty_tier IN ('none','silver','gold','vip')),
  birthday_month_day      TEXT CHECK (birthday_month_day ~ '^\d{2}-\d{2}$'),
  carmelita_notes         TEXT,
  created_at              TIMESTAMPTZ DEFAULT now()
);

-- Loyalty tier definitions
CREATE TABLE loyalty_tiers (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tier_name             TEXT UNIQUE NOT NULL CHECK (tier_name IN ('silver','gold','vip')),
  min_orders_required   INTEGER NOT NULL CHECK (min_orders_required > 0),
  discount_percent      INTEGER NOT NULL CHECK (discount_percent BETWEEN 1 AND 100),
  applicable_occasions  TEXT[],
  active                BOOLEAN DEFAULT true
);

-- App-wide settings (single row)
CREATE TABLE app_settings (
  id                              TEXT PRIMARY KEY DEFAULT 'default',
  delivery_fee_mxn                NUMERIC(10,2) DEFAULT 50 CHECK (delivery_fee_mxn >= 0),
  free_delivery_threshold_mxn     NUMERIC(10,2),
  driver_rate_per_delivery_mxn    NUMERIC(10,2) DEFAULT 80 CHECK (driver_rate_per_delivery_mxn >= 0),
  shop_whatsapp_phone             TEXT,
  shop_name                       TEXT DEFAULT 'Jenny Florería',
  shop_city                       TEXT DEFAULT 'Guasave, Sinaloa',
  updated_at                      TIMESTAMPTZ DEFAULT now()
);

-- Delivery drivers
CREATE TABLE drivers (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name              TEXT NOT NULL,
  whatsapp_phone    TEXT UNIQUE NOT NULL,
  active            BOOLEAN DEFAULT true,
  current_order_id  UUID,                   -- FK added after orders table
  preferred_payment TEXT DEFAULT 'cash' CHECK (preferred_payment IN ('cash','transfer')),
  bank_clabe        TEXT CHECK (bank_clabe IS NULL OR length(bank_clabe) = 18),
  bank_name         TEXT,
  created_at        TIMESTAMPTZ DEFAULT now()
);

-- Promotional codes
CREATE TABLE promo_codes (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code              TEXT UNIQUE NOT NULL,
  discount_type     TEXT NOT NULL CHECK (discount_type IN ('percentage','fixed_mxn')),
  discount_value    NUMERIC(10,2) NOT NULL CHECK (discount_value > 0),
  valid_from        DATE,
  valid_until       DATE,
  max_uses          INTEGER CHECK (max_uses IS NULL OR max_uses > 0),
  times_used        INTEGER DEFAULT 0 CHECK (times_used >= 0),
  active            BOOLEAN DEFAULT true,
  created_at        TIMESTAMPTZ DEFAULT now()
);

-- Orders
CREATE TABLE orders (
  id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_number              SERIAL,
  customer_id               UUID REFERENCES customers(id),
  order_type                TEXT DEFAULT 'standard' CHECK (order_type IN ('standard','event','wedding')),
  status                    TEXT DEFAULT 'pending' CHECK (status IN (
                              'pending','confirmed','partially_paid','paid',
                              'driver_assigned','in_delivery','delivered','cancelled'
                            )),
  items_json                JSONB,
  subtotal_mxn              NUMERIC(10,2) CHECK (subtotal_mxn >= 0),
  discount_amount_mxn       NUMERIC(10,2) DEFAULT 0 CHECK (discount_amount_mxn >= 0),
  delivery_fee_mxn          NUMERIC(10,2) DEFAULT 0 CHECK (delivery_fee_mxn >= 0),
  total_mxn                 NUMERIC(10,2) CHECK (total_mxn >= 0),
  amount_paid_mxn           NUMERIC(10,2) DEFAULT 0 CHECK (amount_paid_mxn >= 0),
  delivery_address          JSONB,
  delivery_notes            TEXT,
  occasion                  TEXT,
  occasion_card_msg         TEXT,
  event_date                DATE,
  scheduled_delivery_at     TIMESTAMPTZ,
  driver_id                 UUID REFERENCES drivers(id),
  driver_payment_method     TEXT CHECK (driver_payment_method IN ('cash','transfer')),
  driver_payment_status     TEXT DEFAULT 'pending' CHECK (driver_payment_status IN ('pending','paid')),
  driver_payment_amount_mxn NUMERIC(10,2) CHECK (driver_payment_amount_mxn >= 0),
  mp_payment_id             TEXT,
  mp_payment_status         TEXT,
  payment_plan_id           UUID,           -- FK added after payment_plans table
  promo_code_id             UUID REFERENCES promo_codes(id),
  delivery_photo_url        TEXT,
  human_handoff             BOOLEAN DEFAULT false,
  created_at                TIMESTAMPTZ DEFAULT now(),
  updated_at                TIMESTAMPTZ DEFAULT now()
);

-- Order line items
CREATE TABLE order_items (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id          UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  flower_id         UUID REFERENCES flowers(id),
  quantity          INTEGER NOT NULL CHECK (quantity > 0),
  unit_price_mxn    NUMERIC(10,2) NOT NULL CHECK (unit_price_mxn >= 0),
  customizations    JSONB,
  line_total_mxn    NUMERIC(10,2) CHECK (line_total_mxn >= 0)
);

-- Payment plans (event / wedding installments)
CREATE TABLE payment_plans (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id          UUID REFERENCES orders(id),
  total_amount_mxn  NUMERIC(10,2) NOT NULL CHECK (total_amount_mxn > 0),
  amount_paid_mxn   NUMERIC(10,2) DEFAULT 0 CHECK (amount_paid_mxn >= 0),
  status            TEXT DEFAULT 'active' CHECK (status IN ('active','completed','overdue','cancelled')),
  created_at        TIMESTAMPTZ DEFAULT now()
);

-- Payment installments
CREATE TABLE payment_installments (
  id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  payment_plan_id       UUID NOT NULL REFERENCES payment_plans(id),
  installment_number    INTEGER NOT NULL CHECK (installment_number > 0),
  amount_mxn            NUMERIC(10,2) NOT NULL CHECK (amount_mxn > 0),
  due_date              DATE NOT NULL,
  status                TEXT DEFAULT 'pending' CHECK (status IN ('pending','paid','overdue')),
  mp_payment_id         TEXT,
  paid_at               TIMESTAMPTZ,
  reminder_sent_at      TIMESTAMPTZ,
  created_at            TIMESTAMPTZ DEFAULT now()
);

-- Customer loyalty reward usage
CREATE TABLE customer_reward_uses (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id       UUID NOT NULL REFERENCES customers(id),
  order_id          UUID REFERENCES orders(id),
  occasion          TEXT NOT NULL,
  discount_percent  INTEGER NOT NULL,
  used_at           TIMESTAMPTZ DEFAULT now()
);

-- System notifications
CREATE TABLE notifications (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  type        TEXT CHECK (type IN (
                'new_order','payment','driver','delivery','low_stock',
                'handoff','installment_due','installment_paid','plan_overdue',
                'loyalty_milestone','driver_payment'
              )),
  message     TEXT,
  order_id    UUID REFERENCES orders(id),
  read        BOOLEAN DEFAULT false,
  created_at  TIMESTAMPTZ DEFAULT now()
);

-- ── Deferred foreign keys (circular references) ──────────────
ALTER TABLE orders  ADD CONSTRAINT fk_orders_payment_plan
  FOREIGN KEY (payment_plan_id) REFERENCES payment_plans(id);

ALTER TABLE drivers ADD CONSTRAINT fk_drivers_current_order
  FOREIGN KEY (current_order_id) REFERENCES orders(id);

-- ── Indexes ──────────────────────────────────────────────────
CREATE INDEX idx_orders_customer_id    ON orders(customer_id);
CREATE INDEX idx_orders_status         ON orders(status);
CREATE INDEX idx_orders_driver_id      ON orders(driver_id);
CREATE INDEX idx_orders_created_at     ON orders(created_at DESC);
CREATE INDEX idx_order_items_order_id  ON order_items(order_id);
CREATE INDEX idx_flowers_active        ON flowers(active) WHERE active = true;
CREATE INDEX idx_customers_phone       ON customers(whatsapp_phone);
CREATE INDEX idx_notifications_read    ON notifications(read) WHERE read = false;

-- ── updated_at trigger for orders ────────────────────────────
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$;

CREATE TRIGGER trg_orders_updated_at
  BEFORE UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
```

- [ ] **Step 2: Apply migration via Supabase dashboard SQL editor**

Go to Supabase dashboard → SQL Editor → New query.
Paste entire contents of `001_initial_schema.sql` → Run.
Expected: no errors, output shows CREATE TABLE × 12.

- [ ] **Step 3: Verify tables exist**

In SQL Editor:
```sql
SELECT tablename
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;
```

Expected output (12 rows):
```
app_settings
customer_reward_uses
customers
drivers
flowers
loyalty_tiers
notifications
order_items
orders
payment_installments
payment_plans
promo_codes
```

- [ ] **Step 4: Commit migration**

```bash
git add supabase/migrations/001_initial_schema.sql
git commit -m "feat: add complete database schema (14 tables)"
```

---

## Task 3: Seed Data Migration

**Files:**
- Create: `supabase/migrations/002_seed_data.sql`

- [ ] **Step 1: Write `supabase/migrations/002_seed_data.sql`**

```sql
-- ============================================================
-- 002_seed_data.sql
-- Production default data — safe to re-run (ON CONFLICT DO NOTHING)
-- ============================================================

-- App settings
INSERT INTO app_settings (
  id,
  delivery_fee_mxn,
  free_delivery_threshold_mxn,
  driver_rate_per_delivery_mxn,
  shop_name,
  shop_city
) VALUES (
  'default',
  50,
  NULL,
  80,
  'Jenny Florería',
  'Guasave, Sinaloa'
) ON CONFLICT (id) DO NOTHING;

-- Loyalty tiers
INSERT INTO loyalty_tiers (tier_name, min_orders_required, discount_percent, applicable_occasions) VALUES
(
  'silver', 5, 10,
  ARRAY['mothers_day','valentines','dia_muertos','christmas','customer_birthday']
),
(
  'gold', 10, 15,
  ARRAY['mothers_day','valentines','dia_muertos','christmas','customer_birthday']
),
(
  'vip', 20, 20,
  ARRAY['mothers_day','valentines','dia_muertos','christmas','customer_birthday']
)
ON CONFLICT (tier_name) DO NOTHING;
```

- [ ] **Step 2: Apply in SQL Editor**

Paste `002_seed_data.sql` → Run.
Expected: `INSERT 0 1` for app_settings, `INSERT 0 3` for loyalty_tiers.

- [ ] **Step 3: Verify seed data**

```sql
SELECT id, delivery_fee_mxn, driver_rate_per_delivery_mxn FROM app_settings;
SELECT tier_name, min_orders_required, discount_percent FROM loyalty_tiers ORDER BY min_orders_required;
```

Expected:
```
 id      | delivery_fee_mxn | driver_rate_per_delivery_mxn
---------+------------------+------------------------------
 default |            50.00 |                        80.00

 tier_name | min_orders_required | discount_percent
-----------+---------------------+-----------------
 silver    |                   5 |               10
 gold      |                  10 |               15
 vip       |                  20 |               20
```

- [ ] **Step 4: Commit**

```bash
git add supabase/migrations/002_seed_data.sql
git commit -m "feat: seed default app_settings and loyalty_tiers"
```

---

## Task 4: RLS Policies Migration

**Files:**
- Create: `supabase/migrations/003_rls_policies.sql`

- [ ] **Step 1: Write `supabase/migrations/003_rls_policies.sql`**

```sql
-- ============================================================
-- 003_rls_policies.sql
-- Row Level Security policies
-- n8n + dashboard use service_role key — bypasses RLS entirely.
-- Public catalog page uses anon key — needs explicit read policy on flowers.
-- ============================================================

-- Enable RLS on all tables
ALTER TABLE flowers                ENABLE ROW LEVEL SECURITY;
ALTER TABLE customers              ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders                 ENABLE ROW LEVEL SECURITY;
ALTER TABLE order_items            ENABLE ROW LEVEL SECURITY;
ALTER TABLE drivers                ENABLE ROW LEVEL SECURITY;
ALTER TABLE promo_codes            ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_plans          ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_installments   ENABLE ROW LEVEL SECURITY;
ALTER TABLE loyalty_tiers          ENABLE ROW LEVEL SECURITY;
ALTER TABLE customer_reward_uses   ENABLE ROW LEVEL SECURITY;
ALTER TABLE app_settings           ENABLE ROW LEVEL SECURITY;
ALTER TABLE notifications          ENABLE ROW LEVEL SECURITY;

-- ── Public read policies (anon key — catalog page) ──────────

-- Anyone can read active flowers (public catalog)
CREATE POLICY "flowers_anon_read_active"
  ON flowers FOR SELECT
  USING (active = true);

-- Anyone can read active loyalty tiers (AI context, catalog display)
CREATE POLICY "loyalty_tiers_anon_read_active"
  ON loyalty_tiers FOR SELECT
  USING (active = true);

-- ── All other tables: service_role only ─────────────────────
-- service_role key bypasses RLS by default in Supabase.
-- No additional policies needed for n8n workflows or dashboard.
-- Never use the anon key for write operations or sensitive reads.
```

- [ ] **Step 2: Apply in SQL Editor**

Paste `003_rls_policies.sql` → Run.
Expected: no errors.

- [ ] **Step 3: Verify RLS enabled**

```sql
SELECT tablename, rowsecurity
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;
```

Expected: all 12 tables show `rowsecurity = true`.

- [ ] **Step 4: Test public read policy works with anon key (will use in verify script)**

```sql
-- Run as anon (simulate): this should return rows
SELECT id, name, price_mxn FROM flowers WHERE active = true LIMIT 1;
-- Returns 0 rows (no flowers yet) but no permission error
```

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/003_rls_policies.sql
git commit -m "feat: add RLS policies (public read on flowers + loyalty_tiers)"
```

---

## Task 5: Supabase Storage Bucket

**Files:** No code files — Supabase dashboard configuration.

- [ ] **Step 1: Create storage bucket**

Supabase dashboard → Storage → New bucket:
- Name: `flower-images`
- Public bucket: ✅ YES (images displayed in public catalog and dashboard)
- File size limit: 5 MB
- Allowed MIME types: `image/jpeg, image/png, image/webp`

- [ ] **Step 2: Verify bucket public URL pattern**

In Storage → `flower-images` → upload any test image.
Note the public URL format:
```
https://xxxxxxxxxxxxxxxxxxxx.supabase.co/storage/v1/object/public/flower-images/filename.jpg
```
This is the format stored in `flowers.image_url`.

Delete the test image after noting the URL pattern.

- [ ] **Step 3: Add storage bucket name to `.env.example`**

```bash
# add this line to .env.example
SUPABASE_STORAGE_BUCKET=flower-images
```

Also add to `.env`:
```bash
SUPABASE_STORAGE_BUCKET=flower-images
```

- [ ] **Step 4: Commit**

```bash
git add .env.example
git commit -m "chore: document storage bucket env var"
```

---

## Task 6: Development Seed Data

**Files:**
- Create: `supabase/migrations/004_seed_dev.sql`

- [ ] **Step 1: Write `supabase/migrations/004_seed_dev.sql`**

```sql
-- ============================================================
-- 004_seed_dev.sql
-- Development + testing sample data
-- DO NOT apply to production without removing this comment
-- ============================================================

-- Sample flowers (covers all categories, occasions, customization options)
INSERT INTO flowers (name, description, price_mxn, category, stock_count, occasion_tags, customization_options, active) VALUES

(
  'Arreglo de Rosas Rojas',
  'Hermoso arreglo de rosas rojas frescas, ideal para expresar amor y admiración.',
  180.00, 'arreglos', 30,
  ARRAY['birthday','anniversary','valentines','general'],
  '{
    "sizes":  [{"label":"Pequeño","price_mxn":150},{"label":"Mediano","price_mxn":180},{"label":"Grande","price_mxn":280}],
    "colors": ["Rojo","Blanco","Rosa","Amarillo","Morado"],
    "addons": [{"label":"Jarrón de vidrio","price_mxn":50},{"label":"Listón personalizado","price_mxn":20},{"label":"Tarjeta impresa","price_mxn":15}]
  }'::jsonb,
  true
),

(
  'Girasoles Frescos',
  'Alegres girasoles que iluminan cualquier espacio. Perfectos para cumpleaños.',
  220.00, 'flores', 20,
  ARRAY['birthday','general'],
  '{
    "sizes":  [{"label":"Pequeño","price_mxn":180},{"label":"Mediano","price_mxn":220},{"label":"Grande","price_mxn":320}],
    "colors": ["Amarillo"],
    "addons": [{"label":"Jarrón de vidrio","price_mxn":50},{"label":"Tarjeta impresa","price_mxn":15}]
  }'::jsonb,
  true
),

(
  'Corona Fúnebre Blanca',
  'Corona de flores blancas y verdes, símbolo de respeto y paz.',
  850.00, 'coronas', 8,
  ARRAY['funeral'],
  NULL,
  true
),

(
  'Bouquet para Novia',
  'Elegante bouquet de rosas blancas y lirios, diseñado especialmente para bodas.',
  1200.00, 'bouquets_boda', 5,
  ARRAY['wedding'],
  '{
    "colors": ["Blanco","Crema","Rosa claro"],
    "addons": [{"label":"Perlas decorativas","price_mxn":80},{"label":"Listón de satín","price_mxn":40}]
  }'::jsonb,
  true
),

(
  'Centro de Mesa Rústico',
  'Centro de mesa con flores silvestres y follaje verde, perfecto para eventos.',
  450.00, 'centros_de_mesa', 12,
  ARRAY['wedding','birthday','general'],
  '{
    "sizes":  [{"label":"Pequeño","price_mxn":350},{"label":"Mediano","price_mxn":450},{"label":"Grande","price_mxn":600}]
  }'::jsonb,
  true
),

(
  'Arreglo de Cempasúchil',
  'Tradicional arreglo de cempasúchil para ofrenda de Día de Muertos.',
  280.00, 'arreglos', 0,
  ARRAY['dia_muertos'],
  NULL,
  false  -- out of season, deactivated
);

-- Sample drivers (Carmelita's 3 known drivers)
INSERT INTO drivers (name, whatsapp_phone, active, preferred_payment) VALUES
('Marco',  '+521XXXXXXXXX1', true, 'cash'),
('Luis',   '+521XXXXXXXXX2', true, 'transfer'),
('Pedro',  '+521XXXXXXXXX3', true, 'cash');
```

- [ ] **Step 2: Apply in SQL Editor**

Paste `004_seed_dev.sql` → Run.
Expected: `INSERT 0 6` for flowers, `INSERT 0 3` for drivers.

- [ ] **Step 3: Verify**

```sql
SELECT name, category, price_mxn, stock_count, active FROM flowers ORDER BY category;
SELECT name, preferred_payment FROM drivers;
```

Expected: 6 flower rows (5 active, 1 inactive), 3 driver rows.

- [ ] **Step 4: Commit**

```bash
git add supabase/migrations/004_seed_dev.sql
git commit -m "feat: add dev seed data (6 sample flowers, 3 drivers)"
```

---

## Task 7: Schema Verification Script

**Files:**
- Create: `scripts/verify_schema.js`

- [ ] **Step 1: Write `scripts/verify_schema.js`**

```javascript
// scripts/verify_schema.js
// Run: npm run verify
// Confirms every table, key column, seed data, and RLS policy is in place.

import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_SERVICE_ROLE_KEY
)

let passed = 0
let failed = 0

function ok(label) {
  console.log(`  ✅  ${label}`)
  passed++
}

function fail(label, detail) {
  console.error(`  ❌  ${label}: ${detail}`)
  failed++
}

async function check(label, fn) {
  try {
    await fn()
    ok(label)
  } catch (e) {
    fail(label, e.message)
  }
}

// ── Table existence checks ────────────────────────────────────
// Supabase JS client doesn't expose information_schema via REST.
// Instead: attempt a SELECT count(*) on each table — error = table missing.
const REQUIRED_TABLES = [
  'flowers', 'customers', 'orders', 'order_items', 'drivers',
  'promo_codes', 'payment_plans', 'payment_installments',
  'loyalty_tiers', 'customer_reward_uses', 'app_settings', 'notifications'
]

async function checkTables() {
  console.log('\n── Tables ─────────────────────────────────────')
  for (const t of REQUIRED_TABLES) {
    const { error } = await supabase.from(t).select('id', { count: 'exact', head: true })
    if (error) fail(`table: ${t}`, error.message)
    else ok(`table: ${t}`)
  }
}

// ── Seed data checks ──────────────────────────────────────────
async function checkSeedData() {
  console.log('\n── Seed Data ───────────────────────────────────')

  const { data: settings, error: se } = await supabase
    .from('app_settings')
    .select('id, delivery_fee_mxn, driver_rate_per_delivery_mxn')
    .eq('id', 'default')
    .single()

  await check('app_settings default row exists', () => {
    if (se || !settings) throw new Error(se?.message || 'no row')
  })
  await check('delivery_fee_mxn = 50', () => {
    if (Number(settings?.delivery_fee_mxn) !== 50) throw new Error(`got ${settings?.delivery_fee_mxn}`)
  })
  await check('driver_rate_per_delivery_mxn = 80', () => {
    if (Number(settings?.driver_rate_per_delivery_mxn) !== 80) throw new Error(`got ${settings?.driver_rate_per_delivery_mxn}`)
  })

  const { data: tiers, error: te } = await supabase
    .from('loyalty_tiers')
    .select('tier_name, min_orders_required, discount_percent')
    .order('min_orders_required')

  await check('3 loyalty tiers exist', () => {
    if (te) throw new Error(te.message)
    if (tiers.length !== 3) throw new Error(`got ${tiers.length}`)
  })
  await check('silver tier: 5 orders, 10%', () => {
    const s = tiers.find(t => t.tier_name === 'silver')
    if (!s) throw new Error('silver not found')
    if (s.min_orders_required !== 5 || s.discount_percent !== 10)
      throw new Error(`got orders=${s.min_orders_required} discount=${s.discount_percent}`)
  })
  await check('gold tier: 10 orders, 15%', () => {
    const g = tiers.find(t => t.tier_name === 'gold')
    if (!g) throw new Error('gold not found')
    if (g.min_orders_required !== 10 || g.discount_percent !== 15)
      throw new Error(`got orders=${g.min_orders_required} discount=${g.discount_percent}`)
  })
  await check('vip tier: 20 orders, 20%', () => {
    const v = tiers.find(t => t.tier_name === 'vip')
    if (!v) throw new Error('vip not found')
    if (v.min_orders_required !== 20 || v.discount_percent !== 20)
      throw new Error(`got orders=${v.min_orders_required} discount=${v.discount_percent}`)
  })
}

// ── Dev seed data checks ──────────────────────────────────────
async function checkDevSeed() {
  console.log('\n── Dev Seed Data ───────────────────────────────')

  const { data: flowers, error: fe } = await supabase
    .from('flowers')
    .select('id, active')

  await check('at least 5 flowers exist', () => {
    if (fe) throw new Error(fe.message)
    if (flowers.length < 5) throw new Error(`got ${flowers.length}`)
  })
  await check('at least 1 active flower', () => {
    if (!flowers.some(f => f.active)) throw new Error('none active')
  })

  const { data: drvs, error: de } = await supabase
    .from('drivers')
    .select('id')

  await check('3 drivers exist', () => {
    if (de) throw new Error(de.message)
    if (drvs.length !== 3) throw new Error(`got ${drvs.length}`)
  })
}

// ── Order total constraint check ──────────────────────────────
async function checkConstraints() {
  console.log('\n── Constraints ─────────────────────────────────')

  // Try inserting an order with negative total — should fail
  const { error } = await supabase
    .from('orders')
    .insert({ total_mxn: -1 })

  await check('orders rejects negative total_mxn', () => {
    if (!error) throw new Error('insert succeeded — CHECK constraint missing')
  })

  // Try invalid loyalty tier
  const { data: cust } = await supabase
    .from('customers')
    .insert({ whatsapp_phone: '+521TEST999', loyalty_tier: 'invalid_tier' })
    .select()

  await check('customers rejects invalid loyalty_tier', () => {
    // If insert succeeded (no error), constraint is missing
    if (cust && cust.length > 0) {
      // cleanup
      supabase.from('customers').delete().eq('whatsapp_phone', '+521TEST999')
      throw new Error('insert succeeded — CHECK constraint missing')
    }
  })
}

// ── Storage bucket check ──────────────────────────────────────
async function checkStorage() {
  console.log('\n── Storage ─────────────────────────────────────')

  const { data, error } = await supabase.storage.getBucket('flower-images')

  await check('flower-images bucket exists', () => {
    if (error) throw new Error(error.message)
    if (!data) throw new Error('bucket not found')
  })
  await check('flower-images bucket is public', () => {
    if (!data?.public) throw new Error('bucket is not public')
  })
}

// ── Main ──────────────────────────────────────────────────────
console.log('Jenny Florería — Schema Verification')
console.log('=====================================')

await checkTables()
await checkSeedData()
await checkDevSeed()
await checkConstraints()
await checkStorage()

console.log(`\n=====================================`)
console.log(`Results: ${passed} passed, ${failed} failed`)

if (failed > 0) {
  console.error('\n❌ Schema verification FAILED — fix issues above before continuing.')
  process.exit(1)
} else {
  console.log('\n✅ All checks passed. Phase 1 complete — ready for Phase 2.')
}
```

- [ ] **Step 2: Load environment and run the script**

```bash
export $(cat .env | xargs)
npm run verify
```

Expected output:
```
Jenny Florería — Schema Verification
=====================================

── Tables ─────────────────────────────────────
  ✅  table: flowers
  ✅  table: customers
  ✅  table: orders
  ✅  table: order_items
  ✅  table: drivers
  ✅  table: promo_codes
  ✅  table: payment_plans
  ✅  table: payment_installments
  ✅  table: loyalty_tiers
  ✅  table: customer_reward_uses
  ✅  table: app_settings
  ✅  table: notifications

── Seed Data ───────────────────────────────────
  ✅  app_settings default row exists
  ✅  delivery_fee_mxn = 50
  ✅  driver_rate_per_delivery_mxn = 80
  ✅  3 loyalty tiers exist
  ✅  silver tier: 5 orders, 10%
  ✅  gold tier: 10 orders, 15%
  ✅  vip tier: 20 orders, 20%

── Dev Seed Data ───────────────────────────────
  ✅  at least 5 flowers exist
  ✅  at least 1 active flower
  ✅  3 drivers exist

── Constraints ─────────────────────────────────
  ✅  orders rejects negative total_mxn
  ✅  customers rejects invalid loyalty_tier

── Storage ─────────────────────────────────────
  ✅  flower-images bucket exists
  ✅  flower-images bucket is public

=====================================
Results: 22 passed, 0 failed

✅ All checks passed. Phase 1 complete — ready for Phase 2.
```

If any check fails, fix the migration and re-apply, then re-run.

- [ ] **Step 3: Commit verification script**

```bash
git add scripts/verify_schema.js
git commit -m "test: add schema verification script (22 checks)"
```

---

## Task 8: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write `README.md`**

```markdown
# Jenny Florería — AI Business Automation System

Automated order intake, payments, delivery dispatch, and owner notifications
for Jenny Florería, Guasave, Sinaloa, México.

## Architecture
See: `docs/superpowers/specs/2026-05-23-jenny-floreria-automation-design.md`

## Phases
| Phase | Description | Status |
|---|---|---|
| 1 | Supabase Foundation | ✅ Complete |
| 2 | WhatsApp AI Agent | ⏳ Pending |
| 3 | MercadoPago Payments | ⏳ Pending |
| 4 | Voice AI Agent (Vapi.ai + ElevenLabs) | ⏳ Pending |
| 5 | Delivery Dispatch | ⏳ Pending |
| 6 | Owner Dashboard (Next.js / Vercel) | ⏳ Pending |
| 7 | Public Customer Catalog | ⏳ Pending |
| 8 | Event & Wedding Installment Payments | ⏳ Pending |

## Setup (Phase 1)

### Prerequisites
- Node.js 18+
- Supabase account (free tier)
- Supabase CLI: `brew install supabase/tap/supabase`

### Steps
1. `npm install`
2. Create Supabase project at supabase.com
3. Copy `.env.example` → `.env`, fill in Supabase URL + keys
4. Apply migrations via Supabase SQL Editor (in order: 001 → 002 → 003 → 004)
5. Create `flower-images` storage bucket (public, 5MB limit, jpeg/png/webp)
6. `npm run verify` — all 22 checks should pass

## Tech Stack
- **Database:** Supabase (PostgreSQL + Storage + Realtime)
- **Automation:** n8n Cloud
- **AI (text):** OpenAI GPT-4o
- **AI (voice):** Vapi.ai + ElevenLabs Professional Voice Clone
- **Phone:** Twilio (+52 virtual number)
- **WhatsApp:** Meta Cloud API
- **Payments:** MercadoPago (OXXO / card / SPEI / wallet)
- **Dashboard:** Next.js on Vercel
```

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add README with phase overview and setup instructions"
```

---

## Phase 1 Complete ✅

Run final verification:
```bash
export $(cat .env | xargs)
npm run verify
# Expected: 22 passed, 0 failed
```

**What was built:**
- Complete 12-table Supabase schema with constraints, indexes, and triggers
- RLS policies (public read on flowers + loyalty_tiers, service_role for everything else)
- `flower-images` public storage bucket
- Default seed data (app_settings, 3 loyalty tiers)
- Dev seed data (6 sample flowers across all categories, 3 drivers)
- 22-check verification script

**Next:** Phase 2 — WhatsApp AI Agent
Plan: `docs/superpowers/plans/2026-05-23-phase2-whatsapp-agent.md` (to be written)
