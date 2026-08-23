# Hotel Room Booking — AI Lead Management & Follow-up System

An n8n automation that receives WhatsApp inquiries from potential hotel guests, uses AI to qualify how serious each lead is, and automatically responds with the right message — an instant price/availability offer for ready-to-book guests, or a timed follow-up for guests who need more nurturing. Every lead is logged and updated in Google Sheets so nothing falls through the cracks.

**Problem it solves:** Out of every 100 WhatsApp room inquiries, only ~15 turn into bookings — mostly because replies are slow or interested guests never get a follow-up. This workflow makes sure every single inquiry gets an immediate, intent-aware response and a scheduled follow-up if it isn't a "hot" lead.

---

## How It Works (Flow Overview)

```
WhatsApp message in
      │
      ▼
Extract Lead Info  ──►  Save Customer Info (Google Sheets, Status: New)
      │
      ▼
AI Lead Qualification (OpenAI gpt-4o-mini + Structured Output Parser)
      │
      ▼
Router (Hot / Warm / Cold)
      │
   ┌──┴────────────────┬─────────────────────┐
   ▼                    ▼                     ▼
 HOT                  WARM                  COLD
Check Availability    Wait 3 hours          Wait 1 day
+ Price (PMS API)          │                     │
   │                       ▼                     ▼
   ▼                Send Warm Follow-up   Send Cold Nurture Message
Send Offer
(price + hold/book)
   │                       │                     │
   └───────────┬───────────┴─────────────────────┘
               ▼
      Update CRM Status (Google Sheets)
```

## Step-by-Step Node Breakdown

| # | Node | Type | What it does |
|---|------|------|---------------|
| 1 | **WhatsApp Trigger** | Webhook | Receives incoming WhatsApp messages at the `whatsapp-lead` path. This is where Meta's WhatsApp Cloud API sends the raw payload. |
| 2 | **Extract Lead Info** | Set | Pulls `customer_phone`, `customer_message`, `customer_name`, and `received_at` out of the nested WhatsApp webhook JSON so the rest of the workflow can use clean fields. |
| 3 | **Save Customer Info** | Google Sheets (Append) | Logs the new lead into the **Leads** sheet immediately, with `Status = New`, so every inquiry is captured even before AI processing runs. |
| 4 | **AI Lead Qualification** | LangChain Agent | Sends the customer's message to the AI with a system prompt that classifies the lead. |
| 5 | **OpenAI Model** | LM Chat (OpenAI) | Supplies `gpt-4o-mini` as the language model powering node 4. |
| 6 | **Structured Output Parser** | Output Parser | Forces the AI's reply into a strict JSON shape: `category`, `score`, `intent_summary`, `check_in_date`, `check_out_date`, `room_type`, `guests`. |
| 7 | **Router (Hot / Warm / Cold)** | Switch | Branches the workflow into 3 paths based on `output.category`. |
| 8 | **Check Room Availability + Price** | HTTP Request | *(Hot path only)* Calls your Property Management System (PMS) API with the guest's dates and room type to get live availability and price. |
| 9 | **Send Offer (Booking Opportunity)** | WhatsApp | *(Hot path)* Sends the guest the available room, price, and asks if they want it held or want the booking link — while it's still top of mind. |
| 10 | **Wait – Warm Follow-up Delay** | Wait | *(Warm path)* Pauses 3 hours before re-engaging a warm lead. |
| 11 | **Send Warm Follow-up** | WhatsApp | *(Warm path)* Nudges the guest to share dates so a rate can be checked. |
| 12 | **Wait – Cold Follow-up Delay** | Wait | *(Cold path)* Pauses 1 day before re-engaging a cold/browsing lead. |
| 13 | **Send Cold Nurture Message** | WhatsApp | *(Cold path)* Sends a soft nudge with a "special rate this week" hook to re-open the conversation. |
| 14 | **Update CRM Status** | Google Sheets (Update) | All three paths converge here. Matches the lead by `Phone` and updates `Status`, `Score`, `RoomType`, `CheckIn`, `CheckOut` in the Leads sheet. |

## Lead Classification Logic (AI Prompt Rules)

The AI qualifies every inbound message into one of three categories:

- **Hot** — Ready to book now: gave dates, or asked directly for price/availability.
- **Warm** — Interested but missing details: no firm dates yet, comparing options.
- **Cold** — Vague inquiry: just browsing, no clear booking intent.

It also extracts (when available): a 0–100 lead score, a one-line intent summary, check-in/check-out dates, room type, and guest count.

## Google Sheet Structure ("Leads" tab)

| Column | Set by | Description |
|---|---|---|
| Name | Save Customer Info | Guest's WhatsApp profile name |
| Phone | Save Customer Info | Guest's WhatsApp number (used as the match key for updates) |
| Message | Save Customer Info | Original inquiry text |
| ReceivedAt | Save Customer Info | Timestamp of the inbound message |
| Status | Save Customer Info → Update CRM Status | Starts as `New`, updated to `hot` / `warm` / `cold` after AI qualification |
| Score | Update CRM Status | AI-assigned lead score (0–100) |
| RoomType | Update CRM Status | Room type extracted from the message |
| CheckIn / CheckOut | Update CRM Status | Extracted stay dates |

## Setup Checklist

Before importing this workflow into n8n, replace the following placeholders:

1. **WhatsApp Cloud API**
   - `YOUR_WHATSAPP_PHONE_NUMBER_ID` — your Meta WhatsApp Business phone number ID
   - `YOUR_WHATSAPP_CREDENTIAL_ID` — n8n WhatsApp API credential
   - Point your Meta webhook to the n8n webhook URL for the `whatsapp-lead` path

2. **Google Sheets**
   - `YOUR_GOOGLE_SHEET_ID` — the spreadsheet holding your **Leads** tab (create the columns listed above)
   - `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` — n8n Google Sheets OAuth2 credential

3. **OpenAI**
   - `YOUR_OPENAI_CREDENTIAL_ID` — n8n OpenAI API credential (used with `gpt-4o-mini`)

4. **Hotel PMS API**
   - `https://YOUR_PMS_API/availability` — replace with your actual property management system's availability/pricing endpoint. It must accept `check_in`, `check_out`, and `room_type` as query parameters and return `room_type` and `price` in the response, since the "Send Offer" message references `$json.room_type` and `$json.price` directly.

## Known Gaps / Things to Double-Check Before Going Live

- **PMS response shape isn't fixed yet.** The offer message assumes the PMS API returns `room_type` and `price` fields — confirm this matches your actual API's response, or add a Set node after "Check Room Availability + Price" to map fields correctly.
- **No "no availability" branch.** If the PMS API returns no rooms for the requested dates, the current flow will still try to send an offer message. You may want an IF node after the availability check to handle a "sorry, fully booked" reply.
- **No re-trigger safeguard.** If the same customer messages again while a Wait node is pending, a second instance of the workflow could run in parallel. Consider adding a check against the Leads sheet status before sending follow-ups.
- **Currency is hardcoded** (৳ Taka) in the offer message — fine if this is BDT-only, but flag if you expand to other currencies.

## Why This Design Solves the Original Problem

- **No inquiry is lost** — every message is logged to Sheets *before* AI processing even runs (node 3), so even if qualification fails, the lead exists in the CRM.
- **Hot leads get instant action** — real-time availability + price, sent while the guest is still actively chatting.
- **Warm and cold leads aren't abandoned** — timed follow-ups (3 hours / 1 day) re-engage guests who didn't convert on the first message, which is exactly the "no follow-up" gap causing the original 15% conversion rate.
