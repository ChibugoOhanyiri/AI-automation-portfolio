# Calendar & Meeting Prep Automation
<img width="1747" height="628" alt="Screenshot 2026-05-03 093947" src="https://github.com/user-attachments/assets/0b3cf113-069e-43e9-80d2-d3c17269e87e" />


> **An n8n workflow that fetches tomorrow's meetings from Google Calendar, enriches each attendee with context from HubSpot, ClickUp, Gmail, Fyxer, and LinkedIn, and delivers a structured meeting prep brief to your inbox every morning — written by Claude.**

---

## Preview

### Email Brief — Desktop


<img width="1667" height="721" alt="Screenshot 2026-05-03 151347" src="https://github.com/user-attachments/assets/71f2c17c-3a0c-4c8d-911e-35d774d9b9e9" />







<img width="1663" height="694" alt="image" src="https://github.com/user-attachments/assets/bc96b807-8d1a-47b6-8236-b4c302d89f4a" />


### Email Brief — Mobile

<img width="512" height="754" alt="Screenshot 2026-05-03 133035" src="https://github.com/user-attachments/assets/957c4e92-0339-4d98-8651-66e45d5dd8bc" />


## Table of Contents

- [What It Does](#what-it-does)
- [Architecture Overview](#architecture-overview)
- [Workflow Groups](#workflow-groups)
- [Data Sources](#data-sources)
- [Edge Cases Handled](#edge-cases-handled)
- [Sample Output](#sample-output)
- [Setup Guide](#setup-guide)
- [Mock vs Live Integrations](#mock-vs-live-integrations)
- [Key Architecture Decisions](#key-architecture-decisions)
- [What I Would Build Next](#what-i-would-build-next)
- [Cost Estimate](#cost-estimate)
- [File Structure](#file-structure)

---

## What It Does

A founder runs 5–10 meetings per day. Manually pulling context from HubSpot, Gmail, ClickUp, Fyxer, and LinkedIn before each meeting takes 20–30 minutes he does not have.

This workflow runs every evening and:

1. Fetches all events from Google Calendar for the next day
2. Filters out non-meetings (focus blocks, reminders, all-day events, lunch holds)
3. Classifies each real meeting as **internal**, **external 1:1**, or **group**
4. For each attendee, pulls context from up to 6 data sources in parallel
5. Assembles all context into a structured prompt
6. Calls Claude to write a clean, scannable meeting prep brief
7. Delivers the brief as a formatted HTML email before the day starts

The brief is designed to be read in 2–3 minutes.

---

## Architecture Overview

```
Google Calendar
      │
      ▼
Filter Real Meetings
(remove focus blocks, reminders, all-day events)
      │
      ├── No meetings? → Send "no meetings today" email
      │
      ▼
Switch — Classify by meeting type
      │
      ├── INTERNAL ──────────────────────────────────────────────────────┐
      │   ├── ClickUp: fetch all tasks                                    │
      │   ├── Fyxer: get last meeting summary per attendee               │
      │   └── Match: tasks + summaries to each attendee by email         │
      │                                                                   │
      ├── EXTERNAL 1:1 ──────────────────────────────────────────────────┤
      │   │                                                               │
      ├── GROUP ─→ Tag + split per attendee ────────────────────────────►│
      │                                                                   │
      │   Shared enrichment pipeline (runs per attendee):                 │
      │   ├── HubSpot: contact record + deal stage                       │
      │   ├── Gmail: last 3 email threads (simulated)                    │
      │   ├── Fyxer: last meeting summary (simulated)                    │
      │   ├── ClickUp: open tasks matched by email/name                  │
      │   └── LinkedIn: recent posts via Apify (simulated)               │
      │                                                                   │
      └── Merge all routes ◄──────────────────────────────────────────────┘
            │
            ▼
      Prepare Data for AI
      (assemble context object per attendee, group by meeting)
            │
            ▼
      Standardize Prompt for Claude
      (format into structured text, build API request body)
            │
            ▼
      Claude API (claude-sonnet-4)
      (returns structured JSON: internal[], external[], group[])
            │
            ▼
      Parse + Format HTML email
            │
            ▼
      Gmail → founder's inbox
```

**Architecture type:** Deterministic pipeline with a single AI inference step. Not agent-based — see [Key Architecture Decisions](#key-architecture-decisions) for the reasoning.

---

## Workflow Groups

The canvas is organised into 7 labelled sticky note groups:

| Group | Nodes | Purpose |
|---|---|---|
| **Trigger & Scheduling** | Manual trigger, Schedule trigger | Kicks off the workflow |
| **Meeting Gate & Classification** | Calendar fetch, Filter, IF, Switch | Filters and classifies meetings |
| **Task & Calendar Enrichment for Internal Team** | ClickUp fetch, Fyxer mock, Merge, Match | Enriches internal team meetings |
| **Routing Logic for Group and External Meetings** | Standardize, Tag+split, Combine | Prepares external/group items for shared pipeline |
| **Contact Enrichment & External Context** | HubSpot nodes, Gmail mock, Fyxer mock, ClickUp match, LinkedIn mock | Enriches each external attendee |
| **Data Preparation** | Prepare data for AI, Standardize prompt | Assembles and formats the Claude prompt |
| **AI Brief Generation & Delivery** | Claude HTTP, Extract JSON, Arrange HTML, Gmail send | Generates and delivers the brief |

---

## Data Sources

| Source | What it provides | Status in this build |
|---|---|---|
| **Google Calendar** | Meeting title, time, attendees, recurring flag | ✅ Live |
| **HubSpot** | Contact record, job title, company, lifecycle stage, deal name, deal stage, deal amount | ✅ Live (mock contacts) |
| **ClickUp** | Open tasks matched to attendee by email or name | ✅ Live (mock tasks) |
| **Fyxer** | Last meeting date, summary, outstanding action items | 🟡 Simulated |
| **Gmail** | Last 3 email thread subjects, dates, snippets | 🟡 Simulated |
| **LinkedIn via Apify** | Profile headline, 2 most recent posts | 🟡 Simulated |
| **Claude API** | Meeting prep brief generation | ✅ Live |

---

## Edge Cases Handled

| Situation | Handling |
|---|---|
| No meetings tomorrow | Sends a "no meetings today" notification email. No empty brief generated. |
| All-day events | Filtered out by checking for the absence of `dateTime` on the start field. |
| Non-meeting events | Filtered by keywords: focus, block, lunch, ooo, hold, reminder, break. |
| Only the organiser on the invite | Filtered out after removing the organiser from the attendee list. |
| Attendee not in HubSpot | Brief proceeds with available context. HubSpot section is omitted cleanly. |
| No deal associated with contact | Returns `{ note: 'No active deal found' }`. Claude omits the deal section. |
| No Fyxer summary available | Returns `null`. Claude omits the last meeting section. |
| No ClickUp tasks found | Returns `noTasksFound: true`. Claude omits the tasks section. |
| Null or missing data in any field | Handled by `safe()` helper in prompt code. Returns empty string, never crashes. |
| Recurring meeting | `isRecurring: true` is stamped and passed to Claude. System prompt instructs Claude to focus on what has changed, not background context. |
| Group meeting (2+ external attendees) | Each attendee is split into a separate item, enriched individually, then recombined into one group brief. |

---

## Sample Output

The brief is delivered as an HTML email structured like this:

```
Hi [founder name], you have 4 meetings today, being Saturday, 2 May 2026.
Kindly find your meeting brief below.

────────────────────────────────

INTERNAL MEETINGS

10:30 AM | Weekly Team Sync
bubu and john attending. John has three outstanding actions from
April 22: Q2 ops report due Friday, CRM migration executive sign-off
pending, BDM position description review still outstanding. Focus on
realistic completion dates and whether he needs support to close these items.

────────────────────────────────

EXTERNAL MEETINGS

9:00 AM | Discovery Call - Ibrahim Lawal
Ibrahim Lawal, CTO at Microsoft. Deal at proposal sent stage, worth $2,500.
April 29 call confirmed the digital transformation scope across three business
units. Revised proposal addressing legacy systems concerns due today. CFO
budget approval expected by May 7. Push for a CFO feedback timeline and clarify
internal resistance from the legacy systems team.

3:30 PM | Check-in - Fatima Abubakar
Fatima Abubakar, COO at CredPal Africa. Closed a deal worth $4,000. Board
Sign-off expected by May 7. Follow-up on contract signing was overdue on Tuesday.
Confirm board meeting outcome and lock in May 15 kickoff date if signed.

────────────────────────────────

GROUP MEETINGS

12:00 PM | Proposal Review - Lara & Grace
Lara Okeke, COO at MediCore Health, and Grace Eze, Partnership Lead at
HealthLink Systems. The MediCore deal at the negotiation stage is worth $5,000. April 15
session identified timeline concerns for phase 1 and data migration scoping
issues. Revised proposal and compressed timeline due end of this week. Address
Grace's 12-week phase 1 concerns and clarifies HealthLink as a partner, not
subcontractor, before joint sign-off.
```

---

## Setup Guide

### Prerequisites

- n8n Cloud account (v2.17 or higher)
- Google Workspace account (Calendar + Gmail OAuth2)
- HubSpot account with a Private App token
- ClickUp account with OAuth2 credentials
- Anthropic API key (claude-sonnet-4 access)
- Apify account (for production LinkedIn enrichment)
- Fyxer account (for production meeting summaries)

### Step 1 — Import the workflow

In n8n: **Settings → Import Workflow** → upload `Calendar_and_Meeting_Prep_Workflow_Main.json`

### Step 2 — Configure credentials

| Credential name in n8n | Type | Used by |
|---|---|---|
| Google Calendar OAuth2 API | OAuth2 | Fetch all meetings for the next day |
| Gmail OAuth2 API | OAuth2 | Let founder know (no meetings), Send a message |
| ClickUp OAuth2 API | OAuth2 | Get many tasks, Get all tasks on ClickUp |
| HubSpot Private App Token | HTTP Header Auth (Authorization: Bearer) | Get HubSpot Contact record, Get deal stage and info |
| Anthropic API Key | HTTP Header Auth (x-api-key) | Claude generates brief |

### Step 3 — Update configuration constants

In the **Filter Real Meeting** code node, update:

```javascript
const ORGANIZER_EMAIL = 'Your_email';     //  your email
const INTERNAL_DOMAIN = 'Your_company's_domain';             // Company domain
```

In the **Let founder know that there are no meetings** Gmail node:
- Set `sendTo` to founder's email address

In the **Send a message** Gmail node:
- Set `sendTo` to founder's email address

### Step 4 — Configure ClickUp

In **Get many tasks** and **Get all task on ClickUp** nodes:
- Set team, space, folder, and list to match the team's ClickUp workspace

### Step 5 — Replace simulations with live integrations

| Node to replace | Production implementation |
|---|---|
| Fyxer meeting summary simulation | HTTP GET to Fyxer API filtered by attendee email |
| Get last 3 email threads simulation | Gmail Get Many Threads filtered by participant email |
| LinkedIn lookup API simulation via Apify | HTTP POST to Apify actor with LinkedIn URL from HubSpot contact |

### Step 6 — Activate the Schedule Trigger

The Schedule Trigger is included but deactivated. Set the cron expression for your desired run time (e.g. 10 PM daily) and activate it.

---

## Mock vs Live Integrations

This build was constructed without access to the live company's accounts. The architecture, routing, data matching, and AI generation are all production-ready. The following are simulated:

**Gmail threads** — Simulated because Gmail thread search requires access to founder's mailbox. The mock data reflects realistic email history with the demo contacts.

**Fyxer summaries** — Simulated using real anonymised transcripts from Fyxer sessions provided in the task materials. The data was adapted to fit the mock contacts and reflects the expected Fyxer API response schema.

**LinkedIn via Apify** — Simulated because there are no real LinkedIn profiles for the mock contacts. The Apify integration path is fully documented and production-ready.

---

## Key Architecture Decisions

### Why n8n over Make or Zapier

n8n was selected because it provides native JavaScript Code nodes, which this workflow requires for complex data matching (HubSpot contact to deal by ID with name fallback), null safety, and structured prompt assembly. Make and Zapier do not provide this level of scripting capability without external dependencies.

### Why a deterministic pipeline, not an AI agent

An agent-based approach would use an LLM to decide at runtime which APIs to call and in what order. This is the wrong choice for this workflow for three reasons:

1. **The data sources are fixed.** Every meeting needs the same five context sources. There is no decision to make.
2. **Cost.** Agents make multiple LLM calls per step. This workflow runs every day. The compounded cost is significantly higher than one structured Claude call.
3. **Reliability.** Agents introduce non-determinism. A daily briefing system must behave consistently on every run.

The correct use of AI here is at the final step: synthesising pre-assembled, structured context into natural, readable language. Everything before that step is deterministic code.

### Why one Claude call for all meetings

Rather than one Claude call per meeting route, all context is assembled into a single structured prompt and sent in one call. This reduces API cost, reduces latency, and gives Claude full context across all meetings (which occasionally produces useful cross-meeting observations in the brief).

### How LinkedIn is solved

LinkedIn does not offer a public API for profile or post data. The Apify LinkedIn Profile Scraper retrieves public profile data using a maintained cloud actor without requiring LinkedIn API access. In production, the attendee's LinkedIn URL (stored as a custom property in HubSpot) is passed to Apify, which returns headline and recent posts. This data is then included in the Claude prompt.

---

## HubSpot Integration

<img width="1919" height="807" alt="Screenshot 2026-05-03 153022" src="https://github.com/user-attachments/assets/83489649-be63-4a06-a376-59901c3d2619" />


<img width="1872" height="811" alt="Screenshot 2026-05-03 152959" src="https://github.com/user-attachments/assets/7e79b899-f491-4259-acdb-c4741e464257" />




---

## ClickUp Integration

<img width="1912" height="776" alt="Screenshot 2026-05-03 153150" src="https://github.com/user-attachments/assets/63ace69c-ae3e-4cd5-aa9c-8334ddc02891" />




<img width="1901" height="755" alt="Screenshot 2026-05-03 153131" src="https://github.com/user-attachments/assets/cf6a6363-cbb6-4a89-b300-f6d284990362" />


## What I Would Build Next

**Immediate (with account access)**
- Replace Gmail simulation with live Gmail API thread search by participant email
- Replace Fyxer simulation with live Fyxer API call
- Replace LinkedIn simulation with live Apify calls using LinkedIn URLs from HubSpot
- Activate Schedule Trigger and confirm delivery timing

---

## Cost Estimate

| Component | Monthly cost |
|---|---|
| n8n Cloud | Free tier or $20/month |
| Claude API (claude-sonnet-4, ~10 meetings/day) | Under $5/month |
| Apify LinkedIn scraping (production) | $15/month |
| HubSpot, ClickUp, Google Workspace, Fyxer | Existing subscriptions |
| **Total additional cost** | **$20–$40/month** |

Time saved: 20–30 minutes of manual prep per day across 4–10 meetings.

---

## File Structure

```
├── Calendar_and_Meeting_Prep_Workflow_Main.json   # n8n workflow — import this
├── README.md                                       # This file
├── docs/
│   ├── Meeting_Prep_Client_Documentation.docx
│   ├── Meeting_Prep_Developer_Documentation.docx
│   └── Meeting_Prep_Decision_Document.docx
```
