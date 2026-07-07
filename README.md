# Agentic AI Personal Expense Ledger & Analytics Pipeline

A self-hosted automation system built on n8n, Groq LLM models, and the Telegram Bot API. It converts free-text expense messages into structured ledger entries in Google Sheets, with real-time budget alerts and a weekly automated spending report.

---

## Overview

Instead of opening a spreadsheet or app, you text a Telegram bot naturally (e.g. *"Spent 350 on coffee at Starbucks"*). An LLM extracts amount, category, and vendor, validates the message, writes it to a ledger, and fires a high-value purchase alert if it crosses a threshold. A separate scheduled workflow reads the full ledger every Sunday and sends a categorized spending summary.

---

## Visual Architecture

![Workflow Diagram](./Workflow.PNG)

## Architecture

Two independent, decoupled workflows:

**Workflow 1 â€” Real-Time Ingestion**
```
Telegram Trigger â†’ Groq API â†’ If Node (Safety Filter) â†’ Google Sheets â†’ If Node (Risk Gate) â†’ Telegram Notification
```

**Workflow 2 â€” Batch Reporting**
```
Schedule Trigger â†’ Google Sheets (Read All) â†’ Code Node (Aggregation Loop) â†’ Telegram Broadcast
```

---

## Features

| Feature | Description |
|---|---|
| Frictionless logging | Log expenses via natural Telegram messages, no app or form |
| Contextual categorization | LLM infers category and normalizes vendor name from casual phrasing |
| Budget alerts | Real-time Telegram notification when a single transaction exceeds a configured threshold (e.g. â‚¹5,000) |
| Automated weekly reports | Scheduled job aggregates the ledger and sends a spending breakdown every Sunday |

---

## Tech Stack

- **Orchestration:** n8n (self-hosted)
- **AI/LLM:** Groq API (open-source models)
- **Messaging:** Telegram Bot API, Ngrok (tunneling)
- **Storage:** Google Sheets + Drive API (OAuth 2.0)
- **Processing:** JavaScript (ES6+), JSON

---

## Implementation Guide

### Phase 1 â€” Real-Time Ingestion
1. **Telegram bot setup** â€” created via `@BotFather` to receive incoming chat events.
2. **Tunneling** â€” Ngrok exposes the local n8n webhook (`http://localhost:5678`) over HTTPS for Telegram's webhook requirements.
3. **Groq integration** â€” an HTTP Request node calls Groq's API with a system prompt enforcing a fixed JSON schema: `amount`, `category`, `vendor`.
4. **Deserialization** â€” inline JavaScript (`JSON.parse()`) unpacks Groq's string response into usable data fields for downstream nodes.

### Phase 2 â€” Filtering & Ledger Routing
1. **Input safety filter (If node)** â€” rejects non-financial messages (empty `amount`) before they reach the sheet; sends the user a warning on the false branch.
2. **Google Sheets integration** â€” OAuth 2.0 credentials authorize writes to a ledger sheet (`ExpenseTracker`).
3. **Budget escalation gate (second If node)** â€” transactions under â‚¹5,000 get a standard confirmation; transactions at or above â‚¹5,000 trigger a ðŸš¨ budget alert message instead.

### Phase 3 â€” Automated Reporting
1. **Schedule trigger** â€” a cron node runs the second workflow every Sunday evening.
2. **Aggregation (Code node)** â€” a JavaScript loop scans all ledger rows, builds per-category totals, and computes total spend.
3. **Broadcast** â€” the formatted summary is sent to a hardcoded Telegram chat ID.
