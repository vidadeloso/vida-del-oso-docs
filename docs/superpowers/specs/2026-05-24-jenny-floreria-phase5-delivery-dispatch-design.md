# Jenny Florería — Phase 5: Delivery Dispatch Design

**Date:** 2026-05-24  
**Status:** Approved  
**Author:** Montana + Claude  

---

## Goal

Automate delivery dispatch entirely over WhatsApp. When an order is paid, drivers get notified automatically, the first to accept gets assigned, and everyone — customer, driver, and Carmelita — gets a clear update at each step. No app. No dashboard needed for dispatch. Just WhatsApp messages in plain Spanish.

---

## Who Does What

| Person | Their experience |
|---|---|
| **Customer** | Receives automatic WhatsApp updates. Never needs to ask "where is my order?" |
| **Driver** | Gets a WhatsApp, replies one word (ACEPTO or RECHAZO), delivers, replies ENTREGADO |
| **Carmelita** | Gets a WhatsApp summary at each step. Only involved when something goes wrong |

---

## No New Database Migrations

All required tables and columns already exist from Migration 001:

- `drivers` table: `id`, `name`, `whatsapp_phone`, `active`, `current_order_id`, `preferred_payment`, `bank_clabe`, `bank_name`
- `orders` table: `driver_id`, `driver_payment_method`, `driver_payment_status`, `driver_payment_amount_mxn`, `delivery_photo_url`, status values `driver_assigned`, `in_delivery`, `delivered`
- `app_settings` table: `driver_rate_per_delivery_mxn` (default $80 MXN)

---

## Full Flow: Step by Step

```
Customer pays via MercadoPago
        │
        ▼
Workflow 04 confirms payment → sends "¡Pago confirmado!" to customer
        │
        ▼
Workflow 04 triggers Workflow 11 (dispatch) with order_id
        │
        ▼
Workflow 11 checks scheduled_delivery_at:
  ┌─────────────────────────────────┐
  │ NULL or < 45 min away → ASAP   │
  │ Future → skip (scheduler picks │
  │ it up 45 min before)           │
  └─────────────────────────────────┘
        │ (ASAP path)
        ▼
Query available drivers (active=true AND current_order_id IS NULL)
        │
  ┌─────┴──────────────┐
  │ None available     │ One or more available
  ▼                    ▼
Alert Carmelita:    Broadcast WhatsApp to ALL available drivers
"⚠️ Sin repartidores"
        │
        ▼ (driver replies ACEPTO)
Atomic PATCH: orders SET status='driver_assigned', driver_id=X
  WHERE id=Y AND status='paid' AND driver_id IS NULL
        │
  ┌─────┴────────────────────────────┐
  │ Updated (driver won)             │ Empty (another driver was faster)
  ▼                                  ▼
Confirm to winning driver          "Este pedido ya fue tomado. ¡Gracias!"
Notify other drivers: "Ya tomado"
Notify customer: "Tu pedido está en camino 🛵"
Notify Carmelita: "Marco tomó pedido #042"
        │
        ▼ (driver replies ENTREGADO)
Update: status='delivered', driver.current_order_id=null
Notify customer: "¡Tu pedido fue entregado! 🌸"
Notify Carmelita: "✅ Pedido #042 entregado"
```

---

## WhatsApp Messages — Exact Text

### To Driver (dispatch broadcast)
```
🌸 NUEVA ENTREGA - Jenny Florería
Pedido #042
─────────────────────────
Flores: 2 arreglos de rosas rojas
Para: Cumpleaños de Ana
─────────────────────────
📍 Calle Juárez 45
   Entre Obregón y Zaragoza
   Col. Centro
   Frente al OXXO · Casa blanca
🗺️ https://maps.google.com/?q=Calle+Juárez+45,Guasave,Sinaloa
─────────────────────────
🕒 Hoy lo antes posible
─────────────────────────
Responde ACEPTO para tomar el pedido
Responde RECHAZO si no puedes
```

### To Driver (assigned confirmation)
```
✅ ¡Pedido asignado!
Pedido #042 — Ana García
📍 Calle Juárez 45, Entre Obregón y Zaragoza, Col. Centro
🗺️ https://maps.google.com/?q=...
💐 Flores: 2 arreglos de rosas rojas
📝 Mensaje: "Feliz cumpleaños mamá"
🕒 Entregar lo antes posible
💵 Cobro al entregar: $0 (ya pagado)

Cuando entregues, responde: ENTREGADO
Si hay algún problema, responde: PROBLEMA
```

### To Driver (rejected — another driver took it)
```
Este pedido ya fue tomado. ¡Gracias de todas formas! 🌸
```

### To Customer (driver assigned)
```
🛵 ¡Tu pedido está en camino!
Marco lo entregará en aproximadamente 20–30 minutos.
Pedido #042 · Jenny Florería 🌸
```

### To Customer (delivered)
```
🌸 ¡Tu pedido fue entregado!
Esperamos que lo disfrutes mucho.
¡Gracias por elegir Jenny Florería! 🌺
```

### To Carmelita (driver assigned)
```
🚴 Marco tomó pedido #042
Cliente: Ana García
Dirección: Calle Juárez 45, Col. Centro
```

### To Carmelita (delivered)
```
✅ Pedido #042 entregado
Repartidor: Marco
Cliente: Ana García · Calle Juárez 45
```

### To Carmelita (no drivers available)
```
⚠️ Sin repartidores disponibles
Pedido #042 (Ana García) necesita asignación manual.
Todos los repartidores están ocupados o no responden.
```

### To Carmelita (no acceptance in 15 min)
```
⚠️ Sin respuesta de repartidores
Pedido #042 lleva 15+ minutos sin ser aceptado.
Por favor asigna un repartidor manualmente.
```

### To Carmelita (PROBLEMA reported)
```
🚨 PROBLEMA en entrega
Repartidor: Marco
Pedido #042 · Ana García
Calle Juárez 45, Col. Centro
Habla con Marco para resolver.
```

---

## Driver Reply Keywords

Workflow 01 checks if the incoming WhatsApp is from a known driver phone. If yes, it routes to driver reply logic. Keywords are case-insensitive:

| Keyword | Action |
|---|---|
| `ACEPTO` | Attempt atomic order assignment |
| `RECHAZO` | Log rejection (no further action, system waits) |
| `ENTREGADO` | Mark order delivered, free driver, notify all |
| `PROBLEMA` | Alert Carmelita immediately |
| Anything else | Ignore silently (do not reply) |

---

## Scheduled Delivery Flow

Workflow 12 runs every 5 minutes. It does two things:

**1. Trigger upcoming scheduled deliveries:**
Query: `orders WHERE status='paid' AND driver_id IS NULL AND scheduled_delivery_at BETWEEN now()+35min AND now()+50min`  
→ For each result: call Workflow 11 (dispatch) with that `order_id`

**2. Alert Carmelita about stuck dispatches:**
Query: `orders WHERE status='paid' AND driver_id IS NULL AND updated_at < now()-15min`  
→ For each result: WhatsApp Carmelita "⚠️ Sin respuesta de repartidores · Pedido #X"

---

## Google Maps URL Construction

Built in n8n from the `delivery_address` JSONB field:

```javascript
const addr = order.delivery_address  // { street, cross_streets, colonia, landmark, house_color, notes }
const query = encodeURIComponent(`${addr.street}, ${addr.colonia}, Guasave, Sinaloa`)
const mapsUrl = `https://maps.google.com/?q=${query}`
```

---

## Files to Build

| File | Action | Purpose |
|---|---|---|
| `n8n/workflows/01-whatsapp-inbound.json` | Modify | Add driver detection branch before customer routing |
| `n8n/workflows/04-mercadopago-webhook.json` | Modify | Trigger dispatch after payment confirmed |
| `n8n/workflows/11-dispatch-driver.json` | New | Broadcast to available drivers |
| `n8n/workflows/12-dispatch-scheduler.json` | New | 5-min check: scheduled orders + stuck dispatches |

**No new Supabase migrations needed.**

---

## n8n Workflow Sketches

### Workflow 01 — Modified (driver detection branch)

Add two nodes right after "Extract Message":

```
Extract Message
      │
      ▼
Is Sender a Driver? (HTTP GET)
  GET /rest/v1/drivers?whatsapp_phone=eq.{sender}&active=eq.true&select=id,name,current_order_id
      │
  ┌───┴────────────┐
  │ found          │ not found
  ▼                ▼
Driver Reply     Existing customer flow (unchanged)
Handler branch
```

Driver Reply Handler branch (Code node):
```javascript
const keyword = message.trim().toUpperCase()
const driver = driverRow  // from previous query
// routes to: handle-acepto / handle-rechazo / handle-entregado / handle-problema / ignore
```

### Workflow 11 — Dispatch Driver

```
Webhook (POST /webhook/dispatch-driver, receives { order_id })
      │
      ▼
Load Order (GET orders?id=eq.{order_id}&select=*,customers(name,whatsapp_phone))
      │
      ▼
Check scheduled_delivery_at (Code node — skip if future > 45 min)
      │
      ▼
Query Available Drivers (GET drivers?active=eq.true&current_order_id=is.null)
      │
  ┌───┴──────────────┐
  │ zero drivers     │ one or more drivers
  ▼                  ▼
Alert Carmelita     Build dispatch message (Code node — include Google Maps URL)
                          │
                          ▼
                    Send WhatsApp to each driver (SplitInBatches node)
```

### Workflow 12 — Dispatch Scheduler

```
Every 5 Minutes (Schedule Trigger)
      │
      ├──→ Query scheduled orders due in 35-50 min
      │       → For each: POST to workflow 11 webhook
      │
      └──→ Query stuck dispatches (paid, no driver, 15+ min old)
              → For each: WhatsApp alert to Carmelita
```

### Workflow 04 — Modified (add dispatch trigger)

Append after "Notify Customer Payment OK":

```
Notify Customer Payment OK
      │
      ▼
Trigger Dispatch (HTTP POST to n8n dispatch webhook with { order_id })
```

---

## Edge Cases Handled

| Situation | What happens |
|---|---|
| Two drivers ACEPTO at same time | Atomic PATCH on DB — first writer wins, second gets "ya tomado" |
| All drivers busy | Immediate WhatsApp alert to Carmelita |
| All drivers reject | No action — Carmelita alerted at 15-min mark |
| Driver says ENTREGADO with photo | Photo URL saved to `orders.delivery_photo_url` (Phase 6 dashboard shows it) |
| Unrecognized driver message | Silently ignored — no reply sent |
| Scheduled order, driver dispatched early | Scheduler skips if `driver_id IS NOT NULL` already set |

---

## Setup Steps (After Code Deployed)

1. **Add drivers to Supabase:** `INSERT INTO drivers (name, whatsapp_phone, preferred_payment) VALUES ('Marco', '521668XXXXXX', 'cash')`
2. **Import workflows 11 and 12** into n8n Cloud → activate both
3. **Re-import workflow 01** (updated with driver branch) → activate
4. **Re-import workflow 04** (updated with dispatch trigger) → activate
5. **Test:** Create a paid order → confirm dispatch WhatsApp arrives on driver's phone → reply ACEPTO → confirm customer + Carmelita notifications arrive
