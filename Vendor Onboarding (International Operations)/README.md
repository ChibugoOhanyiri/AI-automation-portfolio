# 🌍 International Ops — Vendor Onboarding Automation
<img width="1126" height="695" alt="Screenshot 2026-04-14 140648" src="https://github.com/user-attachments/assets/106e5e39-7198-413d-8d44-9a024c3efa22" />


A production-grade n8n workflow that automates the full vendor onboarding lifecycle for international operations teams. From form submission to invoice delivery, and headcount tracking.


---

## What This Solves

Most ops teams manage vendor onboarding manually. A vendor submits their details, and someone copies them into a database. Someone else emails an acknowledgment, and a reviewer checks the details and replies to approve or reject. If approved, someone generates an invoice. If rejected, someone writes a rejection email explaining why.

Each of those steps takes time, introduces errors, and requires someone to be online and paying attention.

This workflow replaces the entire process with two connected automations that run without human intervention after the initial form submission and a single reviewer click.

---

## Workflow Architecture

The system is split into two workflows that hand off to each other via webhook.

```
WORKFLOW 1 — Vendor Submission & Review Routing
────────────────────────────────────────────────
Vendor fills onboarding form
        ↓
Standardize & validate data
        ↓
Create vendor record in Notion DB (Status: Pending Review)
        ↓
Send acknowledgement email to vendor
        ↓
Send review email to ops reviewer
(email contains Approve button → Workflow 2 webhook)
(email contains Reject button → Tally rejection form → Workflow 2 webhook)


WORKFLOW 2A — Approval Path
────────────────────────────
Reviewer clicks Approve in email
        ↓
Webhook receives decision + all vendor data via URL params
        ↓
Respond to webhook (instant browser response)
        ↓
Extract & clean data from webhook payload
        ↓
Update vendor status in Notion DB → Approved
        ↓
IF vendor approved? → TRUE
        ↓
Build Coupa-ready HTML invoice (Code node)
        ↓
Convert invoice HTML to PDF (html2pdf API)
        ↓
Send invoice PDF to vendor via Gmail  ──→  Upload PDF to Google Drive
        ↓
End


WORKFLOW 2B — Rejection Path
─────────────────────────────
The reviewer clicks Reject in the email
        ↓
Opens Tally form (pre-filled with vendor details via URL params)
Reviewer selects rejection reason from dropdown
        ↓
Tally fires POST to rejection webhook
        ↓
Respond to webhook
        ↓
Extract vendor data + resolve rejection reason UUID → human-readable text
        ↓
Generate reason-specific rejection email message (Code node)
        ↓
Update vendor status in Notion DB → Rejected
        ↓
Send rejection email to vendor with specific reason
        ↓
End
```

---

## Tools & Integrations

| Tool | Role |
|---|---|
| **n8n** | Workflow automation engine |
| **Tally** | Vendor onboarding form + rejection reason form |
| **Notion** | Vendor database (persistent record of all vendors) |
| **Gmail** | Vendor acknowledgment, reviewer notification, invoice delivery, rejection emails |
| **Google Drive** | Invoice PDF storage |
| **html2pdf.app** | HTML to PDF conversion for invoice generation |
| **Google Sheets** | Headcount tracker (updated on employee onboarding) |

---

## Workflow 1 — Node-by-Node Breakdown

### 1. Vendor Onboarding Form (Tally Trigger)
Triggers when a vendor submits the onboarding form. All fields are required except Additional Notes. Form-level validation prevents submission with missing data.

**Fields collected:** Vendor name, country, business type, tax ID, registration number, contact details, banking details (bank name, account name, IBAN, SWIFT/BIC), service description, monthly invoice amount, payment terms, NDA status.

### 2. Standardize Data (Edit Fields)
Cleans and normalizes incoming form data before it touches any downstream system.

- Formats date to `YYYY-MM-DD` by splitting on `T`: `{{ $json.submittedAt.split("T")[0] }}`
- Ensures phone numbers are strings: `{{ String($json['Primary Contact Phone']) }}`
- Strips unnecessary whitespace from field names where Tally appends trailing spaces

### 3. Update Row in DB (Notion)
Creates a new database page in the Notion Vendor Database with all vendor properties mapped. Status is hardcoded to `Pending Review` at creation — it only changes when the reviewer acts.

**Key property mappings:**
- NDA on File → Checkbox: `{{ $json['Do you have a signed NDA on file?'] === 'Yes' }}`
- Status → Select: `Pending Review` (hardcoded)
- Date Submitted → Date (no time): formatted date string

### 4. Configure Reviewer's Details and Get Vendor Row ID (Set Node)
Acts as a config layer. Stores the reviewer's name and email as fixed variables so they can be updated in one place without touching email templates. Also captures the Notion page ID (`$json.id`) from the previous node output as `Vendor row id` — this ID is passed through every downstream step to ensure the correct Notion record gets updated.

```
Reviewer Name  → "Your Name Here."
Reviewer Email → "your-email@domain.com"
Vendor row id  → {{ $json.id }}
```

### 5. Acknowledge Vendors Form (Gmail)
Sends a branded HTML confirmation email to the vendor. Confirms receipt, shows a submission summary table, and sets status pill to "Pending Review". References data from the Standardize Data node.

### 6. Send Email to Reviewer (Gmail)
Sends a branded HTML review notification to the ops reviewer. Contains full vendor details table and two action buttons:

**Approve button** — GET request to Workflow 2 webhook with all vendor data encoded as URL params:
```
https://your-n8n-instance/webhook/[WEBHOOK-ID]?status=Approved
  &vendorId=[Notion page ID]
  &vendorName=[vendor name]
  &vendorEmail=[vendor email]
  &contactName=[contact name]
  &currency=[currency]
  &amount=[invoice amount]
  &country=[country]
  &paymentTerms=[payment terms]
  &taxId=[tax ID]
  &bankName=[bank name]
  &accountName=[account name]
  &iban=[IBAN]
  &swift=[SWIFT code]
```

**Reject button** — Opens Tally rejection form pre-filled with vendor details via URL params. Reviewer selects rejection reason and submits. Tally fires POST to Workflow 2 rejection webhook.

---

## Workflow 2 — Approval Path Node Breakdown

### 1. Listen for Vendor's Approval (Webhook)
- Method: GET
- Receives all vendor data as query parameters from the Approve button click
- **⚠️ Before going live:** Switch from Test URL to Production URL (see Setup section)

### 2. Respond to Webhook
Immediately returns `200 OK` so the reviewer's browser doesn't hang. Set to respond immediately, not wait for workflow completion.

### 3. Get Valid Data (Edit Fields)
Extracts query parameters from `$json.query.*` into clean top-level fields for use by downstream nodes.

### 4. Update Vendor Status in DB to Approved (Notion)
- Action: Update a database page
- Page ID: `{{ $json.vendorId }}`
- Status property: `{{ $json.status }}` → resolves to `Approved`

### 5. Was Vendor Approved? (IF Node)
- Condition: `{{ $json.status }}` equals `Approved`
- True → invoice path
- False → (not used in this workflow — rejection is handled by separate webhook)

### 6. Build Invoice HTML Template (Code Node)
Generates a fully formatted Coupa-ready HTML invoice using vendor data from the webhook payload. Produces:
- Auto-generated invoice number: `INV-[YEAR]-[5-digit random]`
- Issue date (today) and due date (today + 30 days)
- Vendor details (from, bill to)
- Line items table
- Banking details
- Returns `invoiceHtml`, `invoiceNumber`, `vendorEmail`, `subject`

### 7. Convert HTML to PDF (HTTP Request)
- URL: `https://api.html2pdf.app/v1/generate`
- Method: POST
- Converts invoice HTML to binary PDF

### 8. Send Invoice to Vendor (Gmail)
- Sends approval email with PDF attached
- Binary field name: `data`
- Pulls vendor details from `Was vendor approved?` node
- Pulls invoice number from `Build invoice HTML template` node

### 9. Upload Invoice in Drive (Google Drive)
- Uploads PDF binary to `Vendor Invoices` folder
- File named: `[invoiceNumber]_[vendorName]`
- Runs in parallel with Gmail send

---

## Workflow 2 — Rejection Path Node Breakdown

### 1. Listen for Rejection (Webhook)
- Method: POST
- Receives Tally form submission with hidden fields (vendor data) + rejection reason dropdown
- **⚠️ Before going live:** Switch from Test URL to Production URL (see Setup section)

### 2. Respond (Respond to Webhook)
Immediately returns `200 OK`.

### 3. Get Data (Edit Fields)
Extracts fields from Tally's nested JSON structure:
```javascript
// Hidden fields
$json.body.data.fields.find(f => f.label === 'vendorId').value
$json.body.data.fields.find(f => f.label === 'vendorName').value
$json.body.data.fields.find(f => f.label === 'vendorEmail').value
$json.body.data.fields.find(f => f.label === 'contactName').value

// Rejection reason — resolves UUID to text
$json.body.data.fields
  .find(f => f.label === 'Rejection Reason')
  .options
  .find(o => o.id === $json.body.data.fields
    .find(f => f.label === 'Rejection Reason').value[0])
  .text
```

### 4. Get Rejection Message (Code Node)
Maps rejection reason text to a pre-written, professional email message. Supports five reasons:
- Incomplete documentation
- Failed compliance check
- Duplicate vendor record
- Outside service scope
- NDA not on file

Returns `rejectionMessage` and `rejectionReason` for use in the email template.

### 5. Update Vendor Status in DB to Rejected (Notion)
- Page ID: `{{ $json.vendorId }}`
- Status: `Rejected`

### 6. Send a Message (Gmail)
Sends branded rejection email to vendor. Dynamically inserts the reason-specific message from the Code node. Subject line varies for NDA rejections to flag action required.

---

## Setup Instructions

### Prerequisites
- n8n instance (cloud or self-hosted)
- Notion account with integration token
- Google account with Gmail and Drive OAuth2 configured
- Tally account (free tier is sufficient)
- html2pdf.app API key (free tier is sufficient for demos)

### Step 1 — Import the workflow
1. Download `International_Ops.json`
2. In n8n, go to Workflows → Import from file
3. Select the JSON file

### Step 2 — Configure credentials
Add the following credentials in n8n Settings → Credentials:
- **Notion API** — Internal integration token from notion.so/my-integrations
- **Gmail OAuth2** — Google OAuth2 with Gmail scope
- **Google Drive OAuth2** — Google OAuth2 with Drive scope
- **Google Sheets OAuth2** — Google OAuth2 with Sheets scope (for headcount workflow)

### Step 3 — Connect Notion integration to your database
1. Open your Notion Vendor Database
2. Click the three dots (…) → Connections → Connect to → select your n8n integration

### Step 4 — Update the Config node
Open the node named **"Configure reviewer's details and get vendor row ID from DB"** and update:
```
Reviewer Name  → your actual name
Reviewer Email → your actual email address
```

### Step 5 — Update webhook URLs in the reviewer email
Open the **"Send email to reviewer"** Gmail node. Find the Approve button URL and update the webhook ID:
```
# Replace [WEBHOOK-ID] with your actual webhook path
https://your-n8n-instance.app.n8n.cloud/webhook/[WEBHOOK-ID]
```

### Step 6 — Update Tally rejection form webhook
1. Go to your Tally rejection form → Integrations → Webhooks
2. Update the URL to your rejection workflow's production webhook URL

### Step 7 — Switch webhooks to production
**⚠️ Critical step before going live**

Both Workflow 2 webhooks default to Test URL during development. Before activating:

1. Open **"Listen for vendor's approval"** webhook node
2. Click the **Production URL** tab — copy this URL
3. Update the Approve button URL in your reviewer email template to use this production URL

4. Open **"Listen for rejection"** webhook node
5. Click the **Production URL** tab — copy this URL
6. Update the Tally webhook integration to use this production URL

7. Activate both workflows using the toggle in the top right of the canvas

Test URLs only work when you manually click "Listen for test event" — production URLs are always live once the workflow is activated.

### Step 8 — Connect Tally form to n8n
In your Tally onboarding form:
1. Go to Integrations → Webhooks
2. Connect to your n8n Tally trigger node
3. Ensure hidden field identifiers match exactly: `vendorId`, `vendorName`, `vendorEmail`, `contactName`

---

## Notion Database Schema

The Vendor Database requires these properties:

| Property | Type |
|---|---|
| Vendor Name | Title |
| Country | Select |
| Business Type | Select |
| Tax ID / VAT Number | Rich Text |
| Business Registration Number | Number |
| Contact Name | Rich Text |
| Contact Email | Email |
| Contact Phone | Phone |
| Business Website | URL |
| Bank Name | Rich Text |
| Account Name | Rich Text |
| Account Number / IBAN | Number |
| SWIFT / BIC Code | Rich Text |
| Currency | Select |
| Service Description | Rich Text |
| Monthly Invoice Amount | Number |
| Payment Terms | Select |
| NDA on File | Checkbox |
| Status | Select (Pending Review / Approved / Rejected) |
| Date Submitted | Date |

---

## Production Considerations

This workflow was built as a functional demo. For production deployment, the following additions are recommended:

**Error handling** — n8n supports a dedicated Error Workflow that catches failures from any node, logs which step broke, and sends an alert email. Enable this in Workflow Settings → Error Workflow.

**Retry logic** — Enable retries on the Notion, Gmail, and Google Drive nodes (3 retries at 30-second intervals) to handle transient API failures gracefully.

**Signature collection** — The invoice delivery step currently sends a PDF for manual signature return. In production, replace the Gmail attachment step with a DocuSign or PandaDoc node for e-signature collection and automated return handling.

**Coupa integration** — The invoice is formatted to Coupa's standard invoice fields. In a live environment with Coupa credentials, the Google Drive upload step can be replaced or supplemented with a direct Coupa API upload.

---

## File Structure

```
├── International_Ops.json          # n8n workflow export (import this)
├── README.md                       # This file
├── templates/
│   ├── email_vendor_acknowledgement.html
│   ├── email_reviewer_notification.html
│   ├── email_approval_invoice.html
│   └── email_rejection.html
├── assets/
│   └── invoice_template.docx       # Google Docs invoice template with placeholders
└── data/
    └── vendor_database_sample.csv  # Sample data for Notion DB import
```

---

## Built With

- [n8n](https://n8n.io) — Workflow automation
- [Notion API](https://developers.notion.com) — Database management
- [Tally](https://tally.so) — Form collection
- [html2pdf.app](https://html2pdf.app) — PDF generation
- [Google Workspace](https://workspace.google.com) — Gmail, Drive, Sheets

---

## Author

Built by Chibugo Ohanyiri — automation engineer and ops systems builder.

[GitHub](https://github.com/yourusername) · [LinkedIn](https://linkedin.com/in/yourprofile) · [Twitter/X](https://x.com/yourhandle)
