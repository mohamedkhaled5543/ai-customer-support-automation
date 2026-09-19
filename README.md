# AI Customer Support Automation

An end-to-end customer support system built with **n8n**.

It reads incoming customer emails, uses an AI agent to classify them, routes each request to the right handler, logs it, and notifies the customer and the team automatically.

---

## Problem

Support inboxes are slow and messy.

- Emails are read and sorted by hand.
- Requests get lost or handled twice.
- Refunds and operational requests need manual follow-up.
- Failures go unnoticed.

## Solution

Three connected n8n workflows handle the full lifecycle:

1. **Intake** reads and classifies the email.
2. **Handler** routes and processes the request.
3. **Error Handler** alerts you when anything fails.

---

## Architecture

```
Gmail Trigger
     |
     v
Validate + Clean (JavaScript)
     |
     v
Valid? --- No ---> Notify Customer (Gmail)
     |
    Yes
     |
     v
AI Agent (LLM + Structured Output)
     |
     v
Route by Request Type
     |
     v
Customer Support Handler
     |
     +--- Operational (e.g. refund) ---> Log to Google Sheets
     |                                       |
     |                                       v
     |                              Telegram Alert to Team
     |                                       |
     |                                       v
     |                              Confirmation Email (Gmail)
     |
     +--- Inquiry ---> AI Agent (with memory) ---> Reply Email (Gmail)

Any failure ---> Error Handler ---> Telegram Alert
```

---

## Workflows

### 1. Business Operations Intake — v1

Entry point. Turns a raw email into a structured request.

- **Gmail Trigger** receives new customer emails.
- **Code (JavaScript)** cleans and validates the data.
- **If** checks that the request is valid.
- **AI Agent** classifies the request using an LLM through OpenRouter.
- **Structured Output Parser** forces a clean, predictable JSON output.
- **Switch** routes by the classified type.
- **Execute Workflow** hands the request to the Support Handler.
- Invalid requests trigger a **Gmail** message asking the customer to fix and resend.

AI output fields:

| Field | Example |
|-------|---------|
| `request_type` | complaint |
| `department` | customer_support |
| `priority` | medium |
| `requested_action` | escalate complaint |

### 2. Customer Support Handler

Sub-workflow. Processes each classified request.

**Operational path (e.g. refunds):**
- Maps the correct sheet configuration.
- Reads existing rows from **Google Sheets**.
- Prepares the record and appends or updates it.
- Formats an operational email.
- Sends a **Telegram** alert to the responsible team.
- Sends a confirmation email to the customer via **Gmail**.

**Inquiry path:**
- An **AI Agent** (LLM through OpenRouter, with memory) writes the answer.
- Formats the reply and sends it via **Gmail**.

Final output of a processed request:

```
request_id:   REQ-<timestamp>
status:       processed
action:       Process refund for order
route:        refund
processed_at: <timestamp>
```

### 3. Customer Operations — Error Handler

Safety net for the whole system.

- **Error Trigger** fires when any linked workflow fails.
- **Edit Fields** builds a readable error message.
- **Telegram** sends the alert instantly.

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| n8n | Workflow orchestration |
| OpenRouter | LLM access for the AI agents |
| Gmail | Intake and customer emails |
| Google Sheets | Request log |
| Telegram | Team notifications and error alerts |
| JavaScript | Validation and data cleaning |

---

## Screenshots

**Intake workflow**
![Intake workflow](images/intake-workflow.png)

**Support handler**
![Support handler](images/support-handler.png)

**Error handler**
![Error handler](images/error-handler.png)

---

## Setup

1. Install n8n (cloud or self-hosted).
2. Import the JSON files from `/workflows`.
3. Add credentials: Gmail, Google Sheets, Telegram, OpenRouter.
4. Create the Google Sheet with the columns used in the **Map Sheet Config** node.
5. Set the Telegram chat ID in the alert nodes.
6. Link the Error Handler in each workflow's settings.
7. Publish the Intake and Handler workflows.

---

## Engineering Decisions

- **Structured output.** The AI returns fixed fields, so routing stays predictable.
- **Validation before AI.** Bad input is rejected early. It saves tokens and avoids junk results.
- **Sub-workflow design.** Intake and handling are separate. Each can be tested and changed alone.
- **Dedicated error workflow.** Failures reach Telegram instead of failing silently.

---

## Future Improvements

- Knowledge base for answers (RAG)
- Multi-language support
- Sentiment tracking
- Ticket status dashboard
- Human approval step for high-priority cases
- Database instead of Google Sheets

---


