# Rental Voice IO

A voice-enabled equipment rental management system that automates the full customer-to-vendor-to-return lifecycle using AI voice agents (VAPI), workflow automation (n8n), and CRM integration (HubSpot).

---

## Folder Structure

```
/rental-voice-io
│
├── /database
│   └── SchemaRental.sql
│
├── /agents
│   ├── /inbound-coordinator
│   │   ├── inbound-agent.json         ← n8n Workflow
│   │   ├── system-prompt.md           ← VAPI System Prompt
│   │   └── tools/
│   │       ├── get_inventory_details.json
│   │       └── create_rental_booking.json
│   │
│   ├── /vendor-outbound
│   │   ├── vendor-outbound-dispatcher.json
│   │   ├── system-prompt.md
│   │   └── tools/
│   │       └── submit_dispatch_to_n8n.json
│   │
│   ├── /client-outbound
│   │   ├── client-outbound-agent.json
│   │   ├── system-prompt.md
│   │   └── tools/
│   │       └── submitCallResult.json
│   │
│   └── /rental-success-manager
│       ├── rental-success-manager.json
│       ├── system-prompt.md
│       └── tools/
│           └── submit_rental_result.json
│
└── README.md
```

---

## System Overview

```
CUSTOMER INBOUND CALL
        │
        ▼
┌───────────────────┐
│ Inbound Coordinator│  ← webhook /vapi-test
└────────┬──────────┘
         │
   ┌─────┴──────┐
   ▼            ▼
AVAILABLE    NOT AVAILABLE
   │            │
   ▼            ▼
Create       Query HubSpot vendors
Booking      Insert partners_dispatch
   │            │
   ▼            ▼
HubSpot +    Email vendors + Slack
Slack alert       │
                  ▼
      ┌──────────────────────┐
      │ Vendor Outbound       │  ← scheduled every 1 min
      │ Dispatcher            │
      └──────────┬───────────┘
                 │
       ┌─────────┴─────────┐
       ▼                   ▼
   AVAILABLE           UNAVAILABLE
       │                   │
   Calculate markup    Next vendor
   Update dispatch          │
   Cancel lower-priority ◄──┘
       │
       ▼
┌─────────────────────┐
│ Client Outbound Agent│  ← scheduled every 1 min
└──────────┬──────────┘
           │
   ┌───────┴──────────┐
   ▼        ▼         ▼
ACCEPTED  REJECTED  NO_ANSWER
   │         │         │
Create    Cancel    Retry
Rental    dispatch  (max 2x)
HubSpot   Email vendor
Slack     release
           │
           ▼
┌────────────────────────┐
│ Rental Success Manager  │  ← scheduled every 1 min
└───────────┬────────────┘
            │
   ┌────────┴────────┐
   ▼                 ▼
OWNED RENTAL    PARTNER RENTAL
(can extend)    (no extensions)
   │                 │
   ├─CONFIRM    ─────┤
   │  Mark closed    │ CONFIRM → closed + pickup
   │                 │
   ├─EXTEND     ─────┤
   │  Add days       │ ACTION_REQUIRED → Slack alert
   │  Update DB      │
   └─ACTION_REQ──────┘
      Flag manual
```

---

## Agents

### 1. Inbound Coordinator

**File:** [agents/inbound-coordinator/inbound-agent.json](agents/inbound-coordinator/inbound-agent.json)
**Prompt:** [agents/inbound-coordinator/system-prompt.md](agents/inbound-coordinator/system-prompt.md)
**Trigger:** Webhook — customer calls in

**Role:** Handles inbound customer equipment requests. Checks inventory availability, collects and normalizes customer data (name, email, phone, address, dates), and either creates a booking directly or initiates a partner vendor search.

**Key logic:**
- Normalizes spoken data: `"at"` → `@`, `"dot"` → `.`, spoken digits → numeric
- Extracts `equipment_id` (UUID) from inventory response — never fabricates IDs
- Collects mandatory fields before proceeding to booking
- Creates HubSpot contact and deal on confirmation

**Tools:**
| Tool | Purpose |
|------|---------|
| `get_inventory_details` | Check equipment availability and retrieve equipment_id, model_name, daily_rate |
| `create_rental_booking` | Submit a new rental booking with full customer and equipment details |

---

### 2. Vendor Outbound Dispatcher

**File:** [agents/vendor-outbound/vendor-outbound-dispatcher.json](agents/vendor-outbound/vendor-outbound-dispatcher.json)
**Prompt:** [agents/vendor-outbound/system-prompt.md](agents/vendor-outbound/system-prompt.md)
**Trigger:** Scheduled — every 1 minute

**Role:** Calls partner vendors to check equipment availability and collect daily rates. Processes vendors in priority order. When a vendor confirms availability, lower-priority vendors for the same customer/equipment are automatically cancelled.

**Key logic:**
- Queries `partners_dispatch` for `pending_vendor_contact` records
- Calls vendors one at a time, collects rates per equipment sequentially
- Calculates `markup = vendor_base_rate × 20%` → `client_final_rate = base + markup`
- Cancels competing lower-priority dispatches when one vendor accepts

**Tools:**
| Tool | Purpose |
|------|---------|
| `submit_dispatch_to_n8n` | Submit vendor availability, rates, and dispatch status back to n8n |

---

### 3. Client Outbound Agent

**File:** [agents/client-outbound/client-outbound-agent.json](agents/client-outbound/client-outbound-agent.json)
**Prompt:** [agents/client-outbound/system-prompt.md](agents/client-outbound/system-prompt.md)
**Trigger:** Scheduled — every 1 minute

**Role:** Calls customers to present available equipment from the partner vendor network. Presents pricing and dates, confirms customer details, and submits the result.

**Key logic:**
- Queries dispatches where `dispatch_status = available` and `client_call_attempts < 2`
- On **accepted**: creates client record, rental records, HubSpot deal, Slack alert
- On **rejected**: cancels dispatch, emails vendor a release notification
- On **no_answer**: retries up to 2 attempts total
- Strips apostrophes from company names before tool submission

**Tools:**
| Tool | Purpose |
|------|---------|
| `submitCallResult` | Submit call outcome (accepted/rejected/no_answer) with all booking and pricing details |

---

### 4. Rental Success Manager

**File:** [agents/rental-success-manager/rental-success-manager.json](agents/rental-success-manager/rental-success-manager.json)
**Prompt:** [agents/rental-success-manager/system-prompt.md](agents/rental-success-manager/system-prompt.md)
**Trigger:** Scheduled — every 1 minute

**Role:** Calls customers with active/overdue rentals due within 3 hours or already overdue. Handles equipment return confirmation or extension requests.

**Key logic:**
- Groups rentals by customer phone (one call per customer, no mixed OWNED/PARTNER calls)
- **OWNED rentals**: can offer extensions — calculates `new_date = original_return_time + extension_days` (never adds days to today's date)
- **PARTNER rentals**: no extensions allowed; must return today
- Tracks `return_call_attempts` (max 3)
- Response values: `confirm`, `extend`, `action_required`, `no_answer`

**Tools:**
| Tool | Purpose |
|------|---------|
| `submit_rental_result` | Submit return call outcome, extension decisions, and updated return times |

---

## Database

**File:** [database/SchemaRental.sql](database/SchemaRental.sql)
**Engine:** PostgreSQL

### Table: `partners_dispatch`

Tracks vendor outreach and availability for partner-sourced equipment requests.

| Column | Type | Description |
|--------|------|-------------|
| `dispatch_id` | UUID PK | Unique dispatch record |
| `rental_id` | FK | Linked rental |
| `client_id` | FK | Linked client |
| `equipment_id` | FK | Requested equipment |
| `crm_vendor_id` | string | HubSpot vendor ID |
| `vendor_name/phone/email` | string | Vendor contact info |
| `vendor_base_rate` | decimal | Vendor's quoted daily rate |
| `markup_amount` | decimal | 20% markup on base rate |
| `client_final_rate` | decimal | Rate charged to customer |
| `dispatch_status` | enum | `searching`, `available`, `unavailable` |
| `customer_name/email/phone` | string | Customer contact info |
| `equipment_requested` | string | Equipment description |
| `city` | string | Delivery city |
| `call_attempts` | int | Vendor call count |
| `call_result` | string | Vendor call outcome |
| `last_call_at` | timestamp | Last vendor call time |
| `start_date/end_date` | date | Rental period |
| `client_call_attempts` | int | Client outbound call count |
| `client_call_status` | enum | Client call state |
| `vendor_priority` | int | Lower = higher priority |

### Table: `rentals`

Tracks the full lifecycle of equipment rentals.

| Column | Type | Description |
|--------|------|-------------|
| `rental_id` | UUID PK | Unique rental record |
| `client_id` | FK | Renting customer |
| `equipment_id` | FK | Rented equipment |
| `start_time` | timestamp | Rental start |
| `expected_return_time` | timestamp | Originally agreed return |
| `actual_return_time` | timestamp | Actual return (null if open) |
| `status` | enum | `active`, `overdue`, `closed`, `extended` |
| `total_price` | decimal | Total charged |
| `hubspot_deal_id` | string | Linked HubSpot deal |
| `return_call_attempts` | int | Recovery call count (max 3) |
| `return_last_called_at` | timestamp | Last recovery call |
| `extended_return_time` | timestamp | New return after extension |
| `extension_count` | int | Number of extensions granted |
| `total_fine_amount` | decimal | Accumulated late fees |
| `return_call_status` | enum | Current recovery status |
| `extended_total_price` | decimal | Updated price after extension |

---

## Tools (VAPI Function Calling)

All tools make `POST` requests to n8n webhooks. Replace `YOUR_N8N_DOMAIN` with your actual n8n instance URL.

### `get_inventory_details`
- **Endpoint:** `POST https://YOUR_N8N_DOMAIN/webhook/vapi-test`
- **Agent:** Inbound Coordinator
- **Input:** city, email, address, full_name, start_time, postal_code, company_name, phone_number, state_region, equipment_requested, expected_return_time
- **Output:** equipment_id (UUID), model_name, daily_rate, availability status

### `create_rental_booking`
- **Endpoint:** `POST https://YOUR_N8N_DOMAIN/webhook/vapi-booking`
- **Agent:** Inbound Coordinator
- **Input:** city, email, address, full_name, daily_rate, model_name, start_time, postal_code, total_price, company_name, equipment_id, phone_number, state_region, expected_return_time

### `submit_dispatch_to_n8n`
- **Endpoint:** `POST https://YOUR_N8N_DOMAIN/webhook/vapi-callback-vendor`
- **Agent:** Vendor Outbound Dispatcher
- **Input:** city, end_date, start_date, vendor details, equipment_list, dispatch_status (Available/Unavailable/no_answer), vendor_base_rate (comma-separated), vendor_priority

### `submitCallResult`
- **Endpoint:** `POST https://YOUR_N8N_DOMAIN/webhook/vapi-result`
- **Agent:** Client Outbound Agent
- **Input:** call_result (accepted/rejected/no_answer), all_dispatch_ids, customer details, company_name, address, equipment_list, client_final_rate, vendor details, vendor_base_rate, markup_amount, crm_vendor_id, start_time, expected_return_time

### `submit_rental_result`
- **Endpoint:** `POST https://YOUR_N8N_DOMAIN/webhook/rental-manager`
- **Agent:** Rental Success Manager
- **Input:** rental_ids, equipment_ids, client_name, client_phone, company_name, items_list, call_type (OWNED/PARTNER), daily_rates, late_fees, return_times, response (confirm/extend/action_required/no_answer), new_return_time

---

## Integrations

| Service | Purpose |
|---------|---------|
| **VAPI** | AI voice agent platform — hosts all 4 agents, handles calls |
| **n8n** | Workflow automation — orchestrates all scheduling, DB ops, and routing |
| **PostgreSQL** | Primary database — rentals, dispatches, clients, equipment |
| **HubSpot** | CRM — vendor lookup, contact creation, deal tracking |
| **Slack** | Internal notifications — confirmations, alerts, action-required flags |
| **Gmail** | Transactional email — vendor release notices, customer confirmations |

---

## Pricing Logic

```
vendor_base_rate       = vendor's quoted daily rate
markup_amount          = vendor_base_rate × 0.20
client_final_rate      = vendor_base_rate + markup_amount
total_price            = client_final_rate × rental_days
```

---

## Configuration Checklist

Before deploying, replace all placeholder values in workflow JSON files:

| Placeholder | Replace With |
|-------------|-------------|
| `YOUR_POSTGRES_CREDENTIAL_ID` | PostgreSQL credential ID in n8n |
| `YOUR_VAPI_ASSISTANT_ID` | VAPI assistant ID for each agent |
| `YOUR_VAPI_PHONE_NUMBER_ID` | VAPI outbound phone number ID |
| `YOUR_HUBSPOT_APP_TOKEN` | HubSpot private app token |
| `YOUR_SLACK_CHANNEL_ID` | Target Slack channel ID |
| `YOUR_SLACK_CREDENTIAL_ID` | Slack OAuth credential ID in n8n |
| `YOUR_GMAIL_OAUTH_CREDENTIAL_ID` | Gmail OAuth credential ID in n8n |
| `YOUR_N8N_DOMAIN` | Your n8n instance domain (in tool JSON files) |
| `YOUR_WEBHOOK_ID_HERE` | Webhook node IDs in each workflow |

---

## Setup Order

1. Deploy PostgreSQL schema (`database/SchemaRental.sql`)
2. Configure all credentials in n8n
3. Import all 4 workflow JSON files into n8n
4. Create 4 VAPI assistants — one per agent
5. Attach the correct `system-prompt.md` to each VAPI assistant
6. Register each tool JSON in its corresponding VAPI assistant
7. Update `YOUR_N8N_DOMAIN` in all tool JSON files with your actual webhook URLs
8. Activate all n8n workflows
