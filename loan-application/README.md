# Loan Application Email Notification System

An automated email workflow that handles the full communication cycle when a borrower applies for a loan through a lending directory. The system notifies the matched lender, acknowledges the borrower, and follows up automatically after a set waiting period.

---

## What It Does

When a borrower submits a loan application via webhook, the system:

1. Pulls the borrower's application data
2. Queries the lender database to find the matched lender's email
3. Sends a notification email to the lender with the borrower's details
4. Simultaneously sends a confirmation email to the borrower acknowledging receipt
5. Waits a defined period (e.g. 3 days)
6. Sends a follow-up email to the borrower checking if the lender has been in contact

All three emails are sent automatically with no human intervention.

---

## Architecture

```
Webhook Trigger (loan application submitted)
        │
        ▼
Extract borrower data from payload
        │
        ▼
Query lender database (MySQL) — get lender email
        │
        ▼
Merge borrower + lender data
        │
        ▼
Route by email type
   ┌────┼────────────┐
   ▼    ▼            ▼
Lender  Borrower   Borrower
notify  confirm    follow-up
(now)   (now)      (after wait)
        │
        ▼
     Wait node (e.g. 3 days)
        │
        ▼
  Borrower follow-up email
```

---

## Tech Stack

| Layer | Tool |
|---|---|
| Workflow automation | n8n |
| Trigger | Webhook |
| Lender data | MySQL database |
| Email delivery | SendGrid |

---

## Email Types

### 1. Lender Notification (sent immediately)
Notifies the lender that a borrower has expressed interest after finding them in the directory.

**Subject:** You've received a loan request from a user who found you on our platform

**Contains:**
- Borrower name
- Loan amount and currency
- Loan purpose
- Tenor (duration and period)

### 2. Borrower Confirmation (sent immediately)
Acknowledges the borrower's application and sets expectations.

**Subject:** We got your loan request!

**Contains:**
- Confirmation that the request was received
- Next steps / what to expect

### 3. Borrower Follow-up (sent after wait period)
Checks in with the borrower after the defined waiting period to confirm whether the lender has responded.

**Subject:** Checking in on your loan application

**Contains:**
- Status check on lender contact
- Support contact if needed

---

## Data Fields

The workflow extracts these fields from the incoming webhook payload:

| Field | Description |
|---|---|
| `borrower_name` | Full name of the applicant |
| `currency` | Loan currency (e.g. NGN, USD) |
| `loan_amount` | Requested amount |
| `loan_purpose` | Purpose of the loan |
| `tenor` | Loan duration (number) |
| `tenor_period` | Duration unit (months, years) |
| `borrower_email` | Borrower's email for confirmations |

---

## Setup

### Prerequisites
- n8n instance
- MySQL database with lender records including email addresses
- SendGrid account with a verified sender domain
- Webhook endpoint configured in your loan application form/platform

### Credentials to configure in n8n
```
- MySQL (host, database, username, password)
- SendGrid API key
```

### Configuration
1. Update the Webhook node path to match your application's endpoint
2. Configure the MySQL node with your lender database connection
3. Update SendGrid nodes with your verified sender email
4. Adjust the Wait node duration to match your follow-up policy (default: 3 days)
5. Customise email body content per your platform's tone and branding

### Webhook Payload Format
The trigger expects a JSON payload with at minimum:
```json
{
  "borrower_name": "string",
  "borrower_email": "string",
  "loan_amount": number,
  "currency": "string",
  "loan_purpose": "string",
  "tenor": number,
  "tenor_period": "string",
  "lender_id": "string or number"
}
```

---

## Key Design Decisions

**MySQL lender lookup** — lender contact details are pulled live from the database at runtime rather than embedded in the payload. This means lender email changes in the database are reflected automatically without workflow changes.

**Wait node for follow-up** — n8n's built-in Wait node pauses the workflow execution for the defined period before sending the follow-up. No separate scheduled workflow or cron job needed — the follow-up is tied to the specific application that triggered it.

**Three parallel email types** — lender notification and borrower confirmation fire at the same time immediately. The follow-up is a separate delayed step. This keeps the communication timeline clean and predictable.

---

## Use Case Context

Built for a lending directory platform where borrowers browse and apply to lenders listed on the platform. When a borrower applies, the matched lender needs to be notified immediately, the borrower needs acknowledgement, and if no contact is made within a few days, the borrower needs a follow-up to confirm the process is working.

---
