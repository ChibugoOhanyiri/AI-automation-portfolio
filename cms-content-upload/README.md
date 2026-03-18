# CMS Content Upload Automation

<img width="1802" height="760" alt="Screenshot 2026-03-18 064932" src="https://github.com/user-attachments/assets/d2849e1e-0744-4dde-aaac-c41a9ada483e" />

An n8n workflow that automates bulk uploading of provider/directory data from Google Sheets into a headless CMS (Strapi). Built to handle large datasets that are impractical to enter manually through the CMS interface.

---

## What It Does

Reads provider records from a Google Sheets queue, creates a draft entry in the CMS for each record, publishes it immediately, then deletes the processed row from the sheet — working through the queue batch by batch until all records are published.

The workflow runs on a 3-second schedule loop, processing one record per cycle to avoid rate limiting and ensure reliable publishing.

---

## Architecture

```
Schedule Trigger (every 3 seconds)
        │
        ▼
Set authentication token
        │
        ▼
Read first unprocessed row from Google Sheets
        │
        ▼
Get Payload (extract first record)
        │
        ▼
Generate summary (first sentence of 'About' field)
        │
        ▼
POST to CMS — Create Draft entry
        │
        ▼
POST to CMS — Publish the draft
        │
        ▼
Delete processed row from Google Sheets
        │
        ▼
Loop Over Items (continue to next record)
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Workflow automation | n8n |
| Data source | Google Sheets |
| CMS | Strapi (headless CMS) |
| Auth | JWT Bearer token |

---

## Google Sheets Structure

Each row in the queue sheet represents one provider to be uploaded. Expected columns:

| Column | Description |
|---|---|
| `Provider` | Provider/company name |
| `Country` | Primary country |
| `service_type` | Type of service offered |
| `About` | Full description (first sentence auto-extracted as summary) |
| `Countries` | JSON array of countries served |
| `address` | Physical address |
| `phone_number` | Contact number |
| `email` | Contact email |
| `website` | Website URL |
| `Twitter` | Twitter/X profile URL |
| `facebook_url` | Facebook URL |
| `linkedin_url` | LinkedIn URL |
| `instagram_url` | Instagram URL |

---

## How the Summary is Generated

A JavaScript Code node extracts the first sentence of the `About` field automatically:

```javascript
const summary = $input.first().json.About.split('.')[0] + '.';
return { summary }
```

This means you don't need a separate summary column — it's derived from the description on upload.

---

## Setup

### Prerequisites
- n8n instance
- Google Sheets with provider data in the expected format
- Strapi instance with the target collection type configured
- Valid Strapi JWT token with content editor permissions

### Credentials to configure in n8n
```
- Google Sheets OAuth2
- HTTP Request node (no credential needed — auth passed via header)
```

### Configuration
1. Update the `Token` Set node with your Strapi JWT token
2. Update the Google Sheets nodes to point to your queue sheet
3. Update the HTTP Request URLs to match your Strapi instance and collection type
4. Adjust the Schedule Trigger interval if needed (default: 3 seconds)

### Getting a Strapi JWT Token
Log into your Strapi admin panel → go to your profile → API Tokens → create a token with "Full access" or "Custom" permissions scoped to your collection type.

---

## Key Design Decisions

**Processed rows are deleted** — once a record is successfully created and published, the row is removed from the sheet. This means the sheet acts as a live queue — only unpublished records remain. If the workflow is interrupted, you won't accidentally re-publish records.

**Draft then publish** — the workflow creates a draft first, then publishes in a separate request. This mirrors Strapi's content lifecycle and ensures content only goes live after a successful draft creation.

**3-second loop** — slow enough to avoid overwhelming the CMS API, fast enough to process hundreds of records in a reasonable time. Adjust the schedule interval based on your Strapi instance's rate limits.

**Auto summary extraction** — no need to manually write summaries. The first sentence of the `About` field is extracted and used as the CMS summary field automatically.

---

## Use Case Context

Built to solve a data ingestion problem: a large directory of global service providers needed to be published to a CMS. The dataset was too large for manual entry and the CMS had no native bulk import. This workflow turned a Google Sheet into a reliable automated publishing queue.

---
