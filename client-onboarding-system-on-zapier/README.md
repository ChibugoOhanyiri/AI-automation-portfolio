# Automated Client Onboarding System — Zapier + Airtable

A fully automated client onboarding pipeline for a brand and marketing agency. The moment a client submits an intake form, the system cleans their data, stores them in the CRM, routes them by service tier, creates their project tasks, sends a personalised welcome email, notifies the right team on Slack, and creates their Google Drive folder — all without a human touching anything.
 
Screenshots are attached below👇
---

## The Problem

Agencies onboarding clients manually deal with the same friction every time — inconsistent data entry, duplicate CRM records, the wrong team getting notified, welcome emails sent late or forgotten, and project tasks that need to be created one by one. This system eliminates all of it.

---

## What It Does

**Data cleaning first**

Every field from the intake form is cleaned before anything else runs:
- Client name: whitespace trimmed, capitalised, split into first and last name
- Email: whitespace trimmed
- Company name: whitespace trimmed
- Project start date: formatted to a consistent date format

**Duplicate prevention**

Airtable's Create or Update action uses the client's email as the matching key. A returning client updates their existing record. A new client gets a fresh one. No duplicates, ever.

**Lookup Tables for dynamic routing**

Two Lookup Tables run before any path branching:
- Service Tier → Team Lead Email (so emails always go to the right person)
- Service Tier → Google Drive Folder Name (so folders are created in the right location)

If a team member changes, you update one row in one table. Nothing else in the workflow needs touching.

**Three paths by service tier**

The workflow splits at step 11 into Basic, Standard, and Premium paths. Each tier gets exactly what it needs — nothing more, nothing less.

**Airtable native automation for task creation**

Rather than creating 3–4 Airtable task records directly in Zapier (which burns task limits), the workflow delegates task creation to Airtable's own native automation. The moment Zapier creates a client record in the Client Intake table, Airtable detects it and chains through the task creation automatically — no Zapier tasks consumed per task.

---

## Architecture

### Zapier Workflow (23 steps)

```
Google Forms — New Form Response
        │
        ▼
Formatter ×7 — Data Cleaning Pipeline
  Step 2: Trim whitespace from names
  Step 3: Capitalise names
  Step 4: Split client name into first/last
  Step 5: Format project start date
  Step 6: Trim whitespace from email
  Step 7: Trim whitespace from company name
        │
        ▼
Airtable — Store in CRM (Create or Update)
  Matching key: client email
  Prevents duplicates automatically
        │
        ▼
Formatter — Lookup Table: Service Tier → Team Lead Email
        │
        ▼
Formatter — Lookup Table: Service Tier → Google Drive Folder Name
        │
        ▼
Paths — Split by Service Tier
   ┌──────────────┬──────────────────┬──────────────┐
   ▼              ▼                  ▼
Path C          Path B            Path A
BASIC           STANDARD          PREMIUM
   │              │                  │
   ▼              ▼                  ▼
Gmail           Google Drive      Google Drive
Welcome email   Create folder     Create folder
   │              │                  │
   ▼              ▼                  ▼
Slack           Gmail             Gmail
Basic team      Welcome email     Welcome email
   │              │               (full timeline,
   ▼              ▼               team intro,
Google Sheets   Slack             next steps)
Tracking row    Standard team        │
                                     ▼
                                  Slack
                                  Premium team
                                  (priority alert)
```

### Airtable Native Automation (task creation)

```
Trigger: When a record is created in Client Intake table
        │
        ▼
If Service Tier = Standard:
  Create record in Client Task → "Brand discovery call"
  Create record in Client Task → "Strategy document creation"
  Create record in Client Task → "Design kickoff"
  Create record in Client Task → "Review meeting"
        │
If Service Tier = Premium:
  Create record in Client Task → "Onboarding call"
  Create record in Client Task → "Content calendar setup"
  Create record in Client Task → "First post review"
        │
Basic: No tasks created
```

Each task record links back to the triggering Client Intake record via a linked record field — so every client's full task list sits inside their CRM record.

---

## Service Tier Breakdown

| Feature | Basic | Standard | Premium |
|---|---|---|---|
| Google Drive folder | ✗ | ✓ | ✓ |
| Welcome email | Simple | Streamlined | Full timeline + team intro + checklist |
| Slack notification | Basic team | Standard team lead | Priority alert — Premium team |
| Google Sheets tracking | ✓ | ✗ | ✗ |
| Airtable tasks created | 0 | 4 | 3 |
| Task types | — | Discovery, Strategy, Design, Review | Onboarding, Content calendar, First post review |

---

## Tech Stack

| Layer | Tool |
|---|---|
| Workflow automation | Zapier |
| Client intake | Google Forms |
| Data cleaning | Formatter by Zapier ×7 |
| Dynamic routing | Lookup Table by Zapier ×2 |
| Path branching | Paths by Zapier |
| CRM | Airtable (Client Intake table) |
| Task management | Airtable (Client Task table + native automation) |
| File storage | Google Drive |
| Welcome emails | Gmail |
| Team alerts | Slack |
| Basic client log | Google Sheets |

---

## Airtable Base Structure

**Client Intake table**

| Field | Type | Notes |
|---|---|---|
| Client Name | Text | Cleaned and split by Zapier before entry |
| Client Email | Email | Used as duplicate-match key |
| Company Name | Text | Trimmed by Zapier |
| Service Tier | Single select | Basic / Standard / Premium |
| Project Start Date | Date | Formatted by Zapier |
| Notes | Long text | From intake form |

**Client Task table**

| Field | Type | Notes |
|---|---|---|
| Task Name | Text | Auto-created by Airtable automation |
| Company Name | Linked record → Client Intake | Links task to the right client |
| Service Tier | Lookup | Pulled from linked Client Intake record |
| Status | Single select | Todo / In Progress / Done |
| Due Date | Date | Set per task during creation |
| Assigned Team | Text | e.g. marketing |

---

## Key Design Decisions

**Formatter chain before CRM write** — data is cleaned at the source before it ever enters Airtable. This means the CRM always holds consistent, correctly formatted records regardless of how the client typed their name or email.

**Airtable Create or Update instead of always Create** — email as the matching key means the workflow handles both new and returning clients correctly in one step. No separate lookup or duplicate check needed.

**Lookup Tables over hardcoded values** — team lead emails and folder names are stored in Lookup Tables, not hardcoded in the email or Slack steps. If the Premium team lead changes, one table update propagates everywhere instantly.

**Airtable native automation for tasks** — delegating task creation to Airtable's own automation instead of Zapier saves 3–4 Zapier tasks per onboarding. At scale this matters. It also means task logic lives where the data lives — inside Airtable — rather than spread across two systems.

**Three distinct paths** — Basic, Standard, and Premium clients genuinely need different handling. Paths keeps the logic clean and explicit rather than using nested filters or multiple Zaps.

---

## Screenshots

| Screenshot | Description |
|---|---|
|  <img width="909" height="1618" alt="Screenshot 2026-03-16 145711" src="https://github.com/user-attachments/assets/4e774472-bd08-4780-b5b7-05b412e2d3cc" />
| Full 23-step Zapier workflow canvas |
| <img width="1426" height="641" alt="Screenshot 2026-03-16 154245" src="https://github.com/user-attachments/assets/3b29498f-8cb7-461f-acb3-f47d1fd256b1" />
 | Live Slack alert in #premium-team channel |
| <img width="1668" height="396" alt="Screenshot 2026-03-16 154322" src="https://github.com/user-attachments/assets/af0ea8a4-baf0-4bd3-9b92-17a82b3a2451" />
| Client Task table showing auto-created tasks linked to client |
| <img width="1033" height="775" alt="Screenshot 2026-03-16 154348" src="https://github.com/user-attachments/assets/fc5acd33-d209-4e8e-a1c2-e44259835798" />
 | Airtable native automation — task creation logic |
| <img width="976" height="1258" alt="Screenshot 2026-03-16 162048" src="https://github.com/user-attachments/assets/e04401fe-bb21-4217-b614-e8f212398b78" />
 | Workflow logic diagram of full system |
| <img width="421" height="702" alt="Screenshot 2026-03-16 172359" src="https://github.com/user-attachments/assets/abc4ef3b-a85f-47d2-a90c-2852c78c0ded" />
 | Premium tier HTML welcome email |

