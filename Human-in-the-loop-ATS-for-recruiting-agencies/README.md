# Human-in-the-Loop ATS for Recruiting Agencies (n8n)

<img width="1568" height="674" alt="image" src="https://github.com/user-attachments/assets/1dcfdf2d-e7e6-4f17-87cd-c7d3800d882f" />


A mini Applicant Tracking System (ATS) built in n8n for a recruiting agency. Two coordinated workflows handle everything from application intake to interview scheduling with humans staying in control of every hiring decision.

> **No AI was used to decide who gets hired.** Automation handles the admin. Humans handle the judgment.

---

## What It Does

Recruiting teams waste hours on repetitive admin tasks validating emails, deduplicating candidates, chasing resumes, and manually sending interview invites. This system automates all of that while keeping recruiters firmly in the loop for every screening and rejection decision.

The recruiter's only job is to update a candidate's **Stage** in Airtable. Everything else fires automatically from that single action.

---

## Features

- **Typeform-powered application intake** — structured candidate submissions including resume link, role, and contact details
- **Email validation** — verifies deliverability before any record is created
- **Duplicate detection** — searches Airtable by email before creating a new record; silently drops duplicates
- **Auto-generated Airtable record** — creates a clean candidate profile with standardized fields on first application
- **Resume auto-download & storage** — converts the submitted Google Drive resume link to PDF and uploads it to a dedicated `Candidate n8n` Drive folder
- **Airtable record update** — replaces the raw resume link with the stored Drive file link and advances stage to `Screen`
- **Dual notifications on intake** — notifies the recruiter and sends the applicant a confirmation email
- **Stage-change trigger** — Workflow 2 polls Airtable every minute and fires whenever a recruiter updates the `Stage` field
- **Smart routing via Switch node** — routes candidates to interview scheduling or rejection flow based on the new stage
- **Interview scheduling** — creates a Google Calendar event with a Google Meet link, adds both candidate and recruiter as attendees, and sends the candidate a branded HTML interview invitation email
- **Duplicate interview guard** — checks if an interview has already been scheduled (`Interview Scheduled = true`) before creating a new calendar event
- **Rejection email with reason** — maps the recruiter's selected rejection reason from Airtable to a structured, human-readable rejection email
- **Airtable always stays current** — every action updates the candidate's record for full pipeline traceability

---

## Workflow Architecture

### Workflow 1: Application Intake & ATS Setup

```
Typeform Trigger
  → Edit Fields (normalize data)
  → Validate email (deliverability check)
      └── Invalid → No Operation, stop
  → Search Airtable by email
      → Does candidate already exist?
          ├── YES → No Operation (silent dedup, stop)
          └── NO  → Create Airtable record (Stage: New, Status: pending)
                      → Standardize candidate_record_id
                      → Download resume (Google Drive link → PDF)
                      → Upload resume to Google Drive (Candidate n8n folder)
                      → Update Airtable record
                          - Resume URL → Drive file link
                          - Application Status → completed
                          - Stage → Screen
                      → Notify recruiter (Gmail)
                      → Inform applicant of application status (Gmail)
```

**Typeform fields captured:**
- First Name, Last Name, Email, Phone Number, Address
- Position Applied For
- Drop a link to your resume (Google Drive URL)

**Airtable record created with:**

| Field | Initial Value |
|---|---|
| First Name / Last Name | From form |
| Email / Phone Number | From form |
| Role Applied for | From form |
| Resume URL | Google Drive PDF link (updated after upload) |
| Stage | `New` → auto-advanced to `Screen` |
| Application Status | `pending` → `completed` |

---

### Workflow 2: Candidate Qualification & Interview Handling

```
Airtable Trigger (polls every minute — watches Stage field)
  → Routes rejected/interview candidates (Switch node)
      ├── Stage = "Interview"
      │     → Check if Interview Scheduled = true
      │           ├── YES → No Operation (prevent duplicate invite)
      │           └── NO  → Create Google Calendar event (Google Meet)
      │                       → Edit Fields (extract meeting ID, link, timezone)
      │                       → Notify applicant (HTML interview email)
      │                       → Update Airtable: Interview Scheduled = true
      │
      └── Stage = "Rejected"
            → Map rejection reason to template
            → Send rejection email (Gmail)
```

**The recruiter's only action is updating Stage in Airtable.**
The workflow handles everything downstream automatically.

---

## Pipeline Stages

| Stage | Meaning | Triggered By |
|---|---|---|
| `New` | Application received | Workflow 1 (auto) |
| `Screen` | Ready for recruiter review | Workflow 1 (auto) |
| `Interview` | Recruiter approved for interview | Recruiter (manual) |
| `Offer` | Interview passed | Recruiter (manual) |
| `Hired` | Offer accepted | Recruiter (manual) |
| `Rejected` | Not moving forward | Recruiter (manual) |

**Rejection reasons available to recruiter (dropdown in Airtable):**
- Not enough experience
- Skills mismatch
- Role filled
- Location/timezone mismatch
- Compensation mismatch
- Other

---

## Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** | Workflow automation engine |
| **Typeform** | Candidate application form |
| **Airtable** | ATS database & stage management |
| **Google Drive** | Resume storage (`Candidate n8n` folder) |
| **Google Calendar** | Interview scheduling with Google Meet |
| **Gmail** | All candidate and recruiter notifications |
| **HTTP Request node** | Resume download from Google Drive (PDF export) |
| **Switch node** | Stage-based routing (Interview vs. Rejected) |
| **IF node** | Duplicate detection & interview dedup guard |

---

## Airtable Schema

| Field | Type | Description |
|---|---|---|
| `First Name` / `Last Name` | String | Candidate name |
| `Full Name` | Formula | Auto-computed by Airtable |
| `Email` | String | Used as dedup key |
| `Phone Number` | String | Contact number |
| `Role Applied for` | String | Position from form |
| `Resume URL` | String | Google Drive PDF link |
| `Stage` | Select | `New` / `Screen` / `Interview` / `Offer` / `Hired` / `Rejected` |
| `Application Status` | Select | `pending` / `completed` |
| `Interview start` / `Interview end` | DateTime | Set by recruiter for scheduling |
| `Recruiter Name` / `Recruiter Email` | String | Added to calendar invite |
| `Interview Scheduled` | Boolean | Prevents duplicate calendar events |
| `Rejection reason` | Select | Mapped to rejection email template |
| `Rejection note (optional)` | String | Free-text note from recruiter |

---

## Automated Emails

| Trigger | Recipient | Content |
|---|---|---|
| Application received | Applicant | Confirmation of submission |
| Application received | Recruiter | New candidate alert with details |
| Stage → `Interview` | Applicant | Branded HTML email with date, time, timezone, Google Meet link, Meeting ID, and prep tips |
| Stage → `Rejected` | Applicant | Structured rejection email with mapped reason |

---

## End-to-End Flow

```
Candidate submits Typeform
        │
        ▼
Validate email → invalid? → Stop
        │ valid
        ▼
Already in Airtable? → yes → Stop (silent dedup)
        │ no
        ▼
Create Airtable record (Stage: New)
        │
        ▼
Download resume → Upload to Drive → Update record (Stage: Screen)
        │
        ▼
Notify recruiter + applicant
        │
        ▼
[RECRUITER REVIEWS IN AIRTABLE]
        │
   Updates Stage
        │
   ┌────┴────┐
Interview  Rejected
   │           │
Create     Map reason →
Calendar   Send rejection
event      email
   │
Notify applicant
(HTML email + Meet link)
   │
Update: Interview Scheduled = true
```

---

## Setup Instructions

### Prerequisites
- n8n instance (cloud or self-hosted)
- Typeform account with form connected
- Airtable base with the schema above
- Google account with Drive, Calendar, and Gmail OAuth2 connected in n8n
- Gmail OAuth2 credentials for both recruiter and outbound notifications

### Steps

1. **Import the workflow** — upload `Recruiting_agency.json` into n8n via *Import from file*
2. **Connect Typeform** — link your Typeform account and select the correct form ID
3. **Connect Airtable** — update `base` and `table` IDs in all Airtable nodes with your own base/table
4. **Connect Google Drive** — update the `folderId` in the Upload resume node to your own Drive folder
5. **Connect Google Calendar** — update the calendar ID in the *Create an event* node to the recruiter's calendar
6. **Connect Gmail** — link Gmail credentials for all outbound email nodes
7. **Update email templates** — customize the interview invite and rejection email content as needed
8. **Activate both workflows** — Workflow 1 activates on Typeform submissions; Workflow 2 polls Airtable every minute

---

## Design Decisions

**Why human-in-the-loop?**
Hiring decisions carry real consequences for people's careers. Automating the screening decision with AI introduces bias risks and removes accountability. This system keeps all judgment with humans — automation only removes admin friction.

**Why poll every minute instead of a webhook?**
Airtable's native trigger in n8n uses polling. The 1-minute interval provides near-real-time responsiveness without requiring webhook infrastructure setup on the Airtable side.

**Why check `Interview Scheduled` before creating a calendar event?**
Without this guard, every time the recruiter saves the record while it's in `Interview` stage, a new calendar event would be created. The boolean flag ensures exactly one invite per candidate.

**Why store resumes in Google Drive instead of Airtable attachments?**
Drive provides a persistent, organized, shareable location for resumes that the team can access directly. Airtable attachment limits and file management are avoided entirely.
