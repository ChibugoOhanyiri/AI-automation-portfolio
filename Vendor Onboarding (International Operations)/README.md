# 🌍 International Ops — Vendor Onboarding & Headcount Automation

<!-- SCREENSHOT: Full workflow canvas showing all three workflows side by side -->
![Workflow Overview](https://github.com/user-attachments/assets/106e5e39-7198-413d-8d44-9a024c3efa22)

A production-grade n8n workflow that automates the full vendor onboarding lifecycle and employee headcount tracking for international operations teams. From form submission to invoice delivery, rejection handling, and real-time headcount reporting — every step runs automatically.

---

## What This Solves

Most ops teams manage vendor onboarding manually. A vendor submits their details, and someone copies them into a database. Someone else emails an acknowledgment, and a reviewer checks the details and replies to approve or reject. If approved, someone generates an invoice. If rejected, someone writes a rejection email explaining why. Meanwhile, headcount data sits in a spreadsheet that nobody remembers to update until month-end.

Each of those steps takes time, introduces errors, and requires someone to be online and paying attention.

This system replaces the entire process with three connected automations that run without human intervention after the initial trigger.

---

## Workflow Architecture

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
Reviewer clicks Reject in email
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


WORKFLOW 3 — Employee Onboarding & Headcount Tracking
──────────────────────────────────────────────────────
Employee fills onboarding form
        ↓
Standardize data
        ↓
Append new row to Google Sheets Headcount Tracker
        ↓
End (Dashboard updates automatically via formulas)
```

---

## Tools & Integrations

| Tool | Role |
|---|---|
| **n8n** | Workflow automation engine |
| **Tally** | Vendor onboarding form + rejection reason form + employee onboarding form |
| **Notion** | Vendor database (persistent record of all vendors) |
| **Gmail** | Vendor acknowledgment, reviewer notification, invoice delivery, rejection emails |
| **Google Drive** | Invoice PDF storage |
| **Google Sheets** | Headcount tracker — Dashboard, detail view, and monthly trend |
| **html2pdf.app** | HTML to PDF conversion for invoice generation |

---

## Screenshots

### Workflow 1 — Vendor Submission & Review Routing
<!-- SCREENSHOT: Workflow 1 canvas — form trigger through to reviewer email -->
<img width="633" height="129" alt="Screenshot 2026-04-17 162215" src="https://github.com/user-attachments/assets/8a7ccbb7-a6c1-4fe1-bbe8-d756f2ab143d" />


### Workflow 2 — Approval & Rejection Paths
<!-- SCREENSHOT: Workflow 2 canvas showing the IF node split, invoice path top, rejection path bottom -->
<img width="872" height="293" alt="Screenshot 2026-04-17 162259" src="https://github.com/user-attachments/assets/23b2e0a7-fcb4-4227-96a4-6161aff4fd19" />


### Workflow 3 — Employee Onboarding & Headcount Tracking
<!-- SCREENSHOT: Workflow 3 canvas — form trigger, standardize, Google Sheets append -->
<img width="388" height="145" alt="image" src="https://github.com/user-attachments/assets/0662e1b2-cb3c-41d5-86e8-c0b011594156" />



---

### Tally Rejection Reason Form
<!-- SCREENSHOT: Tally rejection form showing only the dropdown visible, hidden fields not shown -->

<img width="1840" height="798" alt="image" src="https://github.com/user-attachments/assets/b232a614-2f1c-4ce8-b4d5-cc7a905905a3" />


---

### Notion Vendor Database
<!-- SCREENSHOT: Notion Vendor Database table view showing columns and a few rows with Pending/Approved/Rejected status pills -->
<img width="1879" height="804" alt="image" src="https://github.com/user-attachments/assets/678165c3-a639-4b2d-904c-88616cc0f261" />


---

### Email — Vendor Acknowledgement
<!-- SCREENSHOT: Rendered vendor acknowledgement email showing the submission summary table and Pending Review status pill -->
<img width="347" height="534" alt="image" src="https://github.com/user-attachments/assets/d0549e04-e677-494e-a380-b9c56ee5f0c9" />

### Email — Reviewer Notification with Action Buttons
<!-- SCREENSHOT: Rendered reviewer email showing vendor details table and the green Approve / red Reject buttons -->
<img width="336" height="603" alt="image" src="https://github.com/user-attachments/assets/0563e14a-0765-406f-9593-fe604ad80f40" />


### Email — Approval with Invoice Attached
<!-- SCREENSHOT: Rendered approval email showing the Next Steps box and PDF attachment visible -->
<img width="1486" height="684" alt="image" src="https://github.com/user-attachments/assets/e1c992fd-4988-4b8f-bfe0-6e0a394929e0" />

### Email — Rejection with Reason
<!-- SCREENSHOT: Rendered rejection email showing the red decision box with reason and the reason-specific message body -->
<img width="303" height="410" alt="image" src="https://github.com/user-attachments/assets/0ad80e1c-dd9f-4cf7-9425-94af04780cdf" />


---

### Invoice PDF (Coupa-Ready)
<!-- SCREENSHOT: Generated invoice PDF showing header, line items table, banking details, and signature section -->
<img width="385" height="429" alt="image" src="https://github.com/user-attachments/assets/da727fae-dc11-4f6d-9ef7-8b5cd7bcb241" />


---

### Headcount Tracker — Dashboard
<!-- SCREENSHOT: Google Sheets Dashboard tab showing KPI cards, country breakdown table, and status split -->
<img width="1211" height="780" alt="image" src="https://github.com/user-attachments/assets/0aad4e75-0f8b-41c0-836a-ed2c281dcc5b" />


### Headcount Tracker — Detail View
<!-- SCREENSHOT: Google Sheets Headcount tab showing employee rows with colour-coded status and MoM change columns -->
<img width="1097" height="692" alt="image" src="https://github.com/user-attachments/assets/1b33f7b3-3608-401d-b2b1-0d897b72685b" />


### Headcount Tracker — Monthly Trend
<!-- SCREENSHOT: Google Sheets Monthly Trend tab showing the stacked bar chart and 7-month data table -->
<img width="1052" height="709" alt="image" src="https://github.com/user-attachments/assets/6ee288f2-cd92-4d06-9bee-b7e3d78b0df0" />


---

## Workflow 1 — Node-by-Node Breakdown

### 1. Vendor Onboarding Form (Tally Trigger)
Triggers when a vendor submits the onboarding form. All fields are required except Additional Notes. Form-level validation prevents submission with missing data.

**Fields collected:** Vendor name, country, business type, tax ID, registration number, contact details, banking details (bank name, account name, IBAN, SWIFT/BIC), service description, monthly invoice amount, payment terms, NDA status.

### 2. Standardize Data (Edit Fields)
Cleans and normalises incoming form data before it touches any downstream system.

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
Reviewer Name  → "Your Name Here"
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
- False → not used (rejection handled by separate webhook)

### 6. Build Invoice HTML Template (Code Node)
Generates a fully formatted Coupa-ready HTML invoice using vendor data from the webhook payload. Produces:
- Auto-generated invoice number: `INV-[YEAR]-[5-digit random]`
- Issue date (today) and due date (today + 30 days)
- Vendor details (from, bill to)
- Line items table with monthly retainer amount
- Banking details section
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
- Runs in parallel with Gmail send from the Convert to PDF node

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

// Rejection reason — resolves UUID to human-readable text
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

## Workflow 3 — Employee Onboarding & Headcount Tracking

### Overview
When a new employee or contractor completes their onboarding form, the workflow standardises the data and appends a new row to the Google Sheets Headcount Tracker. The Dashboard tab updates automatically through formulas — no manual data entry required.

### 1. Employee Onboarding Form (Tally Trigger)
Triggers on employee form submission.

**Fields collected:** Full name, country, employment type (Full-Time / Contractor / Part-Time), role/job title, start date, status.

### 2. Standardize Data (Edit Fields)
Normalises incoming data — formats dates, trims whitespace, sets default Status to `Active`.

### 3. Update Headcount Tracker (Google Sheets)
- Action: Append Row
- Sheet: `Headcount`
- Maps form fields to spreadsheet columns

| Spreadsheet Column | Source |
|---|---|
| Country | `{{ $json['Country'] }}` |
| Employee Name | `{{ $json['Full Name'] }}` |
| Headcount | `1` (hardcoded — each submission = 1 person) |
| Status | `Active` (hardcoded default) |
| MoM Change | `1` (hardcoded — new addition) |
| Start Date | `{{ $json['Start Date'] }}` |
| Role | `{{ $json['Role'] }}` |
| Employment Type | `{{ $json['Employment Type'] }}` |

---

## Headcount Tracker — Google Sheets Structure

The tracker has three tabs that work together as a live reporting system.

### Dashboard Tab
Formula-driven summary view. Updates automatically when new rows are added to the Headcount tab. Contains:
- KPI cards: Total Headcount, Active Employees, Net MoM Change, Countries
- Country breakdown table with current vs last month comparison
- Status breakdown (Active / On Leave / Offboarded / Contractor / Pending) with percentages

<!-- SCREENSHOT: Dashboard tab with KPI cards and country table -->
<img width="1211" height="780" alt="image" src="https://github.com/user-attachments/assets/0aad4e75-0f8b-41c0-836a-ed2c281dcc5b" />

### Headcount Tab
The live data layer. Every row is one employee or contractor. This is the only tab n8n writes to. Contains:
- Country, Employee Name, Headcount (always 1), Status (colour-coded), MoM Change, Start Date, Role, Employment Type

Status colours: Active (green) · On Leave (yellow) · Offboarded (red) · Contractor (blue) · Pending (grey)

### Monthly Trend Tab
Historical view updated manually once per month by copying that month's country totals. Contains a stacked bar chart showing headcount growth by country over 7 months.

> **How the formula chain works:** n8n appends to Headcount → Dashboard formulas like `=SUM(Headcount!D2:D20)` and `=COUNTIF(Headcount!E2:E20,"Active")` recalculate instantly → KPI cards and country totals update with no manual work.

---

## Setup Instructions

### Prerequisites
- n8n instance (cloud or self-hosted)
- Notion account with integration token
- Google account with Gmail, Drive, and Sheets OAuth2 configured
- Tally account (free tier is sufficient)
- html2pdf.app API key (free tier is sufficient for demos)

### Step 1 — Import the workflow
1. Download `International_Ops.json`
2. In n8n → Workflows → Import from file
3. Select the JSON file

### Step 2 — Configure credentials
Add the following credentials in n8n Settings → Credentials:
- **Notion API** — Internal integration token from notion.so/my-integrations
- **Gmail OAuth2** — Google OAuth2 with Gmail scope
- **Google Drive OAuth2** — Google OAuth2 with Drive scope
- **Google Sheets OAuth2** — Google OAuth2 with Sheets scope

### Step 3 — Connect Notion integration to your database
1. Open your Notion Vendor Database
2. Click the three dots (…) → Connections → Connect to → select your n8n integration

### Step 4 — Update the Config node
Open **"Configure reviewer's details and get vendor row ID from DB"** and update:
```
Reviewer Name  → your actual name
Reviewer Email → your actual email address
```

### Step 5 — Update webhook URLs in the reviewer email
Open the **"Send email to reviewer"** Gmail node. Find the Approve button URL and replace the placeholder:
```
https://your-n8n-instance.app.n8n.cloud/webhook/[WEBHOOK-ID]
```

### Step 6 — Update Tally rejection form webhook
1. Go to your Tally rejection form → Integrations → Webhooks
2. Update the URL to your rejection workflow's production webhook URL

### Step 7 — Connect Google Sheets
1. Create a new Google Sheet with three tabs: `Dashboard`, `Headcount`, `Monthly Trend`
2. Import the structure from `data/headcount_tracker_sample.xlsx`
3. In the **"Update Headcount Tracker"** Google Sheets node, update the Document ID to your sheet's ID

### Step 8 — Switch webhooks to production
**⚠️ Critical step before going live**

Both Workflow 2 webhooks default to Test URL during development. Before activating:

1. Open **"Listen for vendor's approval"** → click **Production URL** tab → copy the URL
2. Update the Approve button URL in the reviewer email template to use this production URL
3. Open **"Listen for rejection"** → click **Production URL** tab → copy the URL
4. Update the Tally webhook integration to use this production URL
5. Activate both workflows using the toggle top right of the canvas

> Test URLs only work when you manually click "Listen for test event". Production URLs are always live once the workflow is activated.

### Step 9 — Connect Tally forms to n8n
In your Tally onboarding form → Integrations → Webhooks → connect to your n8n Tally trigger node.

Ensure hidden field identifiers in the rejection form match exactly:
`vendorId` · `vendorName` · `vendorEmail` · `contactName`

---

## Notion Database Schema

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
| Status | Select — Pending Review / Approved / Rejected |
| Date Submitted | Date |

---

## Production Considerations

**Error handling** — n8n supports a dedicated Error Workflow that catches failures from any node, logs which step broke, and sends an alert email. Enable in Workflow Settings → Error Workflow.

**Retry logic** — Enable retries on Notion, Gmail, and Google Drive nodes (3 retries at 30-second intervals) to handle transient API failures.

**Signature collection** — The invoice delivery step sends a PDF for manual signature return. In production, replace with a DocuSign or PandaDoc node for e-signature and automated return handling.

**Coupa integration** — The invoice is formatted to Coupa's standard invoice fields. In a live environment with Coupa credentials, the Drive upload step can be supplemented with a direct Coupa API push.

**Headcount automation** — The Monthly Trend tab is currently updated manually once per month. This can be automated with an additional n8n workflow triggered by a schedule node on the last working day of each month.

---

## File Structure

```
├── International_Ops.json               # n8n workflow export (import this)
├── README.md                            # This file
├── templates/
│   ├── email_vendor_acknowledgement.html
│   ├── email_reviewer_notification.html
│   ├── email_approval_invoice.html
│   └── email_rejection.html
├── assets/
│   └── invoice_template.docx            # Invoice template with {{placeholders}}
└── data/
    ├── vendor_database_sample.csv        # Sample vendor data for Notion import
    └── headcount_tracker_sample.xlsx     # Headcount tracker with Dashboard, Headcount, Monthly Trend tabs
```
