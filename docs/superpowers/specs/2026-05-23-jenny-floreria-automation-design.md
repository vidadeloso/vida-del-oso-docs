# Jenny Florería — AI Business Automation System
**Design Spec · 2026-05-23**

---

## 1. Project Overview

**Client:** Carmelita, owner of Jenny Florería, Guasave, Sinaloa, México  
**Goal:** Fully automate order intake (WhatsApp + phone calls), payment collection, delivery dispatch, and owner notifications — so Carmelita receives orders, gets paid, and confirms deliveries without manually managing any step.  
**Current state:** 100% manual. WhatsApp messages read by hand. Orders tracked on paper. No digital inventory.  
**Budget:** $100–300 USD/month  
**Device:** Samsung Android (web dashboard, no native app)  
**Language:** Spanish (Mexican)  
**UX constraint:** System must be elderly-friendly. Carmelita is 50, many customers are older generation. Large text, high contrast, simple flows, numbered options, 3-tap maximum to any action.

---

## 2. Architecture Overview

```
CUSTOMERS
  │  WhatsApp message          Phone call
  ▼                            ▼
Meta Cloud API            Twilio (+52 668-XXX-XXXX)
  │                            │
  ▼                            ▼
n8n Cloud                 Vapi.ai (voice engine)
(orchestrator)            ElevenLabs voice clone
GPT-4o brain              GPT-4o brain
  │                            │
  └────────────┬───────────────┘
               │ reads / writes
               ▼
          SUPABASE
     (central database)
     flowers + photos
     customers · orders
     drivers · promo codes
               │
    ┌──────────┼──────────────┐
    ▼          ▼              ▼
MercadoPago  Driver       Vercel Dashboard
Payment      WhatsApp     (Carmelita's panel)
Links        Dispatch
```

**One database rules everything.** Both WhatsApp and voice agents read inventory from the same Supabase tables. Status changes propagate instantly to the dashboard via Supabase Realtime.

---

## 3. Sub-Systems & Build Order

| Phase | Sub-System | Dependency |
|---|---|---|
| 1 | CRM + Inventory (Supabase) | Foundation — everything else depends on this |
| 2 | WhatsApp AI Agent (n8n + GPT-4o) | Phase 1 |
| 3 | Payment (MercadoPago) | Phase 2 |
| 4 | Voice AI Agent (Vapi.ai + ElevenLabs) | Phase 1, 3 |
| 5 | Delivery Dispatch | Phase 3 |
| 6 | Owner Dashboard + Notifications | Phase 1+ |
| 7 | Public Customer Catalog (web) | Phase 1 |
| 8 | Event & Wedding Installment Payments | Phase 3 |

---

## 4. Data Layer (Supabase)

### Tables

```sql
-- Flower & ornament inventory
flowers (
  id                    uuid PRIMARY KEY,
  name                  text NOT NULL,
  description           text,
  price_mxn             numeric(10,2) NOT NULL,
  category              text,
    -- "flores" | "arreglos" | "ornamentos" | "coronas" |
    -- "centros_de_mesa" | "bouquets_boda" | "decoracion_evento"
  stock_count           integer DEFAULT 0,
  image_url             text,               -- Supabase Storage bucket
  occasion_tags         text[],             -- ["birthday","anniversary","funeral","quinceañera","wedding","general"]
  customization_options jsonb,
    -- {
    --   "sizes":  [{"label":"Pequeño","price_mxn":150},
    --              {"label":"Mediano","price_mxn":250},
    --              {"label":"Grande", "price_mxn":350}],
    --   "colors": ["Rojo","Blanco","Rosa","Amarillo","Morado"],
    --   "addons": [{"label":"Jarrón de vidrio","price_mxn":50},
    --              {"label":"Listón personalizado","price_mxn":20},
    --              {"label":"Tarjeta impresa","price_mxn":15}]
    -- }
    -- null = no customization available for this item
  active                boolean DEFAULT true,
  created_at            timestamptz DEFAULT now()
)

-- Customers
customers (
  id                uuid PRIMARY KEY,
  whatsapp_phone    text UNIQUE NOT NULL, -- "+521XXXXXXXXXX"
  name              text,
  address_default   jsonb,               -- saved address for repeat orders
  created_at        timestamptz DEFAULT now()
)

-- Orders
orders (
  id                  uuid PRIMARY KEY,
  order_number        serial,              -- human-readable #001, #002...
  customer_id         uuid REFERENCES customers,
  order_type          text DEFAULT 'standard',
    -- "standard" | "event" | "wedding"
  status              text DEFAULT 'pending',
    -- pending → confirmed → paid → driver_assigned → in_delivery → delivered → cancelled
    -- for event/wedding with payment plan: partially_paid is also valid between confirmed and paid
  items_json          jsonb,               -- snapshot of ordered items + customizations at time of order
  subtotal_mxn        numeric(10,2),
  discount_amount_mxn numeric(10,2) DEFAULT 0,
  total_mxn           numeric(10,2),
  amount_paid_mxn     numeric(10,2) DEFAULT 0,   -- tracks partial payments
  delivery_address    jsonb,               -- structured (see below)
  delivery_notes      text,
  occasion            text,               -- "birthday" / "anniversary" / "quinceañera" / "wedding" / etc.
  occasion_card_msg   text,               -- message to print on card
  event_date          date,               -- for weddings/events — the actual event day
  scheduled_delivery_at timestamptz,      -- null = ASAP
  driver_id           uuid REFERENCES drivers,
  mp_payment_id       text,               -- MercadoPago preference ID (standard orders)
  mp_payment_status   text,
  payment_plan_id     uuid,               -- FK to payment_plans (event/wedding orders)
  promo_code_id       uuid REFERENCES promo_codes,
  delivery_photo_url  text,              -- photo sent by driver on completion
  human_handoff       boolean DEFAULT false,
  created_at          timestamptz DEFAULT now(),
  updated_at          timestamptz DEFAULT now()
)

-- delivery_address JSON structure:
-- {
--   "street": "Calle Ángel Flores 142",
--   "cross_streets": "entre Obregón y Zaragoza",
--   "colonia": "Col. Centro",
--   "landmark": "frente al OXXO / junto a Iglesia San Miguel",
--   "house_description": "casa blanca, portón café",
--   "notes": "tocar el timbre, número no se ve"
-- }

-- Order line items
order_items (
  id                uuid PRIMARY KEY,
  order_id          uuid REFERENCES orders,
  flower_id         uuid REFERENCES flowers,
  quantity          integer NOT NULL,
  unit_price_mxn    numeric(10,2) NOT NULL,  -- base price at time of order
  customizations    jsonb,
    -- {
    --   "size":   "Mediano",
    --   "color":  "Rosa",
    --   "addons": ["Jarrón de vidrio","Listón personalizado"],
    --   "addon_total_mxn": 70,
    --   "custom_note": "listón color azul cielo si es posible"
    -- }
  line_total_mxn    numeric(10,2)            -- (unit_price + addon_total) × quantity
)

-- Delivery drivers
drivers (
  id                uuid PRIMARY KEY,
  name              text NOT NULL,
  whatsapp_phone    text UNIQUE NOT NULL,
  active            boolean DEFAULT true,
  current_order_id  uuid REFERENCES orders,  -- null = available
  created_at        timestamptz DEFAULT now()
)

-- Promotional codes
promo_codes (
  id                uuid PRIMARY KEY,
  code              text UNIQUE NOT NULL,     -- "CUMPLE15" / "AMOR20"
  discount_type     text NOT NULL,            -- "percentage" | "fixed_mxn"
  discount_value    numeric(10,2) NOT NULL,   -- 15 = 15% off, 50 = $50 MXN off
  valid_from        date,
  valid_until       date,
  max_uses          integer,                  -- null = unlimited
  times_used        integer DEFAULT 0,
  active            boolean DEFAULT true,
  created_at        timestamptz DEFAULT now()
)

-- Event / wedding payment plans
payment_plans (
  id                uuid PRIMARY KEY,
  order_id          uuid REFERENCES orders,
  total_amount_mxn  numeric(10,2) NOT NULL,
  amount_paid_mxn   numeric(10,2) DEFAULT 0,
  status            text DEFAULT 'active',
    -- "active" | "completed" | "overdue" | "cancelled"
  created_at        timestamptz DEFAULT now()
)

-- Individual installments within a payment plan
payment_installments (
  id                    uuid PRIMARY KEY,
  payment_plan_id       uuid REFERENCES payment_plans,
  installment_number    integer NOT NULL,        -- 1, 2, 3...
  amount_mxn            numeric(10,2) NOT NULL,
  due_date              date NOT NULL,
  status                text DEFAULT 'pending',
    -- "pending" | "paid" | "overdue"
  mp_payment_id         text,                    -- MercadoPago reference when paid
  paid_at               timestamptz,
  reminder_sent_at      timestamptz,             -- last reminder WhatsApp sent
  created_at            timestamptz DEFAULT now()
)

-- System notifications (dashboard feed)
notifications (
  id                uuid PRIMARY KEY,
  type              text,
    -- "new_order" | "payment" | "driver" | "delivery" | "low_stock"
    -- "handoff" | "installment_due" | "installment_paid" | "plan_overdue"
  message           text,
  order_id          uuid REFERENCES orders,
  read              boolean DEFAULT false,
  created_at        timestamptz DEFAULT now()
)
```

### Order Status Flow
```
Standard orders:
pending → confirmed → paid → driver_assigned → in_delivery → delivered
                                                           ↘ cancelled (any stage)

Event / wedding orders (payment plan):
pending → confirmed → partially_paid ─┐
                                       ├─ (installments paid over time)
                                       ▼
                                      paid → driver_assigned → in_delivery → delivered
```

Each status change fires a Supabase trigger → n8n webhook → appropriate WhatsApp messages sent.

### Low Stock Alert
When `stock_count < 5` on any active flower → n8n sends WhatsApp alert to Carmelita's personal number + creates notification record.

---

## 5. WhatsApp AI Agent

**Stack:** Meta WhatsApp Cloud API → n8n webhook → GPT-4o → Meta Cloud API reply

### Conversation Flow

```
Customer message received
        │
        ▼
Check orders.human_handoff for this customer phone
        │
  true ─┤─ false
        │         │
        ▼         ▼
  Forward msg   Load from Supabase:
  to Carmelita  • last 10 messages (conversation history)
  personal WA   • active flowers (stock_count > 0)
  (she replies  • open order for this customer if any
  directly)     • customer record (name, past orders)
                        │
                        ▼
                  GPT-4o (Spanish)
                  System prompt: Jenny persona
                  + inventory context injected
                  + conversation history
                        │
                  Returns JSON:
                  {
                    intent: "browse"|"order"|"promo"|
                            "status"|"faq"|"escalate",
                    reply_text: "...",
                    order_data: {...} | null,
                    escalate: boolean
                  }
                        │
           ┌────────────┼───────────────┐
           │            │               │
        escalate     intent=order   intent=promo
        =true           │               │
           │            ▼               ▼
     Human Handoff  Create order   Validate code
     Flow           in Supabase    → apply discount
                        │
                        ▼
                  MercadoPago link
                  sent to customer
```

### GPT-4o Capabilities (No Escalation Needed)
- Browse inventory ("¿qué rosas tienen?", "¿qué hay para un funeral?")
- Get prices and availability
- Recommend flowers by occasion
- Place order (collects all 6 address fields, occasion, card message, delivery time)
- Apply and validate promo codes
- Check order status ("¿ya salió mi pedido?")
- FAQ ("¿a qué hora entregan?", "¿cuánto tarda?", "¿dónde están ubicados?")

### Elderly-Friendly WhatsApp UX
- AI always presents numbered options when there are choices:
  ```
  ¿Para qué ocasión es?
  1️⃣ Cumpleaños
  2️⃣ Aniversario
  3️⃣ Quinceañera
  4️⃣ Funeral
  5️⃣ Sin ocasión especial
  
  Responde con el número 😊
  ```
- Short messages. One question at a time. Never walls of text.
- Confirmation always read back before finalizing:
  ```
  Déjame confirmar tu pedido:
  
  🌹 2 arreglos de rosas rojas - $360 MXN
  🎂 Ocasión: Cumpleaños
  💌 Tarjeta: "Feliz cumpleaños amor"
  📍 Calle Juárez 45, entre Obregón y Zaragoza
      Casa blanca · frente al OXXO
  🕒 Hoy a las 3:00pm
  💰 Total: $360 MXN
  
  ¿Confirmo? Responde SÍ o NO
  ```

### Address Collection (6 Required Fields)
AI collects all 6 fields, one at a time, before finalizing order:
1. Calle y número
2. Entre qué calles (2 cross streets)
3. Colonia / fraccionamiento
4. Referencia de lugar (church, OXXO, Coppel, school, pharmacy)
5. Color de la casa / portón
6. Nota adicional (optional — ring bell, gate code, etc.)

Stored as structured JSON. Google Maps link auto-generated from fields 1+2+3 + "Guasave, Sinaloa".

### Escalation Triggers
- Customer types: "operadora", "persona real", "quiero hablar con alguien", "Carmelita", "ayuda"
- GPT-4o returns `escalate: true` (cannot resolve after 2 attempts)
- 3 consecutive failed order attempts

### Human Handoff Flow
```
1. n8n sets customer human_handoff = true
2. Customer receives:
   "Ahora te conecto con Carmelita. ¡Un momento! 🌸"
3. Carmelita gets on her personal WhatsApp:
   "⚠️ CLIENTE NECESITA AYUDA
    Cliente: [name] · [phone]
    Último mensaje: [message]
    Responde LISTO cuando termines."
4. All messages from customer forwarded to Carmelita
5. Carmelita replies LISTO → human_handoff = false → bot resumes
```

### Promo Code Validation
```
Customer provides code
        │
        ▼
n8n queries promo_codes:
  active = true
  AND valid_from <= today <= valid_until
  AND (max_uses IS NULL OR times_used < max_uses)
  AND code = [customer input] (case-insensitive)
        │
  valid─┤─invalid
        │        │
        ▼        ▼
  Apply       "Lo siento, ese código
  discount    no es válido o ya expiró."
  increment
  times_used
```

---

## 6. Voice Call Agent

**Stack:** Twilio virtual number (+52 668-XXX-XXXX) → Vapi.ai → ElevenLabs voice clone → GPT-4o → n8n tool webhooks

### Voice Clone Setup (One-Time)
- Carmelita records ~30 minutes of natural Spanish speech (can be done in 5–10 min chunks)
- Audio requirements: clear, no background noise or music, WAV/MP3 at 44.1kHz
- Uploaded to ElevenLabs Professional Voice Clone → returns `voice_id`
- `voice_id` configured in Vapi.ai → all calls sound like Carmelita

### Call Flow
```
Customer dials Twilio number
        │
        ▼
Twilio routes to Vapi.ai
        │
        ▼
Vapi greets in Carmelita's voice:
"Hola, bienvenido a Jenny Florería.
 Soy Jenny, ¿en qué te puedo ayudar hoy?"
        │
Customer speaks → Vapi STT → text
        │
        ▼
GPT-4o (same Jenny persona, same inventory data)
with Vapi "tools" (n8n webhooks):
  • check_inventory   → n8n → Supabase query
  • create_order      → n8n → creates order record
  • validate_promo    → n8n → promo_codes check
  • get_order_status  → n8n → order lookup
  • transfer_to_human → Vapi transfers call to Carmelita's cell
        │
GPT-4o response → Vapi TTS → ElevenLabs voice
Customer hears Carmelita's voice
        │
   loop until resolved
```

### Elderly-Friendly Voice UX
- Slow, clear speech pace (configured in ElevenLabs voice settings)
- AI repeats options if customer is silent for 5+ seconds
- Numbered options spoken aloud: "Para rosas, diga UNO. Para girasoles, diga DOS."
- Always confirms order back fully before creating
- Patient tone — never rushes

### Phone Order Completion
```
AI collects during call:
  • flowers + quantity
  • occasion and card message
  • all 6 address fields (street, cross streets, colonia,
    landmark, house color, notes)
  • scheduled delivery time or ASAP
  • promo code if they have one
  • their WhatsApp number (to send payment link)
        │
        ▼
Full order summary read back:
"Entonces voy a confirmar su pedido:
 Dos arreglos de rosas rojas, para cumpleaños.
 La tarjeta dirá: Feliz cumpleaños amor.
 Entrega en Calle Juárez cuarenta y cinco,
 entre Obregón y Zaragoza, frente al OXXO,
 casa blanca con portón café.
 Para hoy a las tres de la tarde.
 El total es trescientos sesenta pesos.
 ¿Confirmo su pedido? Diga SÍ o NO."
        │
Customer confirms
        │
        ▼
Vapi calls create_order webhook → n8n → Supabase
n8n sends MercadoPago link to customer's WhatsApp
AI: "Listo, su pedido está registrado.
     Le mandamos el link de pago a su WhatsApp
     ahora mismo. ¡Gracias por llamar a Jenny Florería!"
```

### Call Transfer to Carmelita
Triggered when:
- Customer says: "operadora", "persona real", "quiero hablar con usted", "Carmelita"
- GPT-4o cannot resolve after 2 attempts

```
Vapi says (in Carmelita's voice):
"Claro, con mucho gusto le comunico con Carmelita.
 Un momento por favor."
        │
        ▼
Vapi native transfer → Carmelita's personal cell number
(seamless, no re-dial required)
```

### Cost Estimate
```
~$0.05/min Vapi + ~$0.03/min ElevenLabs + ~$0.01/min GPT-4o
= ~$0.09/min total

30 min/day  → ~$81/month
50 min/day  → ~$135/month
```

---

## 7. Payment (MercadoPago)

**Why MercadoPago over Stripe:**
- OXXO cash payments (customers pay at any OXXO — huge for unbanked customers)
- SPEI bank transfer (standard Mexican banking)
- MercadoPago wallet (familiar brand, trusted by Mexican consumers)
- Better fraud protection tuned for LatAm
- Fees: ~3.49% + IVA for cards, flat fee (~$13 MXN) for OXXO

### Payment Flow
```
Order confirmed (WhatsApp or voice)
        │
        ▼
n8n calls MercadoPago Preferences API
Creates payment preference with:
  • order total
  • order reference ID
  • success/failure/pending webhooks
        │
        ▼
MercadoPago returns payment URL
n8n sends to customer via WhatsApp:
"💳 Paga tu pedido aquí:
 mercadopago.com/...
 
 Puedes pagar con:
 💳 Tarjeta · 🏪 OXXO · 🏦 Transferencia
 
 El link es válido por 24 horas."
        │
Customer pays
        │
        ▼
MercadoPago webhook → n8n
n8n updates order: status = "paid", mp_payment_id saved
        │
        ▼
Driver dispatch triggered (see Section 8)
```

### Abandoned Payment Handling
If order remains `confirmed` (unpaid) for 2 hours:
- n8n sends reminder via WhatsApp: "¿Sigues interesado en tu pedido? El link de pago sigue disponible."
- If unpaid after 24 hours: order marked `cancelled`, stock released back.

---

## 8. Delivery Dispatch

**No driver app needed.** Entire system runs through WhatsApp. Drivers already use it.

### Dispatch Timing
```
Order status → PAID
        │
        ▼
Check scheduled_delivery_at
        │
  null (ASAP)          future timestamp
        │                     │
        ▼                     ▼
  Dispatch now         n8n schedules dispatch
                       45 min before requested time
```

### Driver Selection & Notification
```
n8n queries drivers table:
  active = true AND current_order_id IS NULL
(only ping available drivers)
        │
        ▼
WhatsApp message to all available drivers:

"🌸 NUEVA ENTREGA - Jenny Florería
 Pedido #042
 ─────────────────────────
 Flores: 2 arreglos de rosas rojas
 Para: Cumpleaños de Ana
 ─────────────────────────
 📍 Calle Juárez 45
    Entre Obregón y Zaragoza
    Col. Centro
    Frente al OXXO · Casa blanca
 🗺️ maps.google.com/?q=...
 ─────────────────────────
 🕒 Hoy a las 3:00pm
 ─────────────────────────
 Responde ACEPTO para tomar el pedido
 Responde RECHAZO si no puedes"
        │
        ▼
First driver replies ACEPTO
        │
    ────┴──────────────────────────
    │                              │
    ▼                              ▼
Assigned driver:              Other drivers:
"✅ Pedido asignado!           "Este pedido ya fue tomado.
 Recoge en la tienda.          ¡Gracias!"
 [Full order details]
 [Google Maps link]"
        │
        ▼
Customer:                      Carmelita:
"🛵 Tu pedido está             "🚴 Marco tomó pedido
 en camino!                     #042 · Ana García
 Marco lo entregará             Calle Juárez 45"
 en ~20-30 min."
```

### Delivery Confirmation
```
Driver sends "ENTREGADO" (+ optional photo)
        │
        ▼
n8n updates Supabase:
  order.status = "delivered"
  order.delivery_photo_url = [if photo]
  driver.current_order_id = null  ← available again
        │
    ────┴──────────────────
    │                      │
    ▼                      ▼
Customer:             Carmelita:
"🌸 ¡Tu pedido fue    "✅ Pedido #042
 entregado!            entregado · Marco
 Gracias por           Ana García · Juárez 45"
 elegir Jenny
 Florería! 🌺"
```

### Edge Cases
| Situation | Action |
|---|---|
| All drivers busy | Alert Carmelita immediately: "⚠️ Todos los repartidores están ocupados. Pedido #042 necesita asignación manual." |
| No acceptance in 15 min | Alert Carmelita: "⚠️ Sin respuesta de repartidores · Pedido #042" |
| All 3 reject | Alert Carmelita: "Ningún repartidor aceptó el pedido #042" |
| Driver sends unrecognized text | n8n ignores — only reacts to ACEPTO / RECHAZO / ENTREGADO |

---

## 9. Owner Dashboard

**Tech:** Next.js → Vercel (free tier). Supabase JS client. Supabase Realtime for live updates.  
**Access:** Chrome on Samsung. Bookmarked to home screen (PWA manifest). Looks like an app.

### Elderly-Friendly UI Requirements (Hard Constraints)
- Minimum font size: 18px body, 24px headings, 32px status labels
- High contrast: dark text on light backgrounds, color-coded status badges with both color AND text label
- Large tap targets: minimum 48×48px for all buttons
- Maximum 3 taps to reach any action
- No jargon — "Nuevo Pedido" not "Pending Order", "En Camino" not "In Transit"
- Confirmation dialogs before any destructive action ("¿Seguro que quieres desactivar estas rosas?")
- Simple bottom navigation: 📦 Pedidos · 🌸 Inventario · 🚴 Repartidores · 🎟️ Códigos · 💰 Ventas

### Status Color System
| Status | Color | Label |
|---|---|---|
| Pendiente | 🟡 Yellow | PENDIENTE |
| Confirmado | 🔵 Blue | CONFIRMADO |
| Pagado | 🟢 Green | PAGADO |
| Repartidor Asignado | 🟣 Purple | ASIGNADO |
| En Camino | 🟠 Orange | EN CAMINO |
| Entregado | ✅ Dark Green | ENTREGADO |
| Cancelado | 🔴 Red | CANCELADO |

### Panel 1 — Live Orders (Home Screen)
Real-time order feed, sorted by urgency (soonest delivery first). Each card shows:
- Order number, customer name, status badge (large, color-coded)
- Flower summary, delivery address, scheduled time
- Driver name if assigned
- Tap to expand full detail

### Panel 2 — Inventory Manager

**Quick-Add for New Arrivals ("Llegó mercancía" flow)**
When a new shipment arrives, Carmelita taps [📦 Llegó Mercancía] — a streamlined 4-step flow optimized for speed:
```
Step 1: Photo         → tap to open camera or photo roll
Step 2: Name + type   → text field + category picker (Flores/Arreglos/Ornamentos/
                        Coronas/Centro de Mesa/Bouquet Boda/Decoración Evento)
Step 3: Price + stock → two large number inputs
Step 4: Occasions     → checkbox grid (Cumpleaños/Aniversario/Boda/
                        Quinceañera/Funeral/General)
→ [Guardar] → item live in catalog and AI inventory immediately
```
Entire flow completable in under 60 seconds per item. No technical knowledge required.

**Full Inventory List**
- Items grouped by category with category header badges
- Each card: photo thumbnail, name, price (large), stock count (color-coded)
  - 🟢 Verde: >10 units · 🟡 Amarillo: 5–10 units · 🔴 Rojo: <5 units
- Tap item → edit any field: price, stock, description, occasions, customization options
- [Desactivar] toggle → item hidden from AI and catalog instantly
- [+ Agregar] for single-item manual add (same 4-step flow as above)

**Customization Options Editor**
Per item, Carmelita can define available options customers can choose:
- Sizes (with price per size)
- Color choices
- Add-ons with prices (jarrón, listón, tarjeta impresa, etc.)
These appear in the catalog and are collected by the AI during ordering.

### Panel 3 — Driver Status
- Each driver shown with availability status (large colored dot)
- 🟢 DISPONIBLE / 🔴 EN RUTA
- Shows current assigned order if in route
- Tap driver → edit name, phone, active status

### Panel 4 — Promo Codes
- List of all codes with status, discount, uses remaining, expiry
- [+ Crear Código] — form: code text, type (% or $MXN), value, date range, max uses
- Toggle active/inactive
- Usage counter displayed per code

### Panel 5 — Customers
- Customer list with name, phone, total orders, total spent, last order date
- Tap customer → full order history
- Useful for loyalty tracking and repeat customer recognition

### Panel 6 — Revenue Summary
```
📅 Hoy:          $1,240 MXN  (4 pedidos)
📅 Esta semana:  $6,890 MXN  (23 pedidos)
📅 Este mes:     $28,440 MXN (91 pedidos)
```
- Top 5 selling flowers
- Busiest delivery hours
- No complex charts — plain numbers in large text

---

## 10. Carmelita Notifications (Personal WhatsApp)

Carmelita stays informed via WhatsApp on her personal number. Dashboard is for managing. WhatsApp is for awareness.

| Event | Message |
|---|---|
| New order placed | "🛒 Nuevo pedido #047 · Rosa Ríos · $380 MXN · Esperando pago" |
| Payment received | "💚 Pago recibido pedido #047 · $380 MXN · Buscando repartidor..." |
| Driver assigned | "🚴 Pedro tomó pedido #047 · Rosa Ríos · Col. Lázaro Cárdenas" |
| Order delivered | "✅ Pedido #047 entregado · Pedro · Rosa Ríos" |
| Low stock | "⚠️ Stock bajo: Rosas Rojas quedan 4 unidades" |
| No driver available | "🚨 Sin repartidor disponible · Pedido #047 necesita atención manual" |
| Customer needs help | "👤 Cliente necesita asistencia · [name] · Último mensaje: [message]. Responde LISTO cuando termines." |
| Abandoned payment | "⏰ Pedido #047 sin pago después de 2 hrs · Rosa Ríos" |
| Installment paid | "💚 Abono recibido · Pedido #052 Boda García · $1,500 MXN · Pagado: $3,000 / $8,500 MXN" |
| Installment due in 3 days | "📅 Próximo abono · Pedido #052 Boda García · $1,500 MXN · Vence: Viernes 30 mayo" |
| Installment overdue | "🚨 Abono vencido · Pedido #052 Boda García · $1,500 MXN · Venció hace 2 días" |
| Plan fully paid | "🎉 ¡Pago completo! Pedido #052 Boda García · $8,500 MXN pagados · Evento: 15 junio" |

---

## 11. Public Customer Catalog

**URL:** `jenny-floreria.vercel.app` (or custom domain)
**Purpose:** Customers browse full inventory with photos, prices, customization options, and tap to order via WhatsApp. No account creation needed.

### Who Uses It
- Any customer — via link shared on WhatsApp, Instagram bio, business cards, shop signage
- Especially useful for older customers who want to see photos before deciding
- Pre-fills WhatsApp message so they don't have to type from scratch

### Page Layout (Elderly-Friendly)

```
┌─────────────────────────────────────┐
│  🌸 Jenny Florería - Guasave        │
│  📞 668-XXX-XXXX                    │
├─────────────────────────────────────┤
│  [🌹 Flores] [💐 Arreglos]         │
│  [👑 Coronas] [🎊 Eventos] [Todos] │
│  ← filter tabs, large tap targets  │
├─────────────────────────────────────┤
│  ┌──────────┐  Rosas Rojas         │
│  │  [photo] │  $180 MXN / arreglo  │
│  │          │  🎂 Cumpleaños       │
│  └──────────┘  💒 Aniversario      │
│  [Ver opciones y ordenar →]        │
├─────────────────────────────────────┤
│  ┌──────────┐  Girasoles           │
│  │  [photo] │  $220 MXN            │
│  └──────────┘  [Ver opciones →]   │
└─────────────────────────────────────┘
```

- Minimum 18px text, 48px tap targets
- Large product photos (full width on mobile)
- Filter by occasion: Cumpleaños / Boda / Quinceañera / Aniversario / Funeral / Todos
- Out-of-stock items shown greyed out with "Agotado" badge — not hidden

### Product Detail Page (Per Flower)

```
[Large photo]
Rosas Rojas — $180 MXN base

📝 Hermoso arreglo de rosas rojas frescas,
   ideal para cumpleaños y aniversarios.

Elige tu tamaño:
○ Pequeño  — $150 MXN
● Mediano  — $180 MXN  ← default
○ Grande   — $250 MXN

Elige color:
[Rojo ✓] [Blanco] [Rosa] [Amarillo]

Extras opcionales:
☐ Jarrón de vidrio     +$50
☐ Listón personalizado +$20
☐ Tarjeta impresa      +$15

¿Algún detalle especial?
[campo de texto libre]

Total estimado: $180 MXN

[🛒 Ordenar por WhatsApp]
```

### "Ordenar por WhatsApp" Button
Generates a pre-filled WhatsApp deep link:
```
https://wa.me/526681234567?text=Hola!%20Quiero%20ordenar%3A%0A
%F0%9F%8C%B9%20Rosas%20Rojas%20Medianas%20(rojo)%0A
%20%20+%20Jarr%C3%B3n%20de%20vidrio%0ATotal%20estimado%3A%20%24230%20MXN
```
Customer taps → WhatsApp opens with pre-filled message → AI receives it and continues from there. Customer never types from scratch.

### Catalog Sync
Supabase → catalog page reads directly via Supabase public API (RLS policy: read-only on active flowers). No sync needed — always live.

---

## 12. Order Customization System

Customizations flow through every layer: catalog → WhatsApp AI → voice AI → order record → driver dispatch message → dashboard.

### How AI Collects Customizations (WhatsApp)

When customer selects an item with `customization_options`, AI presents options one at a time:

```
"Perfecto, elegiste Rosas Rojas 🌹

¿Qué tamaño prefieres?
1️⃣ Pequeño — $150 MXN
2️⃣ Mediano — $180 MXN
3️⃣ Grande  — $250 MXN"

→ Customer: "2"

"¿De qué color?
1️⃣ Rojo
2️⃣ Blanco
3️⃣ Rosa
4️⃣ Amarillo"

→ Customer: "3"  (Rosa)

"¿Deseas agregar algo extra?
1️⃣ Jarrón de vidrio    +$50
2️⃣ Listón personalizado +$20
3️⃣ Tarjeta impresa     +$15
4️⃣ Sin extras

(Puedes elegir varios, ej: 1 y 2)"

→ Customer: "2"

"¿Algún detalle especial para el listón
 u otra nota para este arreglo?
(Responde NO si no hay nada especial)"

→ Customer: "listón color azul cielo si es posible"
```

All selections stored in `order_items.customizations` JSON. Customizations appear in:
- Order confirmation message to customer
- Driver dispatch message
- Dashboard order detail view
- Delivery card printed in-shop (future)

### Items Without Customization
If `customization_options` is null for a flower, AI skips customization questions entirely — no friction for simple items.

---

## 13. Event & Wedding Installment Payments

For large orders (weddings, quinceañeras, graduations, corporate events) where the total may be $3,000–$20,000+ MXN, customers can pay in scheduled installments leading up to the event date.

### How It Works — Full Flow

```
Customer (WhatsApp or call):
"Quiero hacer un pedido para una boda,
 el 15 de junio. ¿Puedo pagar poco a poco?"
        │
        ▼
AI detects event/wedding order type
Collects: event date, full order details,
          customer's preferred payment schedule
        │
        ▼
n8n creates in Supabase:
  orders row  (order_type="wedding", event_date=June 15)
  payment_plans row (total=$8,500 MXN)
  payment_installments rows (3-4 installments)
        │
        ▼
AI sends payment plan summary to customer:
"¡Perfecto! Tu plan de pagos para la boda:

 💒 Pedido Boda García — $8,500 MXN total
 📅 Evento: 15 de junio

 Plan de pagos:
 ✅ Abono 1: $2,500 MXN — hoy (reserva)
 ⏳ Abono 2: $2,000 MXN — 1 de junio
 ⏳ Abono 3: $2,000 MXN — 10 de junio
 ⏳ Abono 4: $2,000 MXN — 13 de junio (saldo final)

 Total: $8,500 MXN

 ¿Confirmas este plan? Responde SÍ o NO"
        │
Customer confirms
        │
        ▼
First installment MercadoPago link sent immediately
Remaining installments scheduled in n8n
```

### Installment Reminder Flow (n8n Scheduled)

```
3 days before due date → WhatsApp to customer:
"📅 Recordatorio de abono — Jenny Florería
 Pedido: Boda García (#052)
 Monto: $2,000 MXN
 Fecha límite: 1 de junio
 
 [Link de pago MercadoPago]
 
 ¿Tienes alguna duda? Escríbenos 😊"

1 day before due date → second reminder

Day of due date (if unpaid):
"⏰ Tu abono de $2,000 MXN vence HOY.
 Pedido Boda García. [Link de pago]"

1 day overdue → alert to Carmelita on personal WhatsApp:
"🚨 Abono vencido · Boda García · $2,000 MXN
 Vencía ayer. Considera contactar al cliente."
```

### Installment Payment Confirmation

```
Customer pays installment
        │
        ▼
MercadoPago webhook → n8n
n8n updates:
  payment_installments.status = "paid"
  payment_plans.amount_paid_mxn += amount
  orders.amount_paid_mxn += amount
        │
  All paid?──yes──→ orders.status = "paid"
                    → normal delivery flow begins
     │
    no
     │
     ▼
Customer confirmation:
"💚 ¡Abono recibido! Gracias 🌸
 Pagado: $4,500 / $8,500 MXN
 Próximo abono: $2,000 MXN el 10 de junio"

Carmelita notification:
"💚 Abono recibido · Boda García · $2,000 MXN
 Total pagado: $4,500 / $8,500 MXN"
```

### Dashboard — Event Orders Panel

Separate section in dashboard for event/wedding orders showing:
- Order name, customer, event date
- Progress bar: amount paid / total (e.g. $4,500 / $8,500)
- Upcoming installments with due dates
- [Ver Plan] → full installment timeline with statuses
- [Editar Plan] → Carmelita can adjust amounts/dates if agreed with customer
- Color-coded: 🟢 on track · 🟡 due soon · 🔴 overdue

### Carmelita Creates Payment Plan (Dashboard)

For orders negotiated in person or by call:
```
Dashboard → Pedidos → [+ Evento/Boda]
1. Customer name + WhatsApp number
2. Order details (items, customizations)
3. Event date
4. Total amount
5. Number of installments (2 / 3 / 4 / custom)
   → system auto-distributes amounts and dates
   → Carmelita can adjust each installment manually
6. [Crear Plan] → plan created + first payment link sent to customer WA
```

### Payment Plan Rules
- First installment (deposit/reserva) required immediately to confirm event slot
- Minimum deposit: 25% of total (configurable by Carmelita)
- Full payment required at least 2 days before event date
- If plan goes overdue by 7+ days → Carmelita notified, order not automatically cancelled (she decides)

---

## 14. Technology Stack & Monthly Cost


| Service | Purpose | Cost |
|---|---|---|
| Supabase | Database, storage, realtime | Free tier |
| n8n Cloud | Workflow automation (WhatsApp, orders, dispatch, notifications) | ~$20/mo |
| Meta WhatsApp Cloud API | WhatsApp send/receive | Free (first 1,000 conv/mo) |
| OpenAI GPT-4o | AI brain for WhatsApp + voice tools | ~$20-30/mo |
| Vapi.ai | Voice call engine (STT + LLM + TTS pipeline) | ~$60-100/mo (volume dependent) |
| ElevenLabs Creator | Carmelita's voice clone + TTS | $22/mo |
| Twilio | Mexican virtual phone number (+52) | ~$5/mo |
| MercadoPago | Payments (OXXO, card, SPEI, wallet) | % per transaction only |
| Vercel | Dashboard hosting | Free |
| **Total** | | **~$127–177/mo ✅** |

MercadoPago fees: ~3.49% + IVA for cards, ~$13 MXN flat for OXXO.

---

## 15. Voice Clone Recording Guide (For Carmelita)

**Before building the voice agent, Carmelita must complete this one-time recording session.**

- Record 30 minutes total (can split into 5-10 minute sessions)
- Use her phone in a quiet room — no music, fans, or street noise
- Speak naturally as if talking to a customer — warm, helpful tone
- Sample sentences to read (provide a script of ~200 varied sentences in Mexican Spanish)
- Save as WAV or MP3
- Upload to ElevenLabs Professional Voice Clone portal
- Process takes ~15 minutes → `voice_id` provided
- Plug `voice_id` into Vapi.ai configuration

---

## 16. What Carmelita Does Daily

After system is live, her daily tasks are minimal:

| Task | How |
|---|---|
| Check new orders | Glance at WhatsApp notifications or open dashboard |
| Add new arrival flowers/ornaments | Dashboard → Inventario → [📦 Llegó Mercancía] → 4 steps, ~60 sec per item |
| Update existing flower stock | Dashboard → Inventario → tap flower → tap stock number → update |
| Handle escalated customer | Reply to WhatsApp forwarded message → type LISTO when done |
| Create a promo code | Dashboard → Códigos → [+ Crear Código] |
| Create event/wedding payment plan | Dashboard → Pedidos → [+ Evento/Boda] → fill form → system sends plan to customer |
| Check installment status | Dashboard → Eventos → see progress bar per order |
| View today's revenue | Dashboard → Ventas |

She does NOT need to:
- Read or reply to customer messages (AI handles)
- Answer ordering calls (AI handles)
- Dispatch drivers (automated)
- Send payment links (automated)
- Confirm deliveries (automated)

---

## 17. Future Expansion (Not In Scope Now)

- Loyalty program (points per order)
- Recurring subscription orders ("flores cada semana")
- Instagram/Facebook DM automation
- Multi-location support
- Printed receipt / order ticket for shop preparation
- SMS fallback for customers without WhatsApp
