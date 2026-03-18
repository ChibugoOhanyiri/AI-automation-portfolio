# Automated Job Application Screening System — Zapier

A two-Zap automated hiring pipeline that screens, routes, and summarises job applications in real time. Built for a technical hiring team receiving 20–50 applications weekly. HR no longer reads applications that don't qualify; the system handles it automatically.

---

## The Problem

HR teams at growing companies spend hours every week manually reviewing applications from candidates who don't meet basic requirements,  wrong skills, insufficient experience, or unsupported locations. Every unqualified application read is time stolen from the ones that actually matter.

---

## What It Does

**Zap 1 — Real-time screening and routing**

The moment a candidate submits an application via Google Forms:

1. The system cleans the data. People type experience in every format — "5+ years", "3-5 years", "2yrs". The formatter strips all of that to a clean number so the filter can actually compare it.

2. Three qualification questions run automatically:
   - Do they have Python?
   - Is their experience at least 1 year?
   - Are they in a supported location?

   If any answer is no — the application stops there. No email sent, no row created, no Slack notification. The unqualified application simply disappears from the process.

3. For everyone who passes:
   - A 30-minute delay runs (buffer for any urgent manual HR review)
   - The hiring manager gets an email with a clean applicant summary — name, email, experience, skills, location, and portfolio link
   - The applicant's details are appended to the daily digest

4. Routing by seniority via Paths:
   - **Senior (4+ years)** → Slack #hiring-senior channel — tech lead gets notified immediately to schedule an interview
   - **Junior (1–3 years)** → Google Sheets — added to a batch review spreadsheet
   - **Other qualified** → Google Sheets — separate tracking row

**Zap 2 — Daily 6pm summary**

A completely separate Zap runs every day at 6pm. It releases the digest — one clean Slack message listing every qualified applicant that came through that day. The team ends their day with full visibility of what arrived, without having manually checked a single application.

---

## Architecture

### Zap 1 — Screening Pipeline (14 steps)

```
Google Forms — New Form Response
        │
        ▼
Formatter — Clean and Transform Data
(Proper case names, trim whitespace, standardise fields)
        │
        ▼
Formatter — Extract Numeric Experience Years
(Strips "years", "yrs", "+", handles ranges like "3-5" → clean integer)
        │
        ▼
Filter — Initial Qualification Gate
(Python skill required + experience ≥ 1 year + supported location)
        │
     [PASS]
        │
        ▼
Delay — 30 minutes (manual HR buffer)
        │
        ▼
Gmail — Send Email to Sales/Hiring Manager
(HTML email with applicant summary, filter pass confirmation, Google Sheet link)
        │
        ▼
Digest — Append Entry to Daily Digest
(One line per qualified applicant — feeds Zap 2)
        │
        ▼
Paths — Branch by Seniority
   ┌────────────────┬─────────────────┐
   ▼                ▼                 ▼
Path A           Path B            Path C
Senior (4+ yrs)  Junior (1-3 yrs)  Other qualified
   │                │                 │
   ▼                ▼                 ▼
Slack            Google Sheets     Google Sheets
#hiring-senior   Batch review      Tracking row
```

### Zap 2 — Daily Summary (3 steps)

```
Schedule — Every Day at 6:00 PM
        │
        ▼
Digest — Release Daily Digest
(Pulls all entries appended by Zap 1 throughout the day)
        │
        ▼
Slack — Send Channel Message
(One clean summary message with all qualified applicants)
```

---

## The Digest Bridge

The two Zaps never directly communicate. The **Digest by Zapier** tool is the bridge:

- Zap 1 appends one line per qualified applicant to a named digest throughout the day
- At 6pm, Zap 2 releases that digest as a single Slack message and resets it for tomorrow
- Same digest name = same data bucket

This means real-time routing and daily summarisation run independently without any dependency between the two Zaps.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Workflow automation | Zapier |
| Application intake | Google Forms |
| Data cleaning | Formatter by Zapier |
| Qualification filter | Filter by Zapier |
| Delay buffer | Delay by Zapier |
| Daily digest | Digest by Zapier |
| Path routing | Paths by Zapier |
| Senior alert | Slack |
| Junior tracking | Google Sheets |
| Manager notification | Gmail (HTML email) |
| Daily summary | Slack |

---

## Email Output

The hiring manager receives a formatted HTML email for every qualified applicant containing:

- Applicant name, email, experience, skills, location, and portfolio link
- Green confirmation badge: **✓ Passed: Python skill + 1+ year experience + supported location**
- Direct link to view full details in Google Sheet

---

## Slack Output

**Real-time (senior applicants):**
```
New Senior App: [Name] ([email])
Experience: X years
Skills: Python, SQL, DevOps (Docker/Kubernetes)
Location: Fully Remote
```

**Daily 6pm summary:**
```
📊 Daily Qualified Job Applications Summary – [date]
[Applicant 1 line]
[Applicant 2 line]
...
Total today: X applicants passed screening.
Senior apps already pinged tech lead. Junior apps are in the Google Sheet for batch review.
```

---

## Key Design Decisions

**Data cleaning before filtering** — experience values arrive in unpredictable formats. Cleaning happens first so the filter always receives a clean integer. Without this, "5+ years" would fail a numeric comparison and qualified candidates would be incorrectly rejected.

**Filter as a hard gate** — Zapier's Filter stops the entire Zap for unqualified applications. No downstream steps run, no tasks are consumed on those records. This keeps task usage lean and ensures HR receives zero noise.

**30-minute delay before email** — gives HR a buffer to manually intervene on anything urgent before the automated email fires. Doesn't block Slack routing for seniors.

**Two independent Zaps via Digest** — rather than building a complex single Zap with time-based branching, the digest pattern keeps each Zap simple and single-purpose. Zap 1 screens. Zap 2 summarises. Clean separation of concerns.

**Paths for seniority routing** — seniors and juniors need different handling (immediate interview vs batch review). Paths handles this without creating duplicate Zaps or complex filter chains.

---

## Screenshots

| Screenshot | Description |
|---|---|
|<img width="1002" height="1484" alt="Screenshot 2026-03-15 154146" src="https://github.com/user-attachments/assets/317728a7-ca0c-4052-aaa6-7b9fbce7bfce" />
| Full Zap 1 canvas — 14-step screening pipeline |
|<img width="1919" height="824" alt="Screenshot 2026-03-15 155401" src="https://github.com/user-attachments/assets/9a1ae4e9-5551-4cc0-8e0c-28db037b27c3" />
| Live Slack notification in #hiring-senior channel |
|<img width="799" height="720" alt="Screenshot 2026-03-15 155933" src="https://github.com/user-attachments/assets/5e268816-d960-4589-ae4e-220984a49d21" />
| Zap 2 canvas — daily digest summary |
| <img width="639" height="657" alt="Screenshot 2026-03-15 161555" src="https://github.com/user-attachments/assets/87763873-6f53-44f7-a92f-1ef16abf31f5" />
| HTML email sent to hiring manager |

---

