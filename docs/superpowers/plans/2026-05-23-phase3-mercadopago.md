# Phase 3 — MercadoPago Payments Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Wire MercadoPago into the WhatsApp order flow so confirmed orders automatically send payment links, paid orders update to `status='paid'`, and abandoned orders get a 2-hour reminder then 24-hour cancellation with stock restore.

**Architecture:** n8n Cloud handles all payment orchestration: workflow 01 (existing) gains 4 inline nodes that call MercadoPago Preferences API and send the payment link after order creation; a new webhook workflow 04 receives MercadoPago payment events and marks orders paid; a new scheduled workflow 05 runs every 30 minutes to send reminders and cancel stale orders. Stock restore is atomic via a Supabase PostgreSQL function (migration 006). No new Supabase columns are needed — `mp_payment_id`, `mp_payment_status`, `status`, and `order_number` already exist on the `orders` table.

**Tech Stack:** n8n Cloud (HTTP Request nodes, Code nodes, Schedule Trigger, Webhook Trigger) · MercadoPago Preferences API + Payments API · Supabase REST + RPC · WhatsApp Cloud API

---

## Context for implementers

**Project root:** `/Users/montanaink/Projects/jenny-floreria/`

**What already exists (do NOT recreate):**
- `supabase/migrations/001_initial_schema.sql` — defines `orders` table with `mp_payment_id TEXT`, `mp_payment_status TEXT`, `order_number SERIAL`, `status` (values: `pending/confirmed/partially_paid/paid/driver_assigned/in_delivery/delivered/cancelled`)
- `n8n/workflows/01-whatsapp-inbound.json` — 31-node workflow; key connection: `Create Order Items → Send WhatsApp Reply` (modified in Task 6)
- `.env.example` — already has `MERCADOPAGO_ACCESS_TOKEN=` commented out (activate in Task 2)

**Workflow 01 node topology (relevant subset):**
```
Order Ready? [node-22] --[true]--> Create Order [node-23] --> Create Order Items [node-24] --> Send WhatsApp Reply [node-28]
             --[false]--> Escalate? [node-25]
```
Task 6 inserts 4 new nodes between `Create Order Items` and `Send WhatsApp Reply`.

**MercadoPago API (Mexico):**
- Create preference: `POST https://api.mercadopago.com/checkout/preferences`
  - Auth: `Authorization: Bearer $MERCADOPAGO_ACCESS_TOKEN`
  - Response includes `id` (preference ID) and `init_point` (payment URL)
- Query payment: `GET https://api.mercadopago.com/v1/payments/{payment_id}`
- Webhook body from MP: `{ "type": "payment", "data": { "id": "999999999" } }`
- Payment approved field: `payment.status === "approved"`, `payment.external_reference` = our order UUID

**Supabase RPC call format:** `POST /rest/v1/rpc/function_name` with JSON body `{"param": "value"}` and headers `apikey + Authorization + Content-Type: application/json`

**WhatsApp `to` field:** phone stored as `+521XXXXXXXXXX` — strip the `+` prefix → `521XXXXXXXXXX`

**n8n Code node context:** `$('Node Name').first().json` returns the raw JSON of the first item output by that node. Supabase REST with `Prefer: return=representation` returns an array, so `$('Create Order').first().json[0]` is the order object.

---

## File Map

| Action | File | Purpose |
|--------|------|---------|
| Create | `supabase/migrations/006_cancel_expired_order_fn.sql` | RPC function for atomic order cancel + stock restore |
| Modify | `.env.example` | Activate MERCADOPAGO_ACCESS_TOKEN, add MERCADOPAGO_NOTIFICATION_URL |
| Create | `n8n/code/build-mp-preference.js` | Builds MercadoPago Preferences API request body |
| Create | `n8n/workflows/04-mercadopago-webhook.json` | Webhook receiver: marks orders paid on MP approval |
| Create | `n8n/workflows/05-abandoned-payment.json` | Scheduled: 2h reminder, 24h cancel + stock restore |
| Modify | `n8n/workflows/01-whatsapp-inbound.json` | Inject 4 payment nodes after Create Order Items |
| Modify | `n8n/README.md` | Phase 3 setup instructions |

---

### Task 1: Migration 006 — cancel_expired_order RPC function

**Files:**
- Create: `supabase/migrations/006_cancel_expired_order_fn.sql`

This function atomically cancels a confirmed order and restores each flower's stock. Called from workflow 05 via Supabase RPC. Without atomicity, a network failure mid-cancel could leave stock unreleased.

- [ ] **Step 1: Create the migration file**

```sql
-- supabase/migrations/006_cancel_expired_order_fn.sql
-- Atomically cancels a confirmed order and restores flower stock.
-- Called by n8n abandoned-payment workflow via POST /rest/v1/rpc/cancel_expired_order

CREATE OR REPLACE FUNCTION cancel_expired_order(p_order_id uuid)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  UPDATE orders
  SET status = 'cancelled', updated_at = now()
  WHERE id = p_order_id AND status = 'confirmed';

  UPDATE flowers f
  SET stock = f.stock + oi.quantity, updated_at = now()
  FROM order_items oi
  WHERE oi.order_id = p_order_id AND oi.flower_id = f.id;
END;
$$;
```

- [ ] **Step 2: Apply to Supabase**

Open Supabase dashboard → SQL Editor → paste and run the SQL above.

Verify: in Table Editor, check that the function appears under Database → Functions → `cancel_expired_order`.

- [ ] **Step 3: Test the function manually**

In Supabase SQL Editor, run (substitute a real `confirmed` order UUID if one exists in dev seed, otherwise just verify syntax):

```sql
-- Verify function exists
SELECT routine_name FROM information_schema.routines
WHERE routine_name = 'cancel_expired_order' AND routine_schema = 'public';
-- Expected: 1 row returned
```

- [ ] **Step 4: Commit**

```bash
cd /Users/montanaink/Projects/jenny-floreria
git add supabase/migrations/006_cancel_expired_order_fn.sql
git commit -m "feat(phase3): add cancel_expired_order RPC function for atomic stock restore"
```

---

### Task 2: .env.example — MercadoPago variables

**Files:**
- Modify: `.env.example`

- [ ] **Step 1: Update .env.example**

Replace the current MercadoPago placeholder block:

```
# Set after Phase 3
# MERCADOPAGO_ACCESS_TOKEN=
```

With this (expanded documentation):

```
# ─── Phase 3: MercadoPago Payments ──────────────────────────
# Get from: mercadopago.com.mx → Desarrolladores → Credenciales
# Use TEST credentials for sandbox, PROD credentials for production
MERCADOPAGO_ACCESS_TOKEN=APP_USR-...

# n8n Cloud webhook URL for MercadoPago payment notifications
# After importing workflow 04, copy its webhook URL from n8n and paste here
# Format: https://your-instance.app.n8n.cloud/webhook/mercadopago-payment
MERCADOPAGO_NOTIFICATION_URL=https://your-instance.app.n8n.cloud/webhook/mercadopago-payment
```

The `.env.example` already has the "Phase 4" block below — insert the new block before it.

- [ ] **Step 2: Verify file looks correct**

Run:

```bash
grep -A 5 "Phase 3" /Users/montanaink/Projects/jenny-floreria/.env.example
```

Expected output contains both `MERCADOPAGO_ACCESS_TOKEN` and `MERCADOPAGO_NOTIFICATION_URL` uncommented.

- [ ] **Step 3: Commit**

```bash
cd /Users/montanaink/Projects/jenny-floreria
git add .env.example
git commit -m "feat(phase3): add MercadoPago env vars to .env.example"
```

---

### Task 3: build-mp-preference.js — MercadoPago request body builder

**Files:**
- Create: `n8n/code/build-mp-preference.js`

This Code node runs inside workflow 01, immediately after `Create Order Items`. It reads from previously executed nodes in the same workflow run and returns the full body for the MercadoPago Preferences API call.

- [ ] **Step 1: Create the file**

```javascript
// n8n/code/build-mp-preference.js
// Runs after Create Order Items in workflow 01.
// Reads: Create Order (Supabase array response), Upsert Customer (Supabase array response).
// Returns: MercadoPago Preferences API request body.

const order = $('Create Order').first().json[0]
const customer = $('Upsert Customer').first().json[0]

// Strip +52 country code, keep 10-digit number with mobile prefix (e.g. 1XXXXXXXXXX)
const phoneRaw = (customer.whatsapp_phone || '').replace(/^\+52/, '')

const expiration = new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString()

return [{
  json: {
    items: [{
      id: `order-${order.id}`,
      title: `Jenny Florería – Pedido #${order.order_number}`,
      quantity: 1,
      unit_price: Number(order.total_mxn),
      currency_id: 'MXN'
    }],
    payer: {
      name: customer.name || 'Cliente',
      phone: {
        area_code: '52',
        number: phoneRaw
      }
    },
    external_reference: order.id,
    notification_url: $env.MERCADOPAGO_NOTIFICATION_URL,
    statement_descriptor: 'Jenny Floreria',
    expires: true,
    expiration_date_to: expiration
  }
}]
```

- [ ] **Step 2: Verify file was created correctly**

```bash
node --input-type=module << 'EOF'
import { readFileSync } from 'fs'
const code = readFileSync('/Users/montanaink/Projects/jenny-floreria/n8n/code/build-mp-preference.js', 'utf8')
// Check required fields appear
const checks = ['external_reference', 'notification_url', 'expiration_date_to', 'unit_price', 'order_number']
checks.forEach(k => {
  if (!code.includes(k)) console.error(`MISSING: ${k}`)
  else console.log(`OK: ${k}`)
})
EOF
```

Expected: 5 lines starting with `OK:`.

- [ ] **Step 3: Commit**

```bash
cd /Users/montanaink/Projects/jenny-floreria
git add n8n/code/build-mp-preference.js
git commit -m "feat(phase3): add MercadoPago preference body builder code node"
```

---

### Task 4: Workflow 04 — MercadoPago payment webhook

**Files:**
- Create: `n8n/workflows/04-mercadopago-webhook.json`

This workflow receives POST notifications from MercadoPago when a payment is created or updated. It queries MP for full payment details, and if `status === 'approved'`, it marks the order as paid in Supabase and sends a WhatsApp confirmation to the customer.

Webhook URL format after import into n8n Cloud: `https://[instance].app.n8n.cloud/webhook/mercadopago-payment`

- [ ] **Step 1: Create the workflow JSON file**

```json
{
  "name": "04 - MercadoPago Payment Webhook",
  "nodes": [
    {
      "id": "mp-01",
      "name": "MP Payment Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [240, 400],
      "webhookId": "mercadopago-payment",
      "parameters": {
        "path": "mercadopago-payment",
        "httpMethod": "POST",
        "responseMode": "onReceived",
        "responseCode": "200",
        "options": {}
      }
    },
    {
      "id": "mp-02",
      "name": "Is Payment Event?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [480, 400],
      "parameters": {
        "conditions": {
          "options": { "caseSensitive": false, "leftValue": "", "typeValidation": "strict" },
          "conditions": [
            {
              "id": "cond-01",
              "leftValue": "={{ $json.type }}",
              "rightValue": "payment",
              "operator": { "type": "string", "operation": "equals" }
            }
          ],
          "combinator": "and"
        }
      }
    },
    {
      "id": "mp-03",
      "name": "Get Payment Details",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [720, 320],
      "parameters": {
        "method": "GET",
        "url": "=https://api.mercadopago.com/v1/payments/{{ $('MP Payment Webhook').first().json.data.id }}",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "Authorization", "value": "=Bearer {{ $env.MERCADOPAGO_ACCESS_TOKEN }}" }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "mp-04",
      "name": "Is Approved?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [960, 320],
      "parameters": {
        "conditions": {
          "options": { "caseSensitive": false, "leftValue": "", "typeValidation": "strict" },
          "conditions": [
            {
              "id": "cond-02",
              "leftValue": "={{ $json.status }}",
              "rightValue": "approved",
              "operator": { "type": "string", "operation": "equals" }
            }
          ],
          "combinator": "and"
        }
      }
    },
    {
      "id": "mp-05",
      "name": "Mark Order Paid",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1200, 240],
      "parameters": {
        "method": "PATCH",
        "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?id=eq.{{ $('Get Payment Details').first().json.external_reference }}",
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ status: 'paid', mp_payment_id: String($('Get Payment Details').first().json.id), mp_payment_status: 'approved', amount_paid_mxn: $('Get Payment Details').first().json.transaction_amount, updated_at: new Date().toISOString() }) }}",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
            { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
            { "name": "Content-Type", "value": "application/json" }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "mp-06",
      "name": "Load Order for Notification",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1440, 240],
      "parameters": {
        "method": "GET",
        "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?id=eq.{{ $('Get Payment Details').first().json.external_reference }}&select=order_number,customers(whatsapp_phone,name)",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
            { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "mp-07",
      "name": "Notify Customer Payment OK",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1680, 240],
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $('Load Order for Notification').first().json[0].customers.whatsapp_phone.replace('+', ''), type: 'text', text: { body: '✅ *¡Pago confirmado!* 🌸\\n\\nTu pedido #' + $('Load Order for Notification').first().json[0].order_number + ' en Jenny Florería ha sido pagado exitosamente.\\n\\nEn breve te avisamos cuando tu pedido esté en camino. ¡Gracias por tu compra! 🌺' } }) }}",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
          ]
        },
        "options": {}
      }
    }
  ],
  "connections": {
    "MP Payment Webhook": {
      "main": [[{ "node": "Is Payment Event?", "type": "main", "index": 0 }]]
    },
    "Is Payment Event?": {
      "main": [
        [{ "node": "Get Payment Details", "type": "main", "index": 0 }],
        []
      ]
    },
    "Get Payment Details": {
      "main": [[{ "node": "Is Approved?", "type": "main", "index": 0 }]]
    },
    "Is Approved?": {
      "main": [
        [{ "node": "Mark Order Paid", "type": "main", "index": 0 }],
        []
      ]
    },
    "Mark Order Paid": {
      "main": [[{ "node": "Load Order for Notification", "type": "main", "index": 0 }]]
    },
    "Load Order for Notification": {
      "main": [[{ "node": "Notify Customer Payment OK", "type": "main", "index": 0 }]]
    }
  },
  "active": false,
  "settings": { "executionOrder": "v1" },
  "staticData": null,
  "tags": [],
  "triggerCount": 0,
  "updatedAt": "2026-05-23T00:00:00.000Z",
  "versionId": "phase3-mp-webhook-v1"
}
```

- [ ] **Step 2: Validate JSON syntax**

```bash
python3 -c "import json; json.load(open('/Users/montanaink/Projects/jenny-floreria/n8n/workflows/04-mercadopago-webhook.json')); print('JSON valid')"
```

Expected: `JSON valid`

- [ ] **Step 3: Verify node count and connections**

```bash
python3 -c "
import json
data = json.load(open('/Users/montanaink/Projects/jenny-floreria/n8n/workflows/04-mercadopago-webhook.json'))
print('Nodes:', len(data['nodes']))
print('Connections:', list(data['connections'].keys()))
"
```

Expected:
```
Nodes: 7
Connections: ['MP Payment Webhook', 'Is Payment Event?', 'Get Payment Details', 'Is Approved?', 'Mark Order Paid', 'Load Order for Notification']
```

- [ ] **Step 4: Commit**

```bash
cd /Users/montanaink/Projects/jenny-floreria
git add n8n/workflows/04-mercadopago-webhook.json
git commit -m "feat(phase3): add MercadoPago payment webhook workflow (04)"
```

---

### Task 5: Workflow 05 — Abandoned payment checker

**Files:**
- Create: `n8n/workflows/05-abandoned-payment.json`

Runs every 30 minutes. Finds confirmed orders with `mp_payment_status` of `pending` or `reminder_sent`. For orders >2h old (and still `pending`): sends WhatsApp reminder, sets `mp_payment_status='reminder_sent'`. For orders >24h old: calls `cancel_expired_order` RPC to atomically cancel + restore stock.

Flow: Schedule → Query Supabase → Code (split array into items) → Code (compute action per order) → If (cancel?) → If (remind?) → relevant HTTP nodes.

- [ ] **Step 1: Create the workflow JSON file**

```json
{
  "name": "05 - Abandoned Payment Checker",
  "nodes": [
    {
      "id": "ap-01",
      "name": "Every 30 Minutes",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [240, 400],
      "parameters": {
        "rule": {
          "interval": [{ "field": "minutes", "minutesInterval": 30 }]
        }
      }
    },
    {
      "id": "ap-02",
      "name": "Query Abandoned Orders",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [480, 400],
      "parameters": {
        "method": "GET",
        "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?status=eq.confirmed&mp_payment_status=in.(pending,reminder_sent)&select=id,order_number,mp_payment_status,created_at,total_mxn,customers(whatsapp_phone,name)&limit=100",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
            { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "ap-03",
      "name": "Split Into Items",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [720, 400],
      "parameters": {
        "jsCode": "const orders = $input.first().json\nif (!Array.isArray(orders) || orders.length === 0) return []\nreturn orders.map(order => ({ json: order }))"
      }
    },
    {
      "id": "ap-04",
      "name": "Compute Action",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [960, 400],
      "parameters": {
        "jsCode": "const order = $input.first().json\nconst createdAt = new Date(order.created_at).getTime()\nconst now = Date.now()\nconst hoursElapsed = (now - createdAt) / (1000 * 60 * 60)\n\nlet action = 'skip'\nif (hoursElapsed >= 24) {\n  action = 'cancel'\n} else if (hoursElapsed >= 2 && order.mp_payment_status === 'pending') {\n  action = 'remind'\n}\n\nreturn [{ json: { ...order, action, hoursElapsed } }]"
      }
    },
    {
      "id": "ap-05",
      "name": "Should Cancel?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1200, 400],
      "parameters": {
        "conditions": {
          "options": { "caseSensitive": false, "leftValue": "", "typeValidation": "strict" },
          "conditions": [
            {
              "id": "cond-01",
              "leftValue": "={{ $json.action }}",
              "rightValue": "cancel",
              "operator": { "type": "string", "operation": "equals" }
            }
          ],
          "combinator": "and"
        }
      }
    },
    {
      "id": "ap-06",
      "name": "Cancel Order RPC",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1440, 280],
      "parameters": {
        "method": "POST",
        "url": "={{ $env.SUPABASE_URL }}/rest/v1/rpc/cancel_expired_order",
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ p_order_id: $json.id }) }}",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
            { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
            { "name": "Content-Type", "value": "application/json" }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "ap-07",
      "name": "Should Remind?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1440, 520],
      "parameters": {
        "conditions": {
          "options": { "caseSensitive": false, "leftValue": "", "typeValidation": "strict" },
          "conditions": [
            {
              "id": "cond-02",
              "leftValue": "={{ $json.action }}",
              "rightValue": "remind",
              "operator": { "type": "string", "operation": "equals" }
            }
          ],
          "combinator": "and"
        }
      }
    },
    {
      "id": "ap-08",
      "name": "Send Reminder WhatsApp",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1680, 440],
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $json.customers.whatsapp_phone.replace('+', ''), type: 'text', text: { body: '💳 ¡Hola! Tu pedido #' + $json.order_number + ' en Jenny Florería sigue pendiente de pago.\\n\\nTu link de pago sigue disponible. Si ya no deseas el pedido, avísanos y lo cancelamos sin problema.\\n\\n¡Estamos aquí para ayudarte! 🌸' } }) }}",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
          ]
        },
        "options": {}
      }
    },
    {
      "id": "ap-09",
      "name": "Mark Reminder Sent",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1920, 440],
      "parameters": {
        "method": "PATCH",
        "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?id=eq.{{ $('Compute Action').first().json.id }}",
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ mp_payment_status: 'reminder_sent', updated_at: new Date().toISOString() }) }}",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
            { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
            { "name": "Content-Type", "value": "application/json" }
          ]
        },
        "options": {}
      }
    }
  ],
  "connections": {
    "Every 30 Minutes": {
      "main": [[{ "node": "Query Abandoned Orders", "type": "main", "index": 0 }]]
    },
    "Query Abandoned Orders": {
      "main": [[{ "node": "Split Into Items", "type": "main", "index": 0 }]]
    },
    "Split Into Items": {
      "main": [[{ "node": "Compute Action", "type": "main", "index": 0 }]]
    },
    "Compute Action": {
      "main": [[{ "node": "Should Cancel?", "type": "main", "index": 0 }]]
    },
    "Should Cancel?": {
      "main": [
        [{ "node": "Cancel Order RPC", "type": "main", "index": 0 }],
        [{ "node": "Should Remind?", "type": "main", "index": 0 }]
      ]
    },
    "Should Remind?": {
      "main": [
        [{ "node": "Send Reminder WhatsApp", "type": "main", "index": 0 }],
        []
      ]
    },
    "Send Reminder WhatsApp": {
      "main": [[{ "node": "Mark Reminder Sent", "type": "main", "index": 0 }]]
    }
  },
  "active": false,
  "settings": { "executionOrder": "v1" },
  "staticData": null,
  "tags": [],
  "triggerCount": 0,
  "updatedAt": "2026-05-23T00:00:00.000Z",
  "versionId": "phase3-abandoned-v1"
}
```

- [ ] **Step 2: Validate JSON syntax**

```bash
python3 -c "import json; json.load(open('/Users/montanaink/Projects/jenny-floreria/n8n/workflows/05-abandoned-payment.json')); print('JSON valid')"
```

Expected: `JSON valid`

- [ ] **Step 3: Verify node names and connections**

```bash
python3 -c "
import json
data = json.load(open('/Users/montanaink/Projects/jenny-floreria/n8n/workflows/05-abandoned-payment.json'))
print('Nodes:', [n['name'] for n in data['nodes']])
print('Connection count:', len(data['connections']))
"
```

Expected:
```
Nodes: ['Every 30 Minutes', 'Query Abandoned Orders', 'Split Into Items', 'Compute Action', 'Should Cancel?', 'Cancel Order RPC', 'Should Remind?', 'Send Reminder WhatsApp', 'Mark Reminder Sent']
Connection count: 7
```

- [ ] **Step 4: Commit**

```bash
cd /Users/montanaink/Projects/jenny-floreria
git add n8n/workflows/05-abandoned-payment.json
git commit -m "feat(phase3): add abandoned payment checker workflow (05)"
```

---

### Task 6: Modify workflow 01 — inject payment link nodes

**Files:**
- Modify: `n8n/workflows/01-whatsapp-inbound.json`

**What changes:** Insert 4 new nodes between `Create Order Items` (node-24) and `Send WhatsApp Reply` (node-28) in the order-ready branch.

New chain: `Create Order Items → Build MP Preference Body → Call MercadoPago API → Update Order MP Status → Send Payment Link → Send WhatsApp Reply`

**Position guide:** Current positions — Create Order Items: `[3320, 520]`, Send WhatsApp Reply: `[3560, 680]`. New nodes get x-positions 3560, 3800, 4040, 4280 at y=520. Send WhatsApp Reply stays at its current position.

- [ ] **Step 1: Read the current workflow file**

```bash
python3 -c "
import json
data = json.load(open('/Users/montanaink/Projects/jenny-floreria/n8n/workflows/01-whatsapp-inbound.json'))
# Verify the connection we're about to change
conn = data['connections']['Create Order Items']['main'][0]
print('Create Order Items currently connects to:', conn)
print('Node count before:', len(data['nodes']))
"
```

Expected:
```
Create Order Items currently connects to: [{'node': 'Send WhatsApp Reply', 'type': 'main', 'index': 0}]
Node count before: 31
```

If output differs, stop and investigate before proceeding.

- [ ] **Step 2: Write the Python modification script**

Create a temporary script (delete after use):

```bash
cat > /tmp/patch_workflow_01.py << 'SCRIPT'
import json, copy

path = '/Users/montanaink/Projects/jenny-floreria/n8n/workflows/01-whatsapp-inbound.json'
data = json.load(open(path))

# 4 new nodes to insert
new_nodes = [
  {
    "id": "node-33",
    "name": "Build MP Preference Body",
    "type": "n8n-nodes-base.code",
    "typeVersion": 2,
    "position": [3560, 520],
    "parameters": {
      "jsCode": "// PASTE CONTENT OF n8n/code/build-mp-preference.js HERE\nconst order = $('Create Order').first().json[0]\nconst customer = $('Upsert Customer').first().json[0]\nconst phoneRaw = (customer.whatsapp_phone || '').replace(/^\\+52/, '')\nconst expiration = new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString()\nreturn [{\n  json: {\n    items: [{ id: `order-${order.id}`, title: `Jenny Florería \\u2013 Pedido #${order.order_number}`, quantity: 1, unit_price: Number(order.total_mxn), currency_id: 'MXN' }],\n    payer: { name: customer.name || 'Cliente', phone: { area_code: '52', number: phoneRaw } },\n    external_reference: order.id,\n    notification_url: $env.MERCADOPAGO_NOTIFICATION_URL,\n    statement_descriptor: 'Jenny Floreria',\n    expires: true,\n    expiration_date_to: expiration\n  }\n}]"
    }
  },
  {
    "id": "node-34",
    "name": "Call MercadoPago API",
    "type": "n8n-nodes-base.httpRequest",
    "typeVersion": 4,
    "position": [3800, 520],
    "parameters": {
      "method": "POST",
      "url": "https://api.mercadopago.com/checkout/preferences",
      "sendBody": True,
      "contentType": "json",
      "body": "={{ JSON.stringify($input.first().json) }}",
      "sendHeaders": True,
      "headerParameters": {
        "parameters": [
          { "name": "Authorization", "value": "=Bearer {{ $env.MERCADOPAGO_ACCESS_TOKEN }}" },
          { "name": "Content-Type", "value": "application/json" }
        ]
      },
      "options": {}
    }
  },
  {
    "id": "node-35",
    "name": "Update Order MP Status",
    "type": "n8n-nodes-base.httpRequest",
    "typeVersion": 4,
    "position": [4040, 520],
    "parameters": {
      "method": "PATCH",
      "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?id=eq.{{ $('Create Order').first().json[0].id }}",
      "sendBody": True,
      "contentType": "json",
      "body": "={{ JSON.stringify({ mp_payment_id: String($('Call MercadoPago API').first().json.id), mp_payment_status: 'pending', updated_at: new Date().toISOString() }) }}",
      "sendHeaders": True,
      "headerParameters": {
        "parameters": [
          { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
          { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
          { "name": "Content-Type", "value": "application/json" }
        ]
      },
      "options": {}
    }
  },
  {
    "id": "node-36",
    "name": "Send Payment Link WhatsApp",
    "type": "n8n-nodes-base.httpRequest",
    "typeVersion": 4,
    "position": [4280, 520],
    "parameters": {
      "method": "POST",
      "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
      "sendBody": True,
      "contentType": "json",
      "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $('Extract Message').first().json.phone, type: 'text', text: { body: '\\ud83d\\udcb3 *Paga tu pedido #' + $('Create Order').first().json[0].order_number + ' aqu\\u00ed:*\\n' + $('Call MercadoPago API').first().json.init_point + '\\n\\nPuedes pagar con:\\n\\ud83d\\udcb3 Tarjeta \\u00b7 \\ud83c\\udfe2 OXXO \\u00b7 \\ud83c\\udfe6 Transferencia\\n\\nEl link es v\\u00e1lido por 24 horas.' } }) }}",
      "sendHeaders": True,
      "headerParameters": {
        "parameters": [
          { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
        ]
      },
      "options": {}
    }
  }
]

data['nodes'].extend(new_nodes)

# Fix connections: Create Order Items → Build MP Preference Body (was → Send WhatsApp Reply)
data['connections']['Create Order Items']['main'][0] = [
  { "node": "Build MP Preference Body", "type": "main", "index": 0 }
]

# Add new connections
data['connections']['Build MP Preference Body'] = {
  "main": [[{ "node": "Call MercadoPago API", "type": "main", "index": 0 }]]
}
data['connections']['Call MercadoPago API'] = {
  "main": [[{ "node": "Update Order MP Status", "type": "main", "index": 0 }]]
}
data['connections']['Update Order MP Status'] = {
  "main": [[{ "node": "Send Payment Link WhatsApp", "type": "main", "index": 0 }]]
}
data['connections']['Send Payment Link WhatsApp'] = {
  "main": [[{ "node": "Send WhatsApp Reply", "type": "main", "index": 0 }]]
}

with open(path, 'w') as f:
    json.dump(data, f, indent=2, ensure_ascii=False)

print('Done. Nodes now:', len(data['nodes']))
print('Create Order Items now connects to:', data['connections']['Create Order Items']['main'][0])
SCRIPT
python3 /tmp/patch_workflow_01.py
```

Expected output:
```
Done. Nodes now: 35
Create Order Items now connects to: [{'node': 'Build MP Preference Body', 'type': 'main', 'index': 0}]
```

- [ ] **Step 3: Verify the modification**

```bash
python3 -c "
import json
data = json.load(open('/Users/montanaink/Projects/jenny-floreria/n8n/workflows/01-whatsapp-inbound.json'))
print('Total nodes:', len(data['nodes']))
new_nodes = ['Build MP Preference Body', 'Call MercadoPago API', 'Update Order MP Status', 'Send Payment Link WhatsApp']
names = [n['name'] for n in data['nodes']]
for nn in new_nodes:
    print(f'  {nn}:', 'FOUND' if nn in names else 'MISSING')
print('Create Order Items → ', data['connections']['Create Order Items']['main'][0][0]['node'])
print('Send Payment Link → ', data['connections']['Send Payment Link WhatsApp']['main'][0][0]['node'])
"
```

Expected:
```
Total nodes: 35
  Build MP Preference Body: FOUND
  Call MercadoPago API: FOUND
  Update Order MP Status: FOUND
  Send Payment Link WhatsApp: FOUND
Create Order Items →  Build MP Preference Body
Send Payment Link →  Send WhatsApp Reply
```

- [ ] **Step 4: Clean up temp script**

```bash
rm /tmp/patch_workflow_01.py
```

- [ ] **Step 5: Commit**

```bash
cd /Users/montanaink/Projects/jenny-floreria
git add n8n/workflows/01-whatsapp-inbound.json
git commit -m "feat(phase3): inject MercadoPago payment link nodes into workflow 01"
```

---

### Task 7: Update README — Phase 3 setup instructions

**Files:**
- Modify: `n8n/README.md`

Add a Phase 3 section after the existing Phase 2 setup content. This section covers MercadoPago account setup, credential configuration, webhook URL, and how to test.

- [ ] **Step 1: Read the current README to find insertion point**

```bash
grep -n "Phase\|Step\|---" /Users/montanaink/Projects/jenny-floreria/n8n/README.md | tail -20
```

This shows you where Phase 2 content ends so you can append Phase 3 after it.

- [ ] **Step 2: Append Phase 3 section to README**

Add the following content at the end of `n8n/README.md`:

```markdown

---

## Phase 3 Setup — MercadoPago Payments

### Prerequisites
- MercadoPago account (mercadopago.com.mx) registered as the business (Jenny Florería)
- Test credentials for sandbox testing, production credentials for go-live

### Step 1: Get MercadoPago credentials

1. Log in to mercadopago.com.mx
2. Go to **Desarrolladores → Mis aplicaciones → Create application**
3. Application name: `Jenny Florería`
4. Enable: **Checkout Pro** (the Preferences API we use)
5. Copy your **TEST Access Token** (starts with `TEST-`) for sandbox testing
6. For production, copy the **PROD Access Token** (starts with `APP_USR-`)

### Step 2: Configure n8n environment variables

In n8n Cloud → Settings → Variables, add:

| Variable | Value |
|---|---|
| `MERCADOPAGO_ACCESS_TOKEN` | Your MP access token (TEST- for sandbox) |
| `MERCADOPAGO_NOTIFICATION_URL` | *(leave blank for now — fill in Step 4)* |

### Step 3: Import workflows 04 and 05

In n8n Cloud → Workflows → Import:
1. Import `n8n/workflows/04-mercadopago-webhook.json`
2. Import `n8n/workflows/05-abandoned-payment.json`

Do **not** activate yet.

### Step 4: Get webhook URL for workflow 04

1. Open workflow 04 in n8n
2. Click the **MP Payment Webhook** node
3. Copy the **Test URL** (for sandbox testing) or **Production URL** (for live)
   - Format: `https://[your-instance].app.n8n.cloud/webhook/mercadopago-payment`
4. Go back to n8n Variables → update `MERCADOPAGO_NOTIFICATION_URL` with this URL
5. Also update `MERCADOPAGO_NOTIFICATION_URL` in your local `.env.example`

### Step 5: Configure MercadoPago webhook (optional but recommended for production)

In MercadoPago → Desarrolladores → Webhooks:
1. Add webhook URL: your n8n webhook URL from Step 4
2. Events: check **Pagos** (Payments)
3. Save

For sandbox testing, MercadoPago sends webhook events automatically when test payments complete. The notification_url in the preference body also works as a direct webhook.

### Step 6: Update workflow 01 (re-import)

Workflow 01 was modified in Task 6 to include payment link nodes. Re-import it:

In n8n Cloud → Workflows → find `01 - WhatsApp Inbound` → Import (overwrite) with `n8n/workflows/01-whatsapp-inbound.json`.

After import, paste code into the 3 code nodes that need it (same as Phase 2 Step 3):
- **Build MP Preference Body**: paste `n8n/code/build-mp-preference.js`
- **Build GPT Prompt**: paste `n8n/code/build-gpt-prompt.js` (unchanged from Phase 2)
- **Parse GPT Response**: paste `n8n/code/parse-gpt-response.js` (unchanged)
- **Prepare Order Payload**: paste `n8n/code/prepare-order.js` (unchanged)

### Step 7: Activate workflows

Activate in this order:
1. Workflow 04 (MP webhook) — must be active before testing payments
2. Workflow 05 (abandoned checker) — runs on schedule, activate when ready
3. Workflow 01 (WhatsApp inbound) — already active from Phase 2, re-activate after re-import

### Step 8: Test payment flow (sandbox)

MercadoPago sandbox test cards (Mexico):
| Card | Number | CVV | Expiry | Result |
|---|---|---|---|---|
| Visa approved | 4509 9535 6623 3704 | 123 | 11/25 | Approved |
| Mastercard rejected | 5031 7557 3453 0604 | 123 | 11/25 | Rejected |
| OXXO | Use any email ending in `@testuser.com` | — | — | Pending |

**Test flow:**
1. Send a WhatsApp message to the test number to start an order
2. Complete the order conversation with Jenny
3. When order is confirmed, you should receive a second WhatsApp message with a payment link
4. Open the payment link → complete test payment with sandbox card
5. MercadoPago sends webhook → workflow 04 fires → order status updates to `paid`
6. Customer receives WhatsApp confirmation "¡Pago confirmado!"

Verify in Supabase: `orders` table should show `status='paid'`, `mp_payment_id`, `mp_payment_status='approved'`

**Test abandoned flow:**
- Temporarily change the abandoned threshold in workflow 05 `Compute Action` code to `hoursElapsed >= 0.01` (about 36 seconds) for testing
- Let an order sit after getting the payment link → within 30 minutes (or the next schedule run), it should get a reminder WhatsApp
- Revert the threshold after testing

### Troubleshooting

**Payment link not sent after order confirmation:**
- Check workflow 01 execution log → look for errors in `Call MercadoPago API` node
- Verify `MERCADOPAGO_ACCESS_TOKEN` is set correctly in n8n Variables
- Sandbox token starts with `TEST-`, production starts with `APP_USR-`
- Check that `MERCADOPAGO_NOTIFICATION_URL` is set (required by MP API)

**Webhook not firing (payment not updating order):**
- Verify workflow 04 is active
- Check that `MERCADOPAGO_NOTIFICATION_URL` in n8n Variables matches the webhook URL shown in workflow 04
- In MP dashboard → Desarrolladores → Webhooks → check delivery status

**`cancel_expired_order` RPC not working:**
- Confirm migration 006 was applied: Supabase → Database → Functions → `cancel_expired_order` should exist
- If missing: run the SQL from `supabase/migrations/006_cancel_expired_order_fn.sql` in Supabase SQL Editor
```

- [ ] **Step 3: Verify README updated**

```bash
grep -c "Phase 3" /Users/montanaink/Projects/jenny-floreria/n8n/README.md
```

Expected: at least `3` (appears in heading, step references, etc.)

- [ ] **Step 4: Commit**

```bash
cd /Users/montanaink/Projects/jenny-floreria
git add n8n/README.md
git commit -m "docs(phase3): add MercadoPago setup instructions to n8n README"
```

---

## Final verification

After all 7 tasks complete, run:

```bash
cd /Users/montanaink/Projects/jenny-floreria

# Check all new files exist
ls -la supabase/migrations/006_cancel_expired_order_fn.sql
ls -la n8n/code/build-mp-preference.js
ls -la n8n/workflows/04-mercadopago-webhook.json
ls -la n8n/workflows/05-abandoned-payment.json

# Verify workflow 01 has 35 nodes
python3 -c "import json; d=json.load(open('n8n/workflows/01-whatsapp-inbound.json')); print('Workflow 01 nodes:', len(d['nodes']))"

# Verify .env.example has MP vars
grep "MERCADOPAGO" .env.example

# Check git log
git log --oneline -7
```

Expected git log (7 commits, most recent first):
```
xxxxxxx docs(phase3): add MercadoPago setup instructions to n8n README
xxxxxxx feat(phase3): inject MercadoPago payment link nodes into workflow 01
xxxxxxx feat(phase3): add abandoned payment checker workflow (05)
xxxxxxx feat(phase3): add MercadoPago payment webhook workflow (04)
xxxxxxx feat(phase3): add MercadoPago preference body builder code node
xxxxxxx feat(phase3): add MercadoPago env vars to .env.example
xxxxxxx feat(phase3): add cancel_expired_order RPC function for atomic stock restore
```

---

## Self-review notes

**Spec coverage check:**
- ✅ Order confirmed → MercadoPago Preferences API → payment URL (Task 3, 6)
- ✅ Send payment link via WhatsApp with card/OXXO/transfer text (Task 6, node-36)
- ✅ MercadoPago webhook → order status = "paid" + mp_payment_id saved (Task 4)
- ✅ 2-hour abandoned reminder (Task 5, Compute Action: `hoursElapsed >= 2`)
- ✅ 24-hour cancellation + stock release (Task 1, 5: RPC + Cancel Order RPC node)
- ✅ Driver dispatch triggered after paid — Note: Phase 5 handles driver dispatch. Phase 3 only sets `status='paid'`. Workflow 04 can be extended in Phase 5 to trigger dispatch after marking paid.

**No placeholders present** — all code blocks contain complete, runnable code.

**Type consistency:**
- `order.id` used consistently (UUID from Supabase)
- `order.order_number` (integer SERIAL) used in WhatsApp messages
- `$('Create Order').first().json[0]` — consistent with Phase 2 pattern (Supabase array response)
- `mp_payment_status` values: `'pending'` → `'reminder_sent'` → `'approved'` — consistent across tasks 3, 4, 5, 6
- WhatsApp `to` field always strips `+` prefix (consistent with Phase 2 design decision)
