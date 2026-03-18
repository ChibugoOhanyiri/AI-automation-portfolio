# Erica | AI Property Assistant for Real Estate

<img width="1828" height="547" alt="image" src="https://github.com/user-attachments/assets/02fa286d-c736-4b86-8610-0a2681db29d3" />


Erica is an intelligent Telegram chatbot built for a real estate company. She acts as a virtual property advisor — qualifying leads, searching listings, booking property inspections, and logging every lead to a CRM automatically. No human intervention required.

---

## What Erica Does

**Lead qualification**
The moment a client messages, Erica collects their name, email, phone number, budget, preferred location, property type, and number of bedrooms conversationally — like a real agent would.

**Semantic property search**
Erica searches a RAG-powered knowledge base built on Supabase Vector Store to find matching listings based on what the client actually describes, not just keywords.

**Property images in chat**
Matching listings are sent directly inside the Telegram conversation with images, no external links.

**Live calendar availability**
Erica checks Google Calendar in real time and offers available inspection slots to the client.

**Inspection booking**
Once the client picks a slot, Erica creates the Google Calendar event and sends a confirmation email with property details and the assigned agent's contacts.

**Voice note support**
Clients can send voice notes instead of typing. Erica transcribes them using Groq Whisper and processes them as normal messages.

**Automatic CRM logging**
Every qualified lead is saved to Google Sheets automatically — name, email, contact number, budget, property preferences. The agent wakes up to an updated CRM.

---

## Architecture

```
Telegram Trigger
      │
      ▼
Does text message exist?
      │
   ┌──┴──┐
  Yes    No (voice note)
   │         │
   │    Download → Convert .oga → Transcribe (Groq Whisper)
   │         │
   └────┬────┘
        ▼
 Standardize text input
        │
        ▼
    AI Agent (GPT-4o-mini)
        │
   ┌────┼────┬──────┬──────┬──────┬──────┐
   ▼    ▼    ▼      ▼      ▼      ▼      ▼
 CRM  KB  Calendar Book  List  Image  Email
      RAG  Check  Insp  Props  Send   Lead
```

**RAG Pipeline (separate workflow)**
```
Google Drive (new file uploaded)
        │
        ▼
  Download file
        │
        ▼
  Chunk document
        │
        ▼
  Embed (OpenAI Embeddings)
        │
        ▼
  Store in Supabase Vector DB
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Chatbot interface | Telegram |
| Workflow automation | n8n |
| AI model | OpenAI GPT-4o-mini |
| Voice transcription | Groq Whisper |
| Vector database | Supabase |
| Knowledge base | Google Drive + n8n RAG pipeline |
| CRM | Google Sheets |
| Calendar | Google Calendar |
| Email | Gmail |

---

## Tools Available to the Agent

| Tool | Purpose |
|---|---|
| `knowledge_base` | Semantic search over property documents and FAQs |
| `property_listings` | Retrieve available property listings |
| `crm` | Log qualified lead details to Google Sheets |
| `check_availability` | Fetch available inspection slots from Google Calendar |
| `schedule_inspection` | Create a calendar event and send confirmation email |
| `send_message` | Send a message to the client on Telegram |
| `send_property_image` | Send property images directly in Telegram chat |
| `email_lead` | Send confirmation email to the client after booking |

---

## Setup

### Prerequisites
- n8n instance (self-hosted or cloud)
- Telegram Bot Token (via BotFather)
- OpenAI API key
- Groq API key
- Supabase project with `documents` table and pgvector enabled
- Google account (Drive, Sheets, Calendar, Gmail)

### Environment Variables / Credentials to configure in n8n
```
- Telegram API credential
- OpenAI API credential
- Groq API credential
- Supabase API credential (URL + service role key)
- Google Drive OAuth2
- Google Sheets OAuth2
- Google Calendar OAuth2
- Gmail OAuth2
```

### Supabase Setup
Run this in your Supabase SQL editor to create the vector table:
```sql
create extension if not exists vector;

create table documents (
  id uuid primary key default gen_random_uuid(),
  content text,
  metadata jsonb,
  embedding vector(1536)
);

create index on documents using ivfflat (embedding vector_cosine_ops);
```

### Knowledge Base
Upload property documents, FAQs, and listing data as PDF or text files to your configured Google Drive folder. The RAG pipeline detects new uploads automatically and indexes them into Supabase.

---

## Key Design Decisions

**Session memory keyed to Telegram chat ID**
Each conversation maintains context using the user's Telegram chat ID as the session key — so every user has their own isolated memory thread.

**Voice note pipeline**
Telegram sends voice notes as `.oga` files. The workflow converts them to `.ogg` before passing to Groq Whisper for transcription, then standardises the output to match the text message format before hitting the agent.

**RAG over hardcoded listings**
Property data lives in a vector database rather than being hardcoded into the prompt. This means the knowledge base updates automatically when new documents are added to Google Drive — no workflow changes needed.

---

## Project Context

Built as a production-ready AI property assistant for a real estate company. The brief required semantic property search, lead capture, inspection booking, and CRM integration — all handled conversationally without a human agent.

---

*Built by Chibugo Ohanyiri — [Arcflow Labs](https://github.com/ChibugoOhanyiri/AI-automation-portfolio)*
