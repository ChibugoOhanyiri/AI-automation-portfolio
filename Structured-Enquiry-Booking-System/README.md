# Instant Enquiry & Booking System — Telegram Bot (n8n)

<img width="1237" height="648" alt="Screenshot 2026-02-19 162403" src="https://github.com/user-attachments/assets/d5b4638a-12b9-4558-b8c9-467bd4a57057" />


An automated enquiry and booking bot built for a salon business on Telegram. Handles the most common daily customer enquiries through structured conversation menus, routes responses based on conversation stage, captures lead details, and notifies the salon owner.

---

## What It Does

Most salon owners lose time answering the same questions over and over on WhatsApp or Telegram — *"Where are you located?" "How much is a hair fix?" "Can I book for Saturday?"*

This workflow intercepts those messages with a structured menu system. Customers pick from predefined options instead of typing freely, which keeps conversations on-script, ensures the owner gets clean and actionable data, and delivers instant responses 24/7.

---

## Features

- **New vs. returning customer detection** — checks Google Sheets by Chat ID on every message to know exactly where the conversation is
- **Structured menu system** — presents 3 top-level options (Book, Ask a Question, Send a Request) with sub-menus for the Ask a Question flow (location, hours, services, pricing)
- **Conversation stage routing** — routes each incoming message differently depending on whether the status is `Options Sent`, `Asking Question`, or `Collecting Details`
- **Contextual responses** — delivers the right answer based on the exact option the customer selects, using a Switch node
- **Lead capture** — collects customer name, email, and message when they submit a booking/request
- **Owner notification** — sends a Telegram message directly to the salon owner when a new lead submits their details
- **CRM tracking** — logs and updates Conversation Status and Lead Status in Google Sheets at every stage of the flow
- **No free-text chaos** — customers navigate menus, not open chat, so the owner never receives vague or out-of-scope messages

---

## Workflow Architecture

The workflow is split into three logical sections:

### 1. Enquiry Intake
```
Telegram Trigger → Check database (Google Sheets)
  → Does Chat ID exist?
      ├── YES → Check conversation status → Route accordingly
      └── NO  → Send welcome message with options → Log to Sheets
```

**New customers** receive a greeting from *Bubu* (the bot persona) and are presented with three options:
1. Book an appointment
2. Ask a question
3. Send a request

### 2. Routes Based on Chosen Option
```
Preserve message text → Switch node
  ├── Option 1 (Book) → Send booking message → Update CRM → Lead: Pending Booking
  ├── Option 2 (Ask)  → Send question sub-menu (4 options) → Update CRM status
  └── Option 3 (Send) → Request details prompt → Update CRM status
```

The Switch node reads the customer's reply and fans out to the correct branch.

### 3. Response Based on Conversation Stage
```
Returning message → Check convo status
  ├── "Asking Question" → Preserve response number → Check what option was selected
  │     → Send correct answer → Update CRM
  ├── "Collecting Details" → Send thank you → Split out details
  │     → Update Client details in Sheets → Notify owner via Telegram
  └── Other → No Operation (do nothing)
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** | Workflow automation engine |
| **Telegram Bot API** | Customer-facing chat interface |
| **Google Sheets** | CRM — conversation tracking & lead storage |
| **Switch node** | Option-based response routing |
| **IF nodes** | Conversation stage detection |

---

## 📋 Google Sheets CRM Structure

The workflow reads and writes the following columns:

| Column | Description |
|---|---|
| `Chat ID` | Unique Telegram chat identifier — used as the primary key |
| `Conversation Status` | Tracks stage: `Options Sent`, `Asking Question`, `Collecting Details`, `Done` |
| `Lead Status` | Tracks lead stage: `New`, `Pending Booking` |
| `Client Name` | Captured when customer submits a request |
| `Client Email` | Captured when customer submits a request |
| `Client Message` | The customer's enquiry or request text |

---

## 🗺️ Conversation Flow

```
Customer messages bot
        │
        ▼
Is this a new customer?
   ├── YES → "Hello! I'm Bubu 👋 How can I help?"
   │           1. Book an appointment
   │           2. Ask a question
   │           3. Send a request
   │
   └── NO  → What stage is the conversation in?
               ├── Options Sent  → Read their choice → Switch to branch
               ├── Asking Question → Read their number → Send answer
               └── Collecting Details → Save details → Notify owner
```

---

## Setup Instructions

### Prerequisites
- An active n8n instance (cloud or self-hosted)
- A Telegram Bot token 
- A Google Sheets spreadsheet with the columns listed above
- Google Sheets OAuth2 credentials configured in n8n

### Steps

1. **Import the workflow** — upload `Instant_enquiry_booking_system.json` into your n8n instance via *Import from file*
2. **Connect your Telegram Bot** — replace the Telegram credentials with your own bot token
3. **Connect Google Sheets** — update the `documentId` and `sheetName` values in all Google Sheets nodes to point to your spreadsheet
4. **Update response text** — edit the Telegram message nodes to reflect your salon's actual name, location, hours, services, and pricing
5. **Set owner Chat ID** — update the *Send a message to owner* node with the salon owner's Telegram Chat ID
6. **Activate the workflow** — toggle the workflow to active and test by messaging your bot

---

## Design Decisions

**Why structured menus instead of free-text AI?**
Free-text bots require LLM integration, careful prompt engineering, and can still go off-script. For a salon owner whose enquiries are predictable and repetitive, a structured menu is faster, cheaper, more reliable, and produces cleaner lead data. This is an enquiry system, not a support bot.

**Why Google Sheets as the CRM?**
Zero setup friction for a small business owner. The salon owner can see every lead and conversation status in a spreadsheet they already understand — no new tool to learn.

**Why track conversation status?**
n8n workflows are stateless by default. Every incoming Telegram message triggers the workflow fresh. Storing conversation stage in Sheets gives the workflow memory across turns without needing a database.
