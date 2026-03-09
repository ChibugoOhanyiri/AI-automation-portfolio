<img width="1809" height="587" alt="image" src="https://github.com/user-attachments/assets/9f3f24da-83b5-4eb4-932d-8a72c4f93f42" />
# Intelligent Email Routing System
An n8n workflow that automatically reads incoming emails, classifies them using AI, and routes them to the correct department via Slack notification and Gmail label assignment.

## Overview
[Company name] receives a high volume of emails daily across multiple departments. Manually sorting these emails causes delays and misrouting. This workflow solves that by monitoring the inbox continuously, using an AI model to classify each email by intent and context, and automatically routing it to the correct team — notifying them via Slack and tagging the email with a department label in Gmail. Emails that have already been processed are skipped to prevent duplicate routing.

## Prerequisites
- n8n (v1.0 or later recommended)
- Gmail OAuth2 credentials
- OpenAI API key (model used: `gpt-4.1-mini`)
- Google Sheets OAuth2 credentials
- Slack OAuth2 credentials
- A Google Sheet configured as a routing config table (see Setup Instructions)
- Gmail labels created for each department + a `Processed` label

## Setup Instructions

1. **Import the workflow** — In n8n, go to Workflows → Import → paste or upload `Intelligent_Email_Routing_System.json`

2. **Configure credentials** — Connect the following in the credentials panel:
   - Gmail OAuth2 (used by Gmail Trigger, Add Label nodes)
   - OpenAI API (used by OpenAI Chat Model node)
   - Google Sheets OAuth2 (used by the sheet lookup node)
   - Slack OAuth2 (used by the Send Notification node)

3. **Set up the Routing Config Google Sheet** — Create a sheet with the following columns:

   | Department | Email | Slack Channel | Label ID |
   |---|---|---|---|
   | Sales | sales@company.com | #sales-team | Label_XXXXXXXXX |
   | Logistics | logistics@company.com | #logistics-team | Label_XXXXXXXXX |
   | Billing | billing@company.com | #billings-team | Label_XXXXXXXXX |
   | Customer Support | support@company.com | #customer-support-team | Label_XXXXXXXXX |
   | Office/Admin | admin@company.com | #admin | Label_XXXXXXXXX |

   To get Label IDs: use the Gmail API or n8n Gmail node (Get Many Labels operation) to retrieve the internal ID for each label you create.

4. **Update the Google Sheets node** — Open the `Get dept. email and slack channel` node and point it to your routing config sheet.

5. **Create Gmail Labels** — In Gmail, manually create labels matching your department names exactly: `Sales`, `Logistics`, `Billing`, `Customer Support`, `Office/Admin`, and `Processed`.

6. **Activate the workflow** — Toggle the workflow to Active. It will poll every minute for new unprocessed emails.

## Basic LLM chain prompt used
### ROLE

You're an intelligent email classifier for [company] Store that classifies emails based on context, not just keywords.

### TASK
Classify the incoming email into exactly ONE of these:
- Sales
- Logistics
- Billing
- Customer Support
- Office/Admin

#### KEYWORDS
- Sales: Product inquiries, bulk orders, pricing discussions, partnerships, purchase requests.
- Logistics: Delivery issues, shipment tracking, inventory movement, dispatch coordination.
- Billing: Invoices, payments, receipts, refunds, and financial questions.
- Customer Support: Complaints, product issues, returns, general assistance.
- Office/Admin: Internal communication, vendor messages, unclear or uncategorized emails.

#### CONFIDENCE SCORING
Assign a confidence score between 0.00 and 1.00 based on how certain you are on your classification:
- 0.75 - 1.00 → High Confidence: clear intent, route directly
- 0.00 - 0.74 → Risky: unclear intent, default label must be Office/Admin

### RULES
- If confidence is below 0.75, you MUST setthe  label to "Office/Admin"
- Never guess -- when in doubt, use Office/Admin
- Ignore email signatures, greetings, and formatting
- Focus only on the core intent of the message
- Do not rely on KEYWORDS alone, analyze email context as well

### OUTPUT
Return this JSON:
{"label": "Department Name", "confidence": 0.00}
From: {{ $json.From }}
Subject: {{ $json.Subject }}
Body: {{ $json.Email_snippet }}
Received: {{ $json.Received_at }}
Message_id: {{ $json.Message_id }}

## Workflow Logic

**1. Gmail Trigger**
Polls the inbox every minute. Only fetches emails that are in the inbox and do not have the `Processed` label, preventing duplicate processing.

**2. Get Email Info (Edit Fields)**
Extracts and normalises key fields from the raw Gmail payload: Message ID, From, Subject, body snippet, and received timestamp.

**3. Handle Routing Logic and Confidence Score (LLM Chain)**
Passes the email subject and body to GPT-4.1-mini with a structured prompt. The model returns a JSON object containing the classified department label and a confidence score between 0.00 and 1.00.

**4. Split Logic Reasoning (Code Node)**
Parses the LLM JSON output. Applies confidence threshold logic: if confidence is 0.75 or above, the AI label is used; if below 0.75, the label is overridden to `Office/Admin`. Outputs a clean object with all email fields plus `finalLabel`, `confidence`, and `confidence_level`.

**5. Get Dept. Email and Slack Channel (Google Sheets)**
Looks up the `finalLabel` in the routing config sheet and returns the corresponding department email, Slack channel name, and Gmail Label ID.

**6. Preserve Email (Code Node)**
Merges the routing data from the sheet with the original email fields into a single unified object for downstream nodes.

**7. Send Notification (Slack)**
Posts a message to the department's Slack channel with the sender, subject, received time, and email ID.

**8. Attach Email Label to Dept (Gmail)**
Adds the department Gmail label (e.g. `Sales`) to the original email using the Label ID from the routing config sheet.

**9. Attach Processed/Routed Label (Gmail)**
Adds the `Processed` label to the email, ensuring it will not be picked up and re-routed on the next polling cycle.

## Notes

- **Confidence threshold** is set at 0.75. Emails below this score are routed to `Office/Admin` by default. You can adjust this in the `Split logic reasoning` code node.
- **Gmail snippet vs full body** — The workflow uses Gmail's snippet (first ~200 characters) for classification. For longer or more complex emails, consider adding a Gmail `Get Message` node to fetch the full body for higher accuracy.
- **Label IDs must be correct** — Gmail's Add Label node requires the internal Label ID (e.g. `Label_4244121651271311066`), not the display name. Incorrect IDs will cause the labelling step to fail.
- **Slack channel names** — The Slack channel field in the routing sheet must match your actual Slack channel IDs or names exactly as configured in your workspace.
- **Polling interval** — Currently set to every minute. Adjust in the Gmail Trigger node based on your volume and n8n plan execution limits.
- **No duplicate protection beyond labels** — Duplicate prevention relies entirely on the `Processed` label. If the labelling step fails mid-execution, the email may be reprocessed on the next poll.
