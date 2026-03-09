
# Loan Eligibility & Offer Management System (n8n)

<img width="1568" height="612" alt="image" src="https://github.com/user-attachments/assets/2fb6025f-e713-49d2-beaf-0966974296e7" />


A fully automated loan processing workflow built in n8n. From application intake to offer response handling and follow-up sequences.

---

## What It Does

Most loan processing workflows require staff to manually review applications, send emails, chase responses, and update spreadsheets. This workflow automates the entire post-application pipeline — eligibility assessment, interest calculation, offer delivery, response tracking, and follow-ups — so the team only needs to act when a loan is confirmed accepted.

---

## Features

- **Application intake via n8n form** — captures applicant name, email, phone, address, salary range, loan amount, loan tenor, and reason for loan
- **Auto-generated applicant ID** — assigns a unique `APP-{timestamp}` ID to every applicant on submission
- **Universal acknowledgement email** — every applicant receives a confirmation email immediately, regardless of eligibility outcome
- **Salary-based eligibility routing** — checks eligibility against three salary bands and routes accordingly
- **Automatic interest & repayment calculation** — calculates interest rate, total payable, and monthly repayment based on loan amount and tenor
- **Structured offer emails** — eligible applicants receive a detailed offer email with **Accept** and **Decline** buttons
- **Response listening** — the workflow listens for button clicks and routes accepted/declined responses into separate flows
- **Acknowledgement on response** — applicants receive a confirmation email whether they accept or decline
- **Google Sheets CRM** — every applicant and their status is logged and updated in real time throughout the workflow
- **Automated follow-up system** — a scheduled trigger checks for non-responders and sends up to **2 follow-up emails** before expiring the offer
- **Offer expiry** — after two unanswered follow-ups, the offer is marked expired, and no further emails are sent

---

## Workflow Architecture

The workflow is broken into five logical sections:

### 1. Loan Request Intake & Acknowledgement
```
On form submission
  → Concatenate name + assign Applicant ID (APP-{timestamp})
  → Append row to Google Sheets (Loan eligibility sheet)
  → Send welcome/acknowledgement email to applicant
```

**Form fields collected:**
- First Name, Last Name, Email, Phone Number, Address
- Salary Range (dropdown: `500k and below` / `>500k to 1 million` / `>1 million`)
- How much do you need?, Reason for loan

### 2. Eligibility Routing
```
Check eligibility based on salary range
  ├── Ineligible → Send rejection email → Update status to "Ineligible"
  └── Eligible   → Preserve loan details → Calculate interest → Send offer email
```

Eligibility is determined by salary band. Ineligible applicants are notified immediately, and their status is updated in Sheets.

### 3. Calculate Interest & Send Eligibility Mail
```
Calculate interest
  ├── Loan > 500k to 1 million → Apply rate → Update info → Send loan offer email
  └── Loan <= 1 million        → Apply rate → Update sheet → Send loan offer email
```

Interest, total payable, and monthly repayment are calculated automatically based on the loan amount and tenor before the offer email is sent.

### 4. Offer Response System
```
Get loan offer response (webhook — button click)
  → If the loan offer was accepted?
      ├── YES → Get accepted response + applicant ID
      │          → Update response status to "Accepted."
      │          → Send offer accepted response
      │          → Find applicant → Send accepted acknowledgement email
      └── NO  → Get declined response
                 → Update response status to "Declined."
                 → Send offer declined response
                 → Find applicant → Send declined acknowledgement email
```

### 5. Follow-Up System
```
Schedule Trigger (daily) / Manual trigger
  → Get all rows from the sheet
  → Filter: who hasn't responded?
  → Switch (by salary band)
      └── If loan submitted >= 3 days ago AND follow_up_count = 0
              → Send follow-up email → Update follow_up_count to 1
          If follow_up_count = 1 AND response_status = "No Response"
              → Send second follow-up → Update follow_up_count to 2 (offer expires)
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| **n8n** | Workflow automation engine |
| **n8n Form Trigger** | Loan application intake |
| **Gmail (via n8n)** | All outbound emails |
| **Google Sheets** | CRM — applicant data, status tracking, follow-up counts |
| **Webhook node** | Listens for Accept/Decline button clicks from offer emails |
| **Set node** | Data transformation, ID generation, interest calculations |
| **IF / Switch nodes** | Eligibility routing and follow-up logic |
| **Schedule Trigger** | Runs the follow-up system daily |

---

## Google Sheets CRM Structure

| Column | Description |
|---|---|
| `applicant_id` | Unique ID — primary key (`APP-{timestamp}`) |
| `name` | Full applicant name |
| `email` | Applicant email |
| `phone` | Phone number |
| `salary` | Salary range band |
| `loan request submitted at` | Timestamp of form submission |
| `loan amount requested` | Loan amount in numbers |
| `loan_term_months` | Loan tenor in months |
| `reason for loan` | Applicant's stated reason |
| `eligibility status` | `Eligible` / `Ineligible` |
| `interest rate` | Calculated interest rate |
| `total payable` | Total repayment amount |
| `monthly_repayment` | Monthly instalment |
| `response status` | `No Response` / `Accepted` / `Declined` / `Expired` |
| `follow_up_count` | Number of follow-up emails sent (0, 1, or 2) |

---

## End-to-End Flow

```
Applicant submits form
        │
        ▼
Assign ID → Log to Sheets → Send acknowledgement email
        │
        ▼
Check eligibility (salary band)
   ├── Ineligible → Send rejection → Done
   └── Eligible   → Calculate interest & repayment
                         │
                         ▼
              Send offer email (Accept / Decline buttons)
                         │
              ┌──────────┴──────────┐
           Accepted              Declined
              │                     │
     Update Sheets            Update Sheets
     Send acceptance          Send decline
     acknowledgement          acknowledgement
              │
           [DONE]

     If no response after 3+ days:
        Follow-up #1 → wait → Follow-up #2 → Offer expires
```

---

## Setup Instructions

### Prerequisites
- Active n8n instance (cloud or self-hosted)
- Gmail account connected via n8n OAuth2 credentials
- Google Sheets with the columns listed above
- Google Sheets OAuth2 credentials configured in n8n

### Steps

1. **Import the workflow** — upload `Loan_eligibility_workflow__1_.json` into n8n via *Import from file*
2. **Connect Gmail** — replace Gmail credentials in all email nodes with your own
3. **Connect Google Sheets** — update the `documentId` and `sheetName` values in all Google Sheets nodes to point to your spreadsheet
4. **Configure the webhook URL** — copy the webhook URL from the *Get loan offer response* node and embed it as the Accept/Decline button links in your offer email template
5. **Set eligibility rules** — update the IF node conditions to match your actual eligibility criteria
6. **Set interest rate logic** — update the Set/Code nodes with your actual interest rate formulas
7. **Activate the Schedule Trigger** — set the desired frequency for the follow-up checker (daily recommended)
8. **Test end-to-end** — submit a test application and walk through the full flow

---

## Design Decisions

**Why auto-assign an Applicant ID?**
Using `APP-{timestamp}` as the primary key means every row in Google Sheets is uniquely identifiable without needing a database. The webhook response from offer buttons carries this ID back, allowing the workflow to look up and update the correct applicant record.

**Why send acknowledgement emails to everyone including ineligibles?**
It's better UX and also ensures no applicant is left wondering if their submission went through. Ineligible applicants are informed promptly rather than being ghosted.

**Why two follow-ups before expiry?**
Balances persistence with respect for the applicant. One follow-up could be missed; three starts to feel like spam. Two follow-ups with a clear expiry creates urgency without being aggressive.

**Why Google Sheets and not a database?**
Zero infrastructure overhead. For a small-to-medium lending operation, Sheets provides a real-time dashboard the team already understands, with no database admin required.
