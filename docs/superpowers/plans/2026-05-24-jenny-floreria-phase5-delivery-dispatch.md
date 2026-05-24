# Jenny Florería Phase 5 — Delivery Dispatch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Automate WhatsApp delivery dispatch to up to 5 drivers — broadcast on payment, first ACEPTO wins, ENTREGADO marks delivered, all parties notified at every step.

**Architecture:** Modify workflow 01 (WhatsApp inbound) to detect driver phone numbers and branch into driver-reply logic (ACEPTO/RECHAZO/ENTREGADO/PROBLEMA). Modify workflow 04 (MercadoPago) to trigger a new dispatch webhook (workflow 11) after payment confirmed. Two scheduler workflows (12, 13) handle scheduled deliveries and stale-dispatch alerts every 5 minutes. No new DB migrations needed — schema already complete in migration 001.

**Tech Stack:** n8n Cloud · Supabase (REST API) · WhatsApp Cloud API · Google Maps URL construction (no API key needed — just URL encoding)

**Spec:** `docs/superpowers/specs/2026-05-24-jenny-floreria-phase5-delivery-dispatch-design.md`

---

## File Map

| File | Action | What it does |
|---|---|---|
| `n8n/workflows/01-whatsapp-inbound.json` | Modify | Add driver detection branch after "Is Status Update?" |
| `n8n/workflows/04-mercadopago-webhook.json` | Modify | Trigger dispatch webhook after payment confirmed |
| `n8n/workflows/11-dispatch-driver.json` | New | Broadcast order to available drivers via WhatsApp |
| `n8n/workflows/12-scheduled-dispatch.json` | New | Every 5 min: find scheduled orders due in 35–50 min → trigger dispatch |
| `n8n/workflows/13-stuck-dispatch-alert.json` | New | Every 5 min: find paid orders with no driver > 15 min → alert Carmelita |

---

## n8n Pattern Reference

All HTTP request nodes use these header patterns (copy exactly):
```json
{ "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" }
{ "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" }
{ "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
```

URL patterns:
```
Supabase GET:  "={{ $env.SUPABASE_URL }}/rest/v1/TABLE?filter&select=..."
Supabase PATCH: "={{ $env.SUPABASE_URL }}/rest/v1/TABLE?id=eq.{{ $json.id }}"
WhatsApp POST: "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages"
```

WhatsApp body pattern:
```javascript
JSON.stringify({ messaging_product: 'whatsapp', to: PHONE, type: 'text', text: { body: 'MESSAGE' } })
```

---

## Task 1: Create Feature Branch

**Files:** git only

- [ ] **Step 1: Create branch**
```bash
cd /Users/montanaink/Projects/jenny-floreria
git checkout -b feat/phase5-delivery-dispatch
```

- [ ] **Step 2: Verify**
```bash
git log --oneline -3
```
Expected: on `feat/phase5-delivery-dispatch`, last commit is `9ff9412 feat: Phase 4 Voice AI Agent`

---

## Task 2: Workflow 11 — Dispatch Driver

**Files:**
- Create: `n8n/workflows/11-dispatch-driver.json`

This workflow receives a POST with `{ order_id }`, loads the order, checks timing, queries available drivers, and broadcasts WhatsApp dispatch messages.

Node layout: `dd-01 → dd-02(respond200) → dd-03 → dd-04 → dd-05 → dd-06 → dd-07[no drivers→dd-08, has drivers→dd-09→dd-10]`

- [ ] **Step 1: Create the file**

```json
{
  "name": "11 - Dispatch Driver",
  "nodes": [
    {
      "id": "dd-01",
      "name": "Dispatch Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [240, 400],
      "webhookId": "dispatch-driver",
      "parameters": {
        "path": "dispatch-driver",
        "httpMethod": "POST",
        "responseMode": "onReceived",
        "responseCode": "200",
        "options": {}
      }
    },
    {
      "id": "dd-02",
      "name": "Load Order",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [480, 400],
      "parameters": {
        "method": "GET",
        "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?id=eq.{{ $json.body.order_id }}&select=id,order_number,status,total_mxn,delivery_address,delivery_notes,occasion,occasion_card_msg,scheduled_delivery_at,customers(name,whatsapp_phone)",
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
      "id": "dd-03",
      "name": "Check Dispatch Timing",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [720, 400],
      "parameters": {
        "jsCode": "const order = $input.first().json[0]\nif (!order) return [{ json: { skip: true, reason: 'order not found' } }]\nconst sched = order.scheduled_delivery_at\nif (sched) {\n  const minsUntil = (new Date(sched).getTime() - Date.now()) / 60000\n  if (minsUntil > 50) return [{ json: { skip: true, reason: 'too early', minsUntil } }]\n}\nconst addr = order.delivery_address || {}\nconst addrStr = [addr.street, addr.colonia, 'Guasave, Sinaloa'].filter(Boolean).join(', ')\nconst mapsUrl = `https://maps.google.com/?q=${encodeURIComponent(addrStr)}`\nconst deliveryTime = sched\n  ? new Date(sched).toLocaleString('es-MX', { weekday: 'short', day: 'numeric', month: 'short', hour: '2-digit', minute: '2-digit' })\n  : 'Hoy lo antes posible'\nreturn [{ json: { skip: false, order, mapsUrl, deliveryTime, addrStr } }]"
      }
    },
    {
      "id": "dd-04",
      "name": "Should Dispatch?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [960, 400],
      "parameters": {
        "conditions": {
          "conditions": [
            {
              "id": "cond-dd-01",
              "leftValue": "={{ $json.skip }}",
              "rightValue": false,
              "operator": { "type": "boolean", "operation": "equals" }
            }
          ],
          "combinator": "and"
        }
      }
    },
    {
      "id": "dd-05",
      "name": "Query Available Drivers",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1200, 320],
      "parameters": {
        "method": "GET",
        "url": "={{ $env.SUPABASE_URL }}/rest/v1/drivers?active=eq.true&current_order_id=is.null&select=id,name,whatsapp_phone",
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
      "id": "dd-06",
      "name": "Has Available Drivers?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1440, 320],
      "parameters": {
        "conditions": {
          "conditions": [
            {
              "id": "cond-dd-02",
              "leftValue": "={{ $json.length }}",
              "rightValue": 0,
              "operator": { "type": "number", "operation": "gt" }
            }
          ],
          "combinator": "and"
        }
      }
    },
    {
      "id": "dd-07",
      "name": "Alert Carmelita No Drivers",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1680, 440],
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $env.CARMELITA_PERSONAL_WHATSAPP, type: 'text', text: { body: '⚠️ *Sin repartidores disponibles*\\n\\nPedido #' + $('Check Dispatch Timing').first().json.order.order_number + ' necesita asignación manual.\\nCliente: ' + $('Check Dispatch Timing').first().json.order.customers.name + '\\nDirección: ' + $('Check Dispatch Timing').first().json.addrStr + '\\n\\nTodos los repartidores están ocupados.' } }) }}",
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
      "id": "dd-08",
      "name": "Build Dispatch Messages",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [1680, 240],
      "parameters": {
        "jsCode": "const drivers = $('Query Available Drivers').first().json\nconst ctx = $('Check Dispatch Timing').first().json\nconst order = ctx.order\nconst addr = order.delivery_address || {}\nconst addrLine = [addr.street, addr.cross_streets ? `Entre ${addr.cross_streets}` : null, `Col. ${addr.colonia}`, addr.landmark, addr.house_color ? `Casa ${addr.house_color}` : null].filter(Boolean).join('\\n   ')\nconst items = order.occasion ? `Para: ${order.occasion}` : ''\nreturn drivers.map(driver => ({\n  json: {\n    driverPhone: driver.whatsapp_phone.replace('+', ''),\n    message: `🌸 NUEVA ENTREGA - Jenny Florería\\nPedido #${order.order_number}\\n─────────────────────────\\n${items}\\n─────────────────────────\\n📍 ${addrLine}\\n🗺️ ${ctx.mapsUrl}\\n─────────────────────────\\n🕒 ${ctx.deliveryTime}\\n─────────────────────────\\nResponde ACEPTO para tomar el pedido\\nResponde RECHAZO si no puedes`\n  }\n}))"
      }
    },
    {
      "id": "dd-09",
      "name": "Send WhatsApp to Driver",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1920, 240],
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $json.driverPhone, type: 'text', text: { body: $json.message } }) }}",
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
    "Dispatch Webhook": {
      "main": [[{ "node": "Load Order", "type": "main", "index": 0 }]]
    },
    "Load Order": {
      "main": [[{ "node": "Check Dispatch Timing", "type": "main", "index": 0 }]]
    },
    "Check Dispatch Timing": {
      "main": [[{ "node": "Should Dispatch?", "type": "main", "index": 0 }]]
    },
    "Should Dispatch?": {
      "main": [
        [{ "node": "Query Available Drivers", "type": "main", "index": 0 }],
        []
      ]
    },
    "Query Available Drivers": {
      "main": [[{ "node": "Has Available Drivers?", "type": "main", "index": 0 }]]
    },
    "Has Available Drivers?": {
      "main": [
        [{ "node": "Build Dispatch Messages", "type": "main", "index": 0 }],
        [{ "node": "Alert Carmelita No Drivers", "type": "main", "index": 0 }]
      ]
    },
    "Build Dispatch Messages": {
      "main": [[{ "node": "Send WhatsApp to Driver", "type": "main", "index": 0 }]]
    }
  },
  "active": false,
  "settings": { "executionOrder": "v1" },
  "staticData": null,
  "tags": [],
  "triggerCount": 0,
  "updatedAt": "2026-05-24T00:00:00.000Z",
  "versionId": "phase5-dispatch-v1"
}
```

- [ ] **Step 2: Commit**
```bash
git add n8n/workflows/11-dispatch-driver.json
git commit -m "feat(phase5): add driver dispatch broadcast workflow (11)"
```

- [ ] **Step 3: Manual smoke test (after importing to n8n)**

POST to the workflow 11 Production URL with a known paid order ID:
```bash
curl -X POST https://YOUR-N8N.app.n8n.cloud/webhook/dispatch-driver \
  -H "Content-Type: application/json" \
  -d '{"order_id": "PASTE-A-REAL-ORDER-UUID-HERE"}'
```
Expected: all active+free drivers receive WhatsApp dispatch message within 3 seconds.

---

## Task 3: Workflow 12 — Scheduled Dispatch Checker

**Files:**
- Create: `n8n/workflows/12-scheduled-dispatch.json`

Runs every 5 minutes. Finds paid orders with `scheduled_delivery_at` between now+35min and now+50min that have no driver yet. Triggers workflow 11 for each.

- [ ] **Step 1: Create the file**

```json
{
  "name": "12 - Scheduled Dispatch Checker",
  "nodes": [
    {
      "id": "sd-01",
      "name": "Every 5 Minutes",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [240, 400],
      "parameters": {
        "rule": {
          "interval": [{ "field": "minutes", "minutesInterval": 5 }]
        }
      }
    },
    {
      "id": "sd-02",
      "name": "Build Time Window",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [480, 400],
      "parameters": {
        "jsCode": "const now = Date.now()\nconst from = new Date(now + 35 * 60 * 1000).toISOString()\nconst to = new Date(now + 50 * 60 * 1000).toISOString()\nreturn [{ json: { from, to } }]"
      }
    },
    {
      "id": "sd-03",
      "name": "Find Scheduled Orders Due",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [720, 400],
      "parameters": {
        "method": "GET",
        "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?status=eq.paid&driver_id=is.null&scheduled_delivery_at=gte.{{ $('Build Time Window').first().json.from }}&scheduled_delivery_at=lte.{{ $('Build Time Window').first().json.to }}&select=id",
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
      "id": "sd-04",
      "name": "Split Orders",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [960, 400],
      "parameters": {
        "jsCode": "const orders = $input.first().json\nif (!Array.isArray(orders) || orders.length === 0) return []\nreturn orders.map(o => ({ json: { order_id: o.id } }))"
      }
    },
    {
      "id": "sd-05",
      "name": "Trigger Dispatch",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1200, 400],
      "parameters": {
        "method": "POST",
        "url": "={{ $env.N8N_WEBHOOK_BASE_URL }}/webhook/dispatch-driver",
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ order_id: $json.order_id }) }}",
        "options": {}
      }
    }
  ],
  "connections": {
    "Every 5 Minutes": {
      "main": [[{ "node": "Build Time Window", "type": "main", "index": 0 }]]
    },
    "Build Time Window": {
      "main": [[{ "node": "Find Scheduled Orders Due", "type": "main", "index": 0 }]]
    },
    "Find Scheduled Orders Due": {
      "main": [[{ "node": "Split Orders", "type": "main", "index": 0 }]]
    },
    "Split Orders": {
      "main": [[{ "node": "Trigger Dispatch", "type": "main", "index": 0 }]]
    }
  },
  "active": false,
  "settings": { "executionOrder": "v1" },
  "staticData": null,
  "tags": [],
  "triggerCount": 0,
  "updatedAt": "2026-05-24T00:00:00.000Z",
  "versionId": "phase5-sched-dispatch-v1"
}
```

- [ ] **Step 2: Commit**
```bash
git add n8n/workflows/12-scheduled-dispatch.json
git commit -m "feat(phase5): add scheduled delivery dispatch checker (workflow 12)"
```

---

## Task 4: Workflow 13 — Stuck Dispatch Alert

**Files:**
- Create: `n8n/workflows/13-stuck-dispatch-alert.json`

Runs every 5 minutes. Finds paid orders with no driver where `updated_at < now - 15 min`. Alerts Carmelita per order.

- [ ] **Step 1: Create the file**

```json
{
  "name": "13 - Stuck Dispatch Alert",
  "nodes": [
    {
      "id": "sa-01",
      "name": "Every 5 Minutes",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.2,
      "position": [240, 400],
      "parameters": {
        "rule": {
          "interval": [{ "field": "minutes", "minutesInterval": 5 }]
        }
      }
    },
    {
      "id": "sa-02",
      "name": "Compute Cutoff Time",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [480, 400],
      "parameters": {
        "jsCode": "const cutoff = new Date(Date.now() - 15 * 60 * 1000).toISOString()\nreturn [{ json: { cutoff } }]"
      }
    },
    {
      "id": "sa-03",
      "name": "Find Stuck Orders",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [720, 400],
      "parameters": {
        "method": "GET",
        "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?status=eq.paid&driver_id=is.null&updated_at=lt.{{ $('Compute Cutoff Time').first().json.cutoff }}&select=id,order_number,customers(name,whatsapp_phone)",
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
      "id": "sa-04",
      "name": "Split Stuck Orders",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [960, 400],
      "parameters": {
        "jsCode": "const orders = $input.first().json\nif (!Array.isArray(orders) || orders.length === 0) return []\nreturn orders.map(o => ({ json: o }))"
      }
    },
    {
      "id": "sa-05",
      "name": "Alert Carmelita Stuck",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4,
      "position": [1200, 400],
      "parameters": {
        "method": "POST",
        "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
        "sendBody": true,
        "contentType": "json",
        "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $env.CARMELITA_PERSONAL_WHATSAPP, type: 'text', text: { body: '⚠️ *Sin respuesta de repartidores*\\n\\nPedido #' + $json.order_number + ' lleva más de 15 minutos sin repartidor asignado.\\nCliente: ' + ($json.customers?.name || 'Desconocido') + '\\n\\nPor favor asigna un repartidor manualmente.' } }) }}",
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
    "Every 5 Minutes": {
      "main": [[{ "node": "Compute Cutoff Time", "type": "main", "index": 0 }]]
    },
    "Compute Cutoff Time": {
      "main": [[{ "node": "Find Stuck Orders", "type": "main", "index": 0 }]]
    },
    "Find Stuck Orders": {
      "main": [[{ "node": "Split Stuck Orders", "type": "main", "index": 0 }]]
    },
    "Split Stuck Orders": {
      "main": [[{ "node": "Alert Carmelita Stuck", "type": "main", "index": 0 }]]
    }
  },
  "active": false,
  "settings": { "executionOrder": "v1" },
  "staticData": null,
  "tags": [],
  "triggerCount": 0,
  "updatedAt": "2026-05-24T00:00:00.000Z",
  "versionId": "phase5-stuck-alert-v1"
}
```

- [ ] **Step 2: Commit**
```bash
git add n8n/workflows/13-stuck-dispatch-alert.json
git commit -m "feat(phase5): add stuck dispatch alert scheduler (workflow 13)"
```

---

## Task 5: Modify Workflow 04 — Trigger Dispatch After Payment

**Files:**
- Modify: `n8n/workflows/04-mercadopago-webhook.json`

Add one new node `mp-08` after `mp-07` (Notify Customer Payment OK). This node POSTs the `order_id` to workflow 11's dispatch webhook.

- [ ] **Step 1: Open `n8n/workflows/04-mercadopago-webhook.json` and add node `mp-08` to the `"nodes"` array**

Add after the last node in the array:
```json
{
  "id": "mp-08",
  "name": "Trigger Driver Dispatch",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [1920, 240],
  "parameters": {
    "method": "POST",
    "url": "={{ $env.N8N_WEBHOOK_BASE_URL }}/webhook/dispatch-driver",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ order_id: $('Get Payment Details').first().json.external_reference }) }}",
    "options": {}
  }
}
```

- [ ] **Step 2: Add the connection for `mp-08` to the `"connections"` object**

Change the existing `"Notify Customer Payment OK"` connection from ending at `[]` to:
```json
"Notify Customer Payment OK": {
  "main": [[{ "node": "Trigger Driver Dispatch", "type": "main", "index": 0 }]]
}
```

- [ ] **Step 3: Commit**
```bash
git add n8n/workflows/04-mercadopago-webhook.json
git commit -m "feat(phase5): trigger driver dispatch after MercadoPago payment confirmed"
```

---

## Task 6: Modify Workflow 01 — Driver Detection & Reply Handling

**Files:**
- Modify: `n8n/workflows/01-whatsapp-inbound.json`

This is the largest change. Insert a driver-detection branch between "Is Status Update?" and "Load Customer". If the sender is a known driver phone → route to driver-reply logic. Otherwise → existing customer flow (unchanged).

**New nodes to add (insert into `"nodes"` array):**

### Driver Detection Nodes

- [ ] **Step 1: Add node `node-d1` — Check Sender Is Driver**

```json
{
  "id": "node-d1",
  "name": "Check Sender Is Driver",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [920, 560],
  "parameters": {
    "method": "GET",
    "url": "={{ $env.SUPABASE_URL }}/rest/v1/drivers?whatsapp_phone=eq.{{ $('Extract Message').first().json.phone }}&active=eq.true&select=id,name,current_order_id",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
        { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" }
      ]
    },
    "options": {}
  }
}
```

- [ ] **Step 2: Add node `node-d2` — Is Driver?**

```json
{
  "id": "node-d2",
  "name": "Is Driver?",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "position": [1160, 560],
  "parameters": {
    "conditions": {
      "conditions": [
        {
          "id": "cond-d1",
          "leftValue": "={{ $json.length }}",
          "rightValue": 0,
          "operator": { "type": "number", "operation": "gt" }
        }
      ],
      "combinator": "and"
    }
  }
}
```

- [ ] **Step 3: Add node `node-d3` — Parse Driver Keyword**

```json
{
  "id": "node-d3",
  "name": "Parse Driver Keyword",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [1400, 480],
  "parameters": {
    "jsCode": "const msg = ($('Extract Message').first().json.messageText || '').trim().toUpperCase()\nconst driver = $('Check Sender Is Driver').first().json[0]\nlet keyword = 'OTHER'\nif (msg.startsWith('ACEPTO')) keyword = 'ACEPTO'\nelse if (msg.startsWith('RECHAZO')) keyword = 'RECHAZO'\nelse if (msg.startsWith('ENTREGADO')) keyword = 'ENTREGADO'\nelse if (msg.startsWith('PROBLEMA')) keyword = 'PROBLEMA'\nreturn [{ json: { keyword, driverId: driver.id, driverName: driver.name, currentOrderId: driver.current_order_id, driverPhone: $('Extract Message').first().json.phone } }]"
  }
}
```

### ACEPTO Branch Nodes

- [ ] **Step 4: Add node `node-d4` — Is ACEPTO?**

```json
{
  "id": "node-d4",
  "name": "Is ACEPTO?",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "position": [1640, 400],
  "parameters": {
    "conditions": {
      "conditions": [
        {
          "id": "cond-d2",
          "leftValue": "={{ $json.keyword }}",
          "rightValue": "ACEPTO",
          "operator": { "type": "string", "operation": "equals" }
        }
      ],
      "combinator": "and"
    }
  }
}
```

- [ ] **Step 5: Add node `node-d5` — Find Pending Order**

```json
{
  "id": "node-d5",
  "name": "Find Pending Order",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [1880, 320],
  "parameters": {
    "method": "GET",
    "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?status=eq.paid&driver_id=is.null&order=updated_at.desc&limit=1&select=id,order_number,total_mxn,delivery_address,occasion,occasion_card_msg,scheduled_delivery_at,customers(name,whatsapp_phone)",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
        { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" }
      ]
    },
    "options": {}
  }
}
```

- [ ] **Step 6: Add node `node-d5b` — Has Pending Order?**

```json
{
  "id": "node-d5b",
  "name": "Has Pending Order?",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "position": [2120, 320],
  "parameters": {
    "conditions": {
      "conditions": [
        {
          "id": "cond-d3",
          "leftValue": "={{ $json.length }}",
          "rightValue": 0,
          "operator": { "type": "number", "operation": "gt" }
        }
      ],
      "combinator": "and"
    }
  }
}
```

- [ ] **Step 7: Add node `node-d5c` — Tell Driver No Orders**

```json
{
  "id": "node-d5c",
  "name": "Tell Driver No Orders",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [2360, 420],
  "parameters": {
    "method": "POST",
    "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $('Parse Driver Keyword').first().json.driverPhone.replace('+', ''), type: 'text', text: { body: 'No hay pedidos disponibles en este momento. ¡Gracias! 🌸' } }) }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
      ]
    },
    "options": {}
  }
}
```

- [ ] **Step 8: Add node `node-d6` — Atomic Assign Driver**

```json
{
  "id": "node-d6",
  "name": "Atomic Assign Driver",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [2360, 240],
  "parameters": {
    "method": "PATCH",
    "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?id=eq.{{ $('Find Pending Order').first().json[0].id }}&status=eq.paid&driver_id=is.null",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ driver_id: $('Parse Driver Keyword').first().json.driverId, status: 'driver_assigned', driver_payment_amount_mxn: 80, updated_at: new Date().toISOString() }) }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
        { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
        { "name": "Content-Type", "value": "application/json" },
        { "name": "Prefer", "value": "return=representation" }
      ]
    },
    "options": {}
  }
}
```

- [ ] **Step 9: Add node `node-d7` — Was Assigned?**

```json
{
  "id": "node-d7",
  "name": "Was Assigned?",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "position": [2600, 240],
  "parameters": {
    "conditions": {
      "conditions": [
        {
          "id": "cond-d4",
          "leftValue": "={{ $json.length }}",
          "rightValue": 0,
          "operator": { "type": "number", "operation": "gt" }
        }
      ],
      "combinator": "and"
    }
  }
}
```

- [ ] **Step 10: Add node `node-d7b` — Tell Driver Already Taken**

```json
{
  "id": "node-d7b",
  "name": "Tell Driver Already Taken",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [2840, 340],
  "parameters": {
    "method": "POST",
    "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $('Parse Driver Keyword').first().json.driverPhone.replace('+', ''), type: 'text', text: { body: 'Este pedido ya fue tomado. ¡Gracias de todas formas! 🌸' } }) }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
      ]
    },
    "options": {}
  }
}
```

- [ ] **Step 11: Add node `node-d8` — Update Driver current_order_id**

```json
{
  "id": "node-d8",
  "name": "Update Driver Current Order",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [2840, 160],
  "parameters": {
    "method": "PATCH",
    "url": "={{ $env.SUPABASE_URL }}/rest/v1/drivers?id=eq.{{ $('Parse Driver Keyword').first().json.driverId }}",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ current_order_id: $('Find Pending Order').first().json[0].id }) }}",
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
```

- [ ] **Step 12: Add node `node-d8b` — Build Driver Confirmation (Code)**

```json
{
  "id": "node-d8b",
  "name": "Build Driver Confirmation",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [3080, 160],
  "parameters": {
    "jsCode": "const order = $('Find Pending Order').first().json[0]\nconst addr = order.delivery_address || {}\nconst addrLine = [addr.street, addr.cross_streets ? `Entre ${addr.cross_streets}` : null, `Col. ${addr.colonia}`, addr.landmark, addr.house_color ? `Casa ${addr.house_color}` : null].filter(Boolean).join('\\n   ')\nconst mapsUrl = `https://maps.google.com/?q=${encodeURIComponent([addr.street, addr.colonia, 'Guasave, Sinaloa'].filter(Boolean).join(', '))}`\nconst deliveryTime = order.scheduled_delivery_at\n  ? new Date(order.scheduled_delivery_at).toLocaleString('es-MX', { weekday: 'short', day: 'numeric', month: 'short', hour: '2-digit', minute: '2-digit' })\n  : 'Lo antes posible'\nconst cardMsg = order.occasion_card_msg ? `\\n📝 Mensaje: \"${order.occasion_card_msg}\"` : ''\nconst driverPhone = $('Parse Driver Keyword').first().json.driverPhone.replace('+', '')\nconst customerPhone = order.customers.whatsapp_phone.replace('+', '')\nreturn [{\n  json: {\n    driverPhone,\n    customerPhone,\n    driverMsg: `✅ ¡Pedido asignado!\\nPedido #${order.order_number} — ${order.customers.name}\\n📍 ${addrLine}\\n🗺️ ${mapsUrl}${cardMsg}\\n💐 ${order.occasion || 'Sin ocasión específica'}\\n🕒 Entregar: ${deliveryTime}\\n💵 Cobro al entregar: $0 (ya pagado)\\n\\nCuando entregues responde: ENTREGADO\\nSi hay problema responde: PROBLEMA`,\n    customerMsg: `🛵 ¡Tu pedido está en camino!\\n${$('Parse Driver Keyword').first().json.driverName} lo entregará en aproximadamente 20–30 minutos.\\nPedido #${order.order_number} · Jenny Florería 🌸`,\n    carmelitaMsg: `🚴 ${$('Parse Driver Keyword').first().json.driverName} tomó pedido #${order.order_number}\\nCliente: ${order.customers.name}\\nDirección: ${addr.street}, Col. ${addr.colonia}`\n  }\n}]"
  }
}
```

- [ ] **Step 13: Add node `node-d8c` — Confirm to Winning Driver**

```json
{
  "id": "node-d8c",
  "name": "Confirm to Winning Driver",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [3320, 80],
  "parameters": {
    "method": "POST",
    "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $json.driverPhone, type: 'text', text: { body: $json.driverMsg } }) }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
      ]
    },
    "options": {}
  }
}
```

- [ ] **Step 14: Add node `node-d8d` — Notify Customer Driver Assigned**

```json
{
  "id": "node-d8d",
  "name": "Notify Customer Driver Assigned",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [3320, 200],
  "parameters": {
    "method": "POST",
    "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $('Build Driver Confirmation').first().json.customerPhone, type: 'text', text: { body: $('Build Driver Confirmation').first().json.customerMsg } }) }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
      ]
    },
    "options": {}
  }
}
```

- [ ] **Step 15: Add node `node-d8e` — Notify Carmelita Driver Assigned**

```json
{
  "id": "node-d8e",
  "name": "Notify Carmelita Driver Assigned",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [3320, 320],
  "parameters": {
    "method": "POST",
    "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $env.CARMELITA_PERSONAL_WHATSAPP, type: 'text', text: { body: $('Build Driver Confirmation').first().json.carmelitaMsg } }) }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
      ]
    },
    "options": {}
  }
}
```

### ENTREGADO Branch Nodes

- [ ] **Step 16: Add node `node-d4b` — Is ENTREGADO?**

```json
{
  "id": "node-d4b",
  "name": "Is ENTREGADO?",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "position": [1640, 560],
  "parameters": {
    "conditions": {
      "conditions": [
        {
          "id": "cond-d5",
          "leftValue": "={{ $('Parse Driver Keyword').first().json.keyword }}",
          "rightValue": "ENTREGADO",
          "operator": { "type": "string", "operation": "equals" }
        }
      ],
      "combinator": "and"
    }
  }
}
```

- [ ] **Step 17: Add node `node-d10` — Load Order for Delivery**

```json
{
  "id": "node-d10",
  "name": "Load Order for Delivery",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [1880, 480],
  "parameters": {
    "method": "GET",
    "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?id=eq.{{ $('Parse Driver Keyword').first().json.currentOrderId }}&select=id,order_number,customers(name,whatsapp_phone)",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "apikey", "value": "={{ $env.SUPABASE_SERVICE_ROLE_KEY }}" },
        { "name": "Authorization", "value": "=Bearer {{ $env.SUPABASE_SERVICE_ROLE_KEY }}" }
      ]
    },
    "options": {}
  }
}
```

- [ ] **Step 18: Add node `node-d10b` — Has Active Order?**

```json
{
  "id": "node-d10b",
  "name": "Has Active Order?",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "position": [2120, 480],
  "parameters": {
    "conditions": {
      "conditions": [
        {
          "id": "cond-d6",
          "leftValue": "={{ $json.length }}",
          "rightValue": 0,
          "operator": { "type": "number", "operation": "gt" }
        }
      ],
      "combinator": "and"
    }
  }
}
```

- [ ] **Step 19: Add node `node-d11` — Mark Order Delivered**

```json
{
  "id": "node-d11",
  "name": "Mark Order Delivered",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [2360, 400],
  "parameters": {
    "method": "PATCH",
    "url": "={{ $env.SUPABASE_URL }}/rest/v1/orders?id=eq.{{ $('Load Order for Delivery').first().json[0].id }}",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ status: 'delivered', updated_at: new Date().toISOString() }) }}",
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
```

- [ ] **Step 20: Add node `node-d12` — Clear Driver**

```json
{
  "id": "node-d12",
  "name": "Clear Driver",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [2600, 400],
  "parameters": {
    "method": "PATCH",
    "url": "={{ $env.SUPABASE_URL }}/rest/v1/drivers?id=eq.{{ $('Parse Driver Keyword').first().json.driverId }}",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ current_order_id: null }) }}",
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
```

- [ ] **Step 21: Add node `node-d12b` — Notify Customer Delivered**

```json
{
  "id": "node-d12b",
  "name": "Notify Customer Delivered",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [2840, 320],
  "parameters": {
    "method": "POST",
    "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $('Load Order for Delivery').first().json[0].customers.whatsapp_phone.replace('+', ''), type: 'text', text: { body: '🌸 ¡Tu pedido fue entregado!\\nEsperamos que lo disfrutes mucho.\\n¡Gracias por elegir Jenny Florería! 🌺' } }) }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
      ]
    },
    "options": {}
  }
}
```

- [ ] **Step 22: Add node `node-d12c` — Notify Carmelita Delivered**

```json
{
  "id": "node-d12c",
  "name": "Notify Carmelita Delivered",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [2840, 440],
  "parameters": {
    "method": "POST",
    "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $env.CARMELITA_PERSONAL_WHATSAPP, type: 'text', text: { body: '✅ Pedido #' + $('Load Order for Delivery').first().json[0].order_number + ' entregado\\nRepartidor: ' + $('Parse Driver Keyword').first().json.driverName + '\\nCliente: ' + $('Load Order for Delivery').first().json[0].customers.name } }) }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
      ]
    },
    "options": {}
  }
}
```

### PROBLEMA Branch Nodes

- [ ] **Step 23: Add node `node-d4c` — Is PROBLEMA?**

```json
{
  "id": "node-d4c",
  "name": "Is PROBLEMA?",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "position": [1640, 720],
  "parameters": {
    "conditions": {
      "conditions": [
        {
          "id": "cond-d7",
          "leftValue": "={{ $('Parse Driver Keyword').first().json.keyword }}",
          "rightValue": "PROBLEMA",
          "operator": { "type": "string", "operation": "equals" }
        }
      ],
      "combinator": "and"
    }
  }
}
```

- [ ] **Step 24: Add node `node-d13` — Alert Carmelita PROBLEMA**

```json
{
  "id": "node-d13",
  "name": "Alert Carmelita PROBLEMA",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [1880, 640],
  "parameters": {
    "method": "POST",
    "url": "=https://graph.facebook.com/v20.0/{{ $env.WHATSAPP_PHONE_NUMBER_ID }}/messages",
    "sendBody": true,
    "contentType": "json",
    "body": "={{ JSON.stringify({ messaging_product: 'whatsapp', to: $env.CARMELITA_PERSONAL_WHATSAPP, type: 'text', text: { body: '🚨 PROBLEMA en entrega\\nRepartidor: ' + $('Parse Driver Keyword').first().json.driverName + '\\nHabla con él para resolver el problema.\\n\\nSu número: ' + $('Parse Driver Keyword').first().json.driverPhone } }) }}",
    "sendHeaders": true,
    "headerParameters": {
      "parameters": [
        { "name": "Authorization", "value": "=Bearer {{ $env.WHATSAPP_ACCESS_TOKEN }}" }
      ]
    },
    "options": {}
  }
}
```

### Shared Terminal Node

- [ ] **Step 25: Add node `node-d9` — Respond: 200 (Driver Handled)**

```json
{
  "id": "node-d9",
  "name": "Respond: 200 (Driver Handled)",
  "type": "n8n-nodes-base.respondToWebhook",
  "typeVersion": 1,
  "position": [3560, 560],
  "parameters": {
    "respondWith": "text",
    "responseBody": "ok",
    "options": { "responseCode": 200 }
  }
}
```

### Connection Changes for Workflow 01

- [ ] **Step 26: Update connections object**

Change `"Is Status Update?"` false branch from `node-09` to `node-d1`:
```json
"Is Status Update?": {
  "main": [
    [{ "node": "Respond: 200 (Status Ignored)", "type": "main", "index": 0 }],
    [{ "node": "Check Sender Is Driver", "type": "main", "index": 0 }]
  ]
}
```

Add all new connections:
```json
"Check Sender Is Driver": {
  "main": [[{ "node": "Is Driver?", "type": "main", "index": 0 }]]
},
"Is Driver?": {
  "main": [
    [{ "node": "Parse Driver Keyword", "type": "main", "index": 0 }],
    [{ "node": "Load Customer", "type": "main", "index": 0 }]
  ]
},
"Parse Driver Keyword": {
  "main": [[{ "node": "Is ACEPTO?", "type": "main", "index": 0 }]]
},
"Is ACEPTO?": {
  "main": [
    [{ "node": "Find Pending Order", "type": "main", "index": 0 }],
    [{ "node": "Is ENTREGADO?", "type": "main", "index": 0 }]
  ]
},
"Find Pending Order": {
  "main": [[{ "node": "Has Pending Order?", "type": "main", "index": 0 }]]
},
"Has Pending Order?": {
  "main": [
    [{ "node": "Atomic Assign Driver", "type": "main", "index": 0 }],
    [{ "node": "Tell Driver No Orders", "type": "main", "index": 0 }]
  ]
},
"Atomic Assign Driver": {
  "main": [[{ "node": "Was Assigned?", "type": "main", "index": 0 }]]
},
"Was Assigned?": {
  "main": [
    [{ "node": "Update Driver Current Order", "type": "main", "index": 0 }],
    [{ "node": "Tell Driver Already Taken", "type": "main", "index": 0 }]
  ]
},
"Update Driver Current Order": {
  "main": [[{ "node": "Build Driver Confirmation", "type": "main", "index": 0 }]]
},
"Build Driver Confirmation": {
  "main": [[{ "node": "Confirm to Winning Driver", "type": "main", "index": 0 }]]
},
"Confirm to Winning Driver": {
  "main": [[{ "node": "Notify Customer Driver Assigned", "type": "main", "index": 0 }]]
},
"Notify Customer Driver Assigned": {
  "main": [[{ "node": "Notify Carmelita Driver Assigned", "type": "main", "index": 0 }]]
},
"Notify Carmelita Driver Assigned": {
  "main": [[{ "node": "Respond: 200 (Driver Handled)", "type": "main", "index": 0 }]]
},
"Tell Driver Already Taken": {
  "main": [[{ "node": "Respond: 200 (Driver Handled)", "type": "main", "index": 0 }]]
},
"Tell Driver No Orders": {
  "main": [[{ "node": "Respond: 200 (Driver Handled)", "type": "main", "index": 0 }]]
},
"Is ENTREGADO?": {
  "main": [
    [{ "node": "Load Order for Delivery", "type": "main", "index": 0 }],
    [{ "node": "Is PROBLEMA?", "type": "main", "index": 0 }]
  ]
},
"Load Order for Delivery": {
  "main": [[{ "node": "Has Active Order?", "type": "main", "index": 0 }]]
},
"Has Active Order?": {
  "main": [
    [{ "node": "Mark Order Delivered", "type": "main", "index": 0 }],
    [{ "node": "Respond: 200 (Driver Handled)", "type": "main", "index": 0 }]
  ]
},
"Mark Order Delivered": {
  "main": [[{ "node": "Clear Driver", "type": "main", "index": 0 }]]
},
"Clear Driver": {
  "main": [[{ "node": "Notify Customer Delivered", "type": "main", "index": 0 }]]
},
"Notify Customer Delivered": {
  "main": [[{ "node": "Notify Carmelita Delivered", "type": "main", "index": 0 }]]
},
"Notify Carmelita Delivered": {
  "main": [[{ "node": "Respond: 200 (Driver Handled)", "type": "main", "index": 0 }]]
},
"Is PROBLEMA?": {
  "main": [
    [{ "node": "Alert Carmelita PROBLEMA", "type": "main", "index": 0 }],
    [{ "node": "Respond: 200 (Driver Handled)", "type": "main", "index": 0 }]
  ]
},
"Alert Carmelita PROBLEMA": {
  "main": [[{ "node": "Respond: 200 (Driver Handled)", "type": "main", "index": 0 }]]
}
```

- [ ] **Step 27: Commit**
```bash
git add n8n/workflows/01-whatsapp-inbound.json
git commit -m "feat(phase5): add driver detection and reply handling to WhatsApp inbound workflow"
```

---

## Task 7: Update README — Phase 5 Setup

**Files:**
- Modify: `n8n/README.md`

Append after the Phase 4 section. This is the operator guide Carmelita/Montana reads to set up Phase 5.

- [ ] **Step 1: Append the following to `n8n/README.md`**

```markdown
---

## Phase 5 Setup — Delivery Dispatch

### Prerequisites
- Phases 1–4 complete and active
- Up to 5 drivers ready with their WhatsApp numbers

### Step 1: Add Drivers to Supabase

In Supabase → SQL Editor, run (replace names and numbers with real driver info):
```sql
INSERT INTO drivers (name, whatsapp_phone, active, preferred_payment)
VALUES
  ('Marco',    '+521668XXXXXX', true, 'cash'),
  ('Luis',     '+521668XXXXXX', true, 'cash'),
  ('Roberto',  '+521668XXXXXX', true, 'transfer'),
  ('Carlos',   '+521668XXXXXX', true, 'cash'),
  ('Fernando', '+521668XXXXXX', true, 'transfer');
```

For drivers who prefer transfer, also add their CLABE:
```sql
UPDATE drivers SET bank_clabe = '012345678901234567', bank_name = 'BBVA'
WHERE whatsapp_phone = '+521668XXXXXX';
```

### Step 2: Import New Workflows

In n8n Cloud → Workflows → Import from file:
1. `n8n/workflows/11-dispatch-driver.json`
2. `n8n/workflows/12-scheduled-dispatch.json`
3. `n8n/workflows/13-stuck-dispatch-alert.json`

Activate all 3.

### Step 3: Re-import Modified Workflows

Re-import (overwrite) the updated versions:
1. `n8n/workflows/04-mercadopago-webhook.json` — now triggers dispatch after payment
2. `n8n/workflows/01-whatsapp-inbound.json` — now detects driver replies

After re-importing 01, paste code into ALL code nodes (same as Phase 3 step 6).
After re-importing 04, no code paste needed — only JSON structure changed.

### Step 4: Test Full Flow

**Test dispatch:**
1. Create a paid order in Supabase OR let a real order come through and pay
2. Within 5 seconds, all available drivers should receive the WhatsApp dispatch message

**Test ACEPTO:**
1. Reply ACEPTO from a driver's phone
2. Expect: driver gets confirmation, customer gets "en camino" message, Carmelita gets notification

**Test ENTREGADO:**
1. Reply ENTREGADO from assigned driver's phone
2. Expect: customer gets "entregado" message, Carmelita gets delivery confirmation

**Test PROBLEMA:**
1. Reply PROBLEMA from a driver's phone
2. Expect: Carmelita gets emergency WhatsApp alert

**Test stuck dispatch:**
1. Create a paid order, do NOT reply ACEPTO from any driver
2. Wait 15+ minutes (or temporarily change cutoff to 1 min in workflow 13 Code node)
3. Expect: Carmelita gets "sin respuesta" alert

### Driver Keywords Reference

Share this with all drivers:

| Palabra | Cuándo usarla |
|---|---|
| **ACEPTO** | Para tomar un pedido cuando recibes la notificación |
| **RECHAZO** | Si no puedes tomar el pedido |
| **ENTREGADO** | Cuando terminas de entregar |
| **PROBLEMA** | Si hay algún problema durante la entrega |

### Troubleshooting

**Drivers not receiving dispatch message:**
- Confirm workflow 11 is active
- Confirm `N8N_WEBHOOK_BASE_URL` variable is set in n8n
- Check Supabase: `SELECT * FROM drivers WHERE active = true AND current_order_id IS NULL` — should return driver rows

**ACEPTO not assigning:**
- Check workflow 01 execution log — find "Is Driver?" node — if it shows `false`, driver phone in Supabase may not match exactly (format: `+521668XXXXXX`)
- Verify driver phone format with: `SELECT whatsapp_phone FROM drivers`

**Order already delivered but driver still shows busy:**
- Supabase SQL: `UPDATE drivers SET current_order_id = NULL WHERE id = 'DRIVER-UUID'`
```

- [ ] **Step 2: Commit**
```bash
git add n8n/README.md
git commit -m "docs(phase5): add delivery dispatch setup guide to n8n README"
```

---

## Task 8: Merge to Main

**Files:** git only

- [ ] **Step 1: Verify all workflows exist**
```bash
ls n8n/workflows/
```
Expected files: `01` through `13` (10 workflow files total, including 06–10 from Phase 4)

- [ ] **Step 2: Merge to main**
```bash
git checkout main
git merge feat/phase5-delivery-dispatch --no-ff -m "feat: Phase 5 Delivery Dispatch (WhatsApp driver automation)"
```

- [ ] **Step 3: Verify**
```bash
git log --oneline -5
```
Expected: Phase 5 merge commit at top, Phase 4 merge below it.

---

## Verification Checklist

End-to-end test sequence (run after all n8n imports are active):

1. **Dispatch fires on payment:** Pay a test order via MercadoPago sandbox → all available drivers receive WhatsApp within 5 seconds
2. **First ACEPTO wins:** Two drivers reply ACEPTO quickly → only first gets confirmation, second gets "ya tomado"
3. **Customer notified:** After ACEPTO → customer receives "en camino" WhatsApp within 5 seconds
4. **Carmelita notified:** After ACEPTO → Carmelita receives driver + order summary
5. **ENTREGADO flow:** Driver replies ENTREGADO → customer receives "entregado" → Supabase order `status='delivered'` → driver `current_order_id = null`
6. **PROBLEMA alert:** Driver replies PROBLEMA → Carmelita receives emergency WhatsApp
7. **No drivers:** Temporarily mark all drivers as busy (`UPDATE drivers SET current_order_id = 'some-uuid'`) → dispatch fires → Carmelita gets "sin repartidores" alert
8. **Stuck dispatch:** Pay order, no ACEPTO for 15 min → Carmelita gets "sin respuesta" alert
9. **Customer messages still work:** Send a regular WhatsApp from non-driver phone → Jenny AI responds normally (driver branch not triggered)
