# CRM Intelligence System

<img width="1496" height="681" alt="Screenshot 2026-05-04 112354" src="https://github.com/user-attachments/assets/9f065543-e486-4e52-9de5-ea0282dc9aaa" />


An automated CRM operations system built with n8n, HubSpot, and Claude AI. Designed for founder-led consulting businesses to eliminate manual lead tracking, automate client communications, and surface deal risks before they become lost revenue.

---

## Overview

This system removes the need for a founder to manually monitor their CRM. It handles two core operations automatically:

1. **New Lead Notification**: when a prospect submits an enquiry form, their contact and deal are created in HubSpot instantly and both the founder and the lead receive branded emails automatically.
2. **Stale Deal Alert**: every morning, the system checks for deals with no activity in 7+ days, generates an AI-written digest using Claude, and emails the founder a prioritised summary.

---

## Architecture

```
Workflow 1: New Lead Notification
Form Submission → Standardize Data → Create HubSpot Contact → Create HubSpot Deal → Notify Founder → Acknowledge Lead

Workflow 2: Stale Deal Alert
Schedule (9am daily) → Fetch All Deals → Filter Stale Deals → IF Stale → Aggregate → Claude AI Summary → Email Founder
                                                                           ↓ No stale deals
                                                                      No Operation
```

---

## Workflow 1 | New Lead Notification

### What it does
A prospect fills out an enquiry form. Within seconds, their contact record and deal are created in HubSpot, the founder receives a structured notification email, and the lead receives a branded acknowledgement email confirming their enquiry was received.

### Nodes

| Node | Type | Purpose |
|------|------|---------|
| On form submission | Google Sheets Trigger | Fires when a new form response is logged |
| Standardize data | Code | Cleans and normalises field names from the form |
| Create new contact record in HubSpot | HTTP Request | POST to HubSpot Contacts API |
| Create deal pipeline to "New Lead" | HTTP Request | POST to HubSpot Deals API with contact association |
| Send notification to founder | Gmail | Branded alert email with lead details |
| Send acknowledgement to lead | Gmail | Confirmation email to the prospect |

### Flow diagram

<img width="1285" height="335" alt="image" src="https://github.com/user-attachments/assets/74ab71b9-5a35-410d-8337-c3b50ef2cf69" />


### Trigger
Google Sheets | new row added when enquiry form is submitted.

Form fields captured:
- First Name
- Last Name
- Job Title
- Company Name
- Email
- What challenge are you facing? (Service Interest)
- How did you hear about us? (Lead Source)

### HubSpot Integration

Contact properties mapped:

```json
{
  "firstname": "{{ First Name }}",
  "lastname": "{{ Last Name }}",
  "email": "{{ Email }}",
  "company": "{{ Company Name }}",
  "jobtitle": "{{ Job Title }}",
  "lead_source": "{{ How did you hear about us? }}",
  "service_interest": "{{ What challenge are you facing? }}"
}
```

Deal created automatically with:
- Deal name: `[First Name] [Last Name] - [Service Interest]`
- Stage: New Lead
- Associated to contact via HubSpot Contacts API ID

<img width="1919" height="807" alt="Screenshot 2026-05-03 153022" src="https://github.com/user-attachments/assets/afbf00b3-8a3d-43be-857a-73442301ba13" />

---


<img width="1872" height="811" alt="Screenshot 2026-05-03 152959" src="https://github.com/user-attachments/assets/7d28bbc8-71dd-4a4f-8fd2-0a8188e97ea3" />


### Email | Founder Notification

Triggered immediately on new lead. Contains name, company, job title, service interest, lead source and date.

<img width="1649" height="707" alt="image" src="https://github.com/user-attachments/assets/d8db934f-17f4-4300-bac3-74dd072126ab" />


### Email | Lead Acknowledgement

Sent to the prospect confirming receipt of their enquiry, summarising what they submitted, and outlining next steps (response within 24 hours, discovery call).

<img width="1600" height="709" alt="image" src="https://github.com/user-attachments/assets/9aa1a219-7eeb-454f-bcbb-825c0c0373c5" />


---

## Workflow 2 | Stale Deal Alert

### What it does
Runs automatically every morning. Fetches all active deals from HubSpot, filters for any with no activity in 7 or more days, aggregates them, passes them to Claude for an AI-generated plain-text summary, and emails the founder a digest so no deal goes cold silently.

### Nodes

| Node | Type | Purpose |
|------|------|---------|
| Schedule Trigger | Schedule | Fires daily at 9am |
| Get all current deals in the pipeline | HTTP Request | GET all deals with properties from HubSpot |
| Filter Stale Deals | Code (JavaScript) | Maps stage names, filters inactive deals, calculates days since last activity |
| If deals are inactive for 7+ days | IF | Routes stale deals to alert path, clean deals to no-operation |
| Aggregate stale deals | Aggregate | Collapses multiple stale deal items into one object for Claude |
| Generate summary using Claude API | HTTP Request | POST to Anthropic API — generates plain-text founder summary |
| Send stale deal alert to founder | Gmail | Branded digest email with AI summary and deal breakdown |
| No Operation, do nothing | No-op | Clean exit when no stale deals exist |

### Flow diagram

<img width="1378" height="434" alt="image" src="https://github.com/user-attachments/assets/254d722f-047c-47e2-bfd2-f98f10d15c58" />


### Deal Stage Mapping

HubSpot internal stage IDs are translated to readable labels:

```javascript
const stageMap = {
  "decisionmakerboughtin": "New Lead",
  "appointmentscheduled": "Discovery Call Booked",
  "qualifiedtobuy": "Proposal Sent",
  "presentationscheduled": "Negotiation",
  "contractsent": "Won",
  "stage_0": "Lost"
};
```

### Filter Logic

```javascript
// Deals are flagged stale if:
// 1. They have a name and stage (not junk records)
// 2. They are not Closed Won or Closed Lost
// 3. They have had no activity for 7 or more days

const daysSince = Math.floor(
  (Date.now() - new Date(lastModified).getTime()) / (1000 * 60 * 60 * 24)
);
return daysSince >= STALE_DAYS;
```

### Claude AI Summary

Stale deals are aggregated and passed to Claude with a plain-text instruction:

```
You are an operations assistant for a founder-led consulting business.
Write a short 2-3 sentence summary about the following stale deals that need follow-up.
Be direct and action-oriented. Use plain text only — no bold, no asterisks, no markdown.
```

Claude returns a natural language summary that appears at the top of the digest email.

### Email | Stale Deal Digest

Contains the AI-generated summary followed by individual deal cards showing deal name, stage, last activity date, days inactive, and a direct HubSpot link for each deal.

<img width="1533" height="659" alt="Screenshot 2026-05-04 113303" src="https://github.com/user-attachments/assets/1e34abeb-2e84-4ef2-9fb6-17ea0df97e25" />


---

## Tech Stack

| Tool | Role |
|------|------|
| n8n | Workflow automation and orchestration |
| HubSpot | CRM — contacts, deals, pipeline |
| Google Sheets | Form response capture and trigger |
| Gmail | Email delivery for all notifications |
| Anthropic Claude API | AI-generated deal digest summaries |

---

## Environment Variables

Store these as credentials in n8n. Never hardcode in workflow nodes.

```
HUBSPOT_PRIVATE_APP_TOKEN=
ANTHROPIC_API_KEY=
GMAIL_OAUTH_CREDENTIALS=
```

HubSpot token is a Private/legacy App token with the following scopes:
- `crm.objects.contacts.read`
- `crm.objects.contacts.write`
- `crm.objects.deals.read`
- `crm.objects.deals.write`

---

## Setup

1. Clone or import the workflow JSON into your n8n instance
2. Add credentials for HubSpot, Gmail and Anthropic in n8n Settings → Credentials
3. Connect your Google Form to a Google Sheet and link the sheet to the trigger node
4. Update HubSpot pipeline stage IDs in the Filter Stale Deals code node to match your account
5. Set `STALE_DAYS` to your preferred threshold (default: 7)
6. Activate the Schedule Trigger for Workflow 2
7. Test Workflow 1 by submitting a form entry

---

## Anonymization Note

All contact names, email addresses, company names, API keys, and account-specific identifiers have been removed from this repository. Pipeline stage IDs are specific to HubSpot accounts and must be reconfigured on import. See setup instructions above.

---

## License

MIT
