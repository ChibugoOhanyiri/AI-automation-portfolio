# 🛒 Customer Support Intelligence Pipeline
<img width="1634" height="594" alt="Screenshot 2026-04-08 155317" src="https://github.com/user-attachments/assets/9ca95038-cc31-4736-a372-08b0d5bef3d8" />

> An end-to-end AI-powered customer support automation system built for an ecommerce brand. Incoming customer messages are classified, scored, escalated where necessary, and routed into a live operations dashboard — with zero manual intervention.

---

## 📌 Project Overview

E-commerce support teams spend hours every day reading, categorising, and routing the same types of customer messages. This pipeline replaces that manual work entirely.

Every incoming message is processed through an AI classification layer that identifies the issue type, detects customer sentiment, assesses urgency, and assigns a confidence score. A guardrail then decides whether the case needs human review or can be auto-resolved. All data flows into a Monday.com operations board where support trends, escalation rates, and bottlenecks are tracked in real time.

**Built with:** n8n · OpenAI GPT · Gmail · Monday.com  
**Deployed in:** Under 24 hours  
**Automation rate:** ~50% of support tickets handled with zero human involvement

---

## 🏗️ System Architecture

```
Webhook (POST)
    │
    ▼
Generate Message Data (Code Node)
    │  Extracts: order_id, message, timestamp
    ▼
Basic LLM Chain (OpenAI GPT)
    │  + Structured Output Parser (enforces JSON schema)
    │  Classifies: category, sentiment, urgency, confidence
    ▼
Format Data (Set Node)
    │  Maps all fields + calculates escalation flag
    ▼
Should We Escalate? (IF Node) ◄── GUARDRAIL
    │
    ├── TRUE (angry + confidence > 0.75)
    │       ▼
    │   Send Escalation Email (Gmail)
    │       ▼
    │   Upload to Escalated Group (Monday.com)
    │
    └── FALSE
            ▼
        Upload to Not Escalated Group (Monday.com)
```

---

## ⚙️ Node-by-Node Breakdown

### 1. 📥 Webhook — Input Layer
**Type:** `n8n-nodes-base.webhook`  
**Method:** POST

Receives incoming customer support messages from any channel — email, chat forms, or CRM integrations. Listens for a JSON payload containing the order ID and message content.

**Expected payload:**
```json
{
  "order_id": "ORD-1001",
  "message": "My order hasn't arrived and I'm frustrated. Can someone help?"
}
```

---

### 2. 🔧 Generate Message Data — Data Extraction
**Type:** `n8n-nodes-base.code`

Extracts the relevant fields from the raw webhook body and appends a UTC timestamp. This step normalises the input before it enters the AI layer, ensuring clean, consistent data regardless of the source.

**Output fields:**
| Field | Description |
|-------|-------------|
| `order_id` | Customer order reference |
| `message` | Raw customer message text |
| `timestamp` | ISO 8601 UTC timestamp |

---

### 3. 🤖 Basic LLM Chain — AI Classification
**Type:** `@n8n/n8n-nodes-langchain.chainLlm`  
**Model:** OpenAI GPT (via n8n free credits)  
**Output Parser:** Structured Output Parser (enforces JSON schema)

The core AI layer. The LLM analyses each message and returns a structured classification. A detailed system prompt instructs the model to follow specific rules — frustration maps to angry sentiment and high urgency, unclear messages receive a low confidence score.

**Prompt rules:**
- Frustration or angry language → `sentiment: angry`, `urgency: high`
- Questions about order location/status → `category: Order Status`
- Requests to send items back → `category: Return`
- Ambiguous or unclear messages → `confidence < 0.75`

**Output schema (enforced by Structured Output Parser):**
```json
{
  "category": "Order Status | Product Question | Complaint | Return",
  "sentiment": "angry | neutral | happy",
  "urgency": "high | medium | low",
  "confidence": 0.00
}
```

> **Why Structured Output Parser?**  
> Without it, the LLM can return markdown, explanations, or inconsistently formatted JSON that breaks downstream nodes. The parser enforces a strict schema — ensuring the pipeline never fails due to an unexpected AI response.

---

### 4. 📋 Format Data — Data Mapping
**Type:** `n8n-nodes-base.set`

Maps the AI output to named fields and calculates two additional values inline — the formatted timestamp and the escalation flag.

**Field mappings:**
| Output Field | Source |
|-------------|--------|
| `Order_id` | `Generate message data` node |
| `Category` | LLM output |
| `Sentiment` | LLM output |
| `Urgency` | LLM output |
| `Confidence` | LLM output |
| `Timestamp` | Converted from ISO to `DD-MM-YYYY` |
| `Escalated` | Calculated: `confidence < 0.75 ? 'Yes' : 'No'` |

**Timestamp conversion expression:**
```javascript
{{ new Date($('Generate message data').item.json.timestamp)
    .toLocaleDateString('en-GB')
    .replace(/\//g, '-') }}
```

---

### 5. ⚡ Should We Escalate? — Guardrail (IF Node)
**Type:** `n8n-nodes-base.if`

This is the confidence-based guardrail. It evaluates two conditions simultaneously:

```
Sentiment === "angry"  AND  Confidence > 0.75
```

**TRUE branch** → The AI is confident the customer is genuinely angry. Escalate immediately.  
**FALSE branch** → Routine ticket or low-confidence result. Auto-resolve and log.

> **Why both conditions?**  
> Sentiment alone isn't enough. If the AI detects anger but isn't confident (e.g. sarcasm, ambiguous phrasing), the system withholds escalation to avoid false alarms. Only high-confidence angry detections trigger the escalation path.

---

### 6. 📧 Send a Message — Escalation Email (Gmail)
**Type:** `n8n-nodes-base.gmail`  
**Triggered by:** TRUE branch of guardrail

Fires an HTML-formatted email alert to the support team the moment a high-urgency complaint is detected. No one needs to monitor an inbox — the right person gets notified immediately.

**Email contains:**
- Order ID
- Issue category
- Detected sentiment
- Urgency level
- AI confidence score
- Timestamp

**Branded to Tobio's** with their green colour scheme (`#2d7a5f`) and footer.

---

### 7. 📌 Upload to Escalated Group — Monday.com (Escalated Path)
**Type:** `n8n-nodes-base.mondayCom`  
**Board:** Customer Support Intelligence Pipeline  
**Group:** Escalated
<img width="1893" height="797" alt="Screenshot 2026-04-02 093036" src="https://github.com/user-attachments/assets/90a7910a-fb1c-40c1-83ed-d8af0d90483d" />


Creates a new item in the Escalated group on the Monday.com board. All column values are mapped using Monday's internal column IDs (retrieved via the Get Board operation before building).

**Column mapping:**
| Monday Column | Internal ID | Value |
|--------------|-------------|-------|
| Category | `color_mm1v1pb0` | `{"label": "Order Status"}` |
| Sentiment | `color_mm1vz814` | `{"label": "angry"}` |
| Urgency | `color_mm1v799m` | `{"label": "high"}` |
| Confidence | `numeric_mm1vdvwa` | `0.9` |
| Escalated | `dropdown_mm1v7syc` | `{"labels": ["Yes"]}` |
| Timestamp | `date_mm1v5ye1` | `{"date": "YYYY-MM-DD"}` |

> **Note on date format:** Monday.com requires dates in `YYYY-MM-DD` format. The timestamp from Format Data is in `DD-MM-YYYY` so it's reversed inline:  
> `{{ value.split('-').reverse().join('-') }}`

---

### 8. 📌 Upload to Not Escalated Group — Monday.com (Auto-Resolved Path)
**Type:** `n8n-nodes-base.mondayCom`  
**Board:** Customer Support Intelligence Pipeline  
**Group:** Not Escalated
<img width="1543" height="722" alt="Screenshot 2026-04-02 093045" src="https://github.com/user-attachments/assets/6d8b4237-44a0-4892-a556-49fc58dc773d" />


Identical column mapping to the escalated node, with one difference — the Escalated dropdown is set to `No`. Routine tickets land here automatically, already categorised and timestamped, ready for reporting.

---

## 📊 Monday.com Dashboard
<img width="879" height="488" alt="Screenshot 2026-04-08 155602" src="https://github.com/user-attachments/assets/4a3b8ff1-5355-49e1-96e7-0ebbd4658b6b" />


The Monday.com board is configured with a live dashboard containing five widgets:

| Widget | Type | What it shows |
|--------|------|---------------|
| Total Support Tickets | Numbers | All tickets processed |
| Total Escalations | Numbers (filtered: Escalated = Yes) | High-urgency cases flagged |
| AI Auto-Resolved | Numbers (filtered: Escalated = No) | Tickets handled automatically |
| Tickets by Category | Bar Chart | Volume breakdown by issue type |
| Customer Sentiment | Pie Chart | Sentiment distribution across all tickets |
| Tickets by Urgency | Bar Chart | Urgency distribution |

**About This Dashboard:**  
> Tobio's Customer Support Intelligence Pipeline processes incoming customer messages in real-time using AI classification. Each ticket is automatically analysed for issue category, customer sentiment, urgency level, and AI confidence score. High-urgency complaints from angry customers are instantly escalated via email alert and routed for immediate human review. All data is populated automatically via n8n, requiring zero manual input.

---

## 🧪 Testing the Workflow

Use Postman or any HTTP client to simulate incoming customer messages.

**Endpoint:** `POST https://your-n8n-instance/webhook-test/{webhook-id}`

**Headers:**
```
Content-Type: application/json
```

**Test payloads:**

```json
// Order Status — normal
{
  "order_id": "ORD-1001",
  "message": "Hi, I placed an order 4 days ago and haven't received any update. Can you help?"
}

// Angry customer — triggers escalation
{
  "order_id": "ORD-1002",
  "message": "This is ridiculous. My order is late and no one is responding. I want an update NOW."
}

// Product question — auto-resolved
{
  "order_id": "ORD-1003",
  "message": "Hi, is your watercolor kit suitable for beginners?"
}

// Return request
{
  "order_id": "ORD-1004",
  "message": "I received the wrong item and I'd like to return it. What's the process?"
}

// Ambiguous — low confidence test
{
  "order_id": "ORD-1005",
  "message": "Hey, something is off with my order but I'm not sure what exactly. Can you check?"
}

// Positive — auto-resolved
{
  "order_id": "ORD-1006",
  "message": "Just wanted to say I love the kit I received, thank you!"
}
```

> **Tip:** Send all 6 to populate the Monday.com board with varied data across both groups.

---

## 🔑 Credentials Required

| Service | Credential Type | Used In |
|---------|----------------|---------|
| OpenAI | API Key | OpenAI Chat Model node |
| Gmail | OAuth2 | Send a message node |
| Monday.com | API Token | Both Monday.com nodes |

> **Getting Monday.com column IDs:**  
> Before mapping column values, use a Monday.com Get Board node to retrieve the internal column IDs. Display names like "Category" do not work — you need the actual IDs (e.g. `color_mm1v1pb0`). Status columns use `{"label": "value"}` format, number columns use a raw number, and dropdown columns use `{"labels": ["value"]}`.

---

## 📁 Repository Structure

```
customer-support-intelligence-pipeline/
│
├── README.md                          # This file
├── Customer_Support_Intelligence_Pipeline.json   # n8n workflow export
└── sample-data/
    └── test-payloads.json             # Postman test payloads
```

---

## 🚀 How to Import

1. Open your n8n instance
2. Go to **Workflows → Import from File**
3. Upload `Customer_Support_Intelligence_Pipeline.json`
4. Add your credentials for OpenAI, Gmail, and Monday.com
5. Update the Monday.com Board ID and Group IDs to match your board
6. Activate the workflow
7. Copy the Production webhook URL and use it as your endpoint

---

## 💡 Business Impact

| Metric | Value |
|--------|-------|
| Tickets auto-resolved | ~50% |
| Time saved per 20 tickets | 110 minutes |
| Projected daily saving (100 tickets/day) | 9+ hours |
| Estimated monthly cost saving | ~£4,100 |
| Escalation response time | Seconds (vs hours) |

---

## 🔮 Phase 2 — Planned Enhancements

- **RAG Response Layer:** Connect a Tobio's-specific knowledge base so the AI can send actual responses to routine questions, not just classify them
- **Webhook Authentication:** Add HMAC signature verification to secure the endpoint
- **Retry Logic:** Handle API failures gracefully with exponential backoff
- **Slack Integration:** Route escalation alerts to a dedicated Slack channel in addition to email
- **Multi-channel Input:** Extend beyond webhook to process emails and chat messages directly

-
