# butler-agent
the butler agent is an automated assistant that handles calendar and event setting, task management, emails, content creation
[README.md](https://github.com/user-attachments/files/26347210/README.md)
# butler-agent

A personal AI ops assistant built on n8n. Accepts natural language commands and autonomously handles calendar management, email operations, task tracking, and content workflows — without requiring multiple apps or manual context switching.

**Demo:** [Watch on X →](https://x.com/0xTrapo/status/2034986520638091494)

---

## Overview

butler-agent is a hub-and-spoke orchestration system. A central orchestrator receives every incoming message, determines which actions are required, and delegates to the appropriate sub-agent. Multiple actions can be triggered from a single message — the orchestrator handles sequencing without follow-up questions.

The system runs entirely on n8n and is operated through a private Telegram bot.

---

## Architecture

```
Telegram Trigger (can also be Slack, Whatsapp)
      │
      ▼
   If (auth gate) ──► reject user (unauthorized)
      │
      ▼
  Orchestrator (LLM Model — Claude Sonnet 4.5)
  ├── Postgres Chat Memory (context window: 10 turns)
  ├── get_contact_data (Google Sheets)
  ├── tavily_search (web research)
  ├── calendarAgent ──► Google Calendar (fetch / create / update / delete)
  ├── emailAgent ──► Gmail (read / send / draft / reply / label / search)
  ├── taskManager ──► Google Sheets Tasks (add / complete / update)
  ├── contentAgent ──► content research and writing tasks (search / create / update)
  └── [output] ──► Telegram reply

```

---

## Agents

### Orchestrator
The main routing brain. Receives raw Telegram messages, resolves intent, pulls contact data, and constructs clean formatted instructions before passing them to any sub-agent. Never passes raw user messages downstream.

Handles multi-action messages (e.g. "book a meeting with Sylvia and draft a follow-up email") in a single pass.

### emailAgent (Mailer)
Handles all Gmail operations:
- Read / search unread messages
- Send new emails
- Reply to threads
- Draft emails without sending
- Add labels
- Summarize threads

Applies composition rules: first-name greeting, 3–5 sentences, professional and direct, no filler. Signs off with the user's name when provided.

### taskManager (Tasker)
Manages a Google Sheets task list with three columns: Date, Task, Priority (1–5 scale).

- **Add** — appends a new row with today's date and default priority 3
- **Complete** — reads the sheet, finds the row by task name or row number, deletes it
- **Update** — shifts date, renames task, or changes priority without touching other columns
- **Meeting-to-tasks** — extracts action items from raw meeting notes and logs each as a separate row

### calendarAgent
Manages Google Calendar events:
- Create meetings with title, datetime, attendee email, and agenda
- Reschedule existing events
- Delete/cancel events
- Fetch upcoming events

The orchestrator always resolves contact email from the Google Sheets contact list before calling this agent — no manual email entry required.

### content agent
Handles content research using Tavily, Google Docs creation, and Google Drive uploads. Responds back to Telegram on completion.

---

## Access Control

An **If node** at the entry point checks the Telegram sender ID against an allowlist. Unauthorized users receive a rejection message and the workflow terminates. No data is processed for unrecognized IDs.

---

## Data Layer

| Store | Purpose |
|---|---|
| Google Sheets (`contactList` sheet) | Contact database — name, email, company, role, notes |
| Google Sheets (`Tasks` sheet) | Task list — date, task description, priority |
| Postgres (`n8n_chat_memory` table) | Conversation context for orchestrator (10-turn window) |
| Postgres (separate instance) | Conversation context for content agent |
| Google Calendar | Live calendar events |
| Gmail | Email send/receive/draft/label |
| Google Drive / Docs | Content storage |

---

## Example Commands

```
"Book a meeting with Miller on April 9th at 3pm. Agenda: discovery call."
"Draft a follow-up email to Hailey about the one-pager."
"Add task: finish client proposal. Priority: high."
"Mark the Miley Immigration task as done."
"What unread emails do I have from last week?"
"Research MacMellan CPA before my call."
"Summarize my email thread with Keira."
```

---

## Setup

### Prerequisites
- n8n (self-hosted or cloud)
- Telegram bot token
- LLM API key
- Google account with Sheets, Calendar, Gmail, and Drive OAuth configured
- Postgres database (for chat memory)
- Tavily API key

### Steps

1. Import `chiefAgent.json` into your n8n instance
2. Reconnect all credentials (Telegram, Anthropic, Google Sheets, Gmail, Google Calendar, Google Drive, Postgres, Tavily)
3. Update the **If node** allowlist with your Telegram user ID(s)
4. Create a Google Sheet with two tabs:
   - `contactList`: columns — name, email, company, role, notes
   - `Tasks`: columns — Date, Task, Priority
5. Update all Google Sheets nodes to point to your sheet ID
6. Ensure your Postgres instance has an `n8n_chat_memory` table (n8n creates this automatically on first run if the LangChain memory node has the right credentials)
7. Activate the workflow

### Getting your Telegram user ID
Message `@userinfobot` on Telegram — it returns your numeric user ID to add to the allowlist.

---

## File Structure

```
butler-agent/
├── chiefAgent.json        # Full n8n workflow export
├── README.md
└── assets/
    └── chiefAgent overview.png       # Workflow canvas screenshot

```

---

## Built by

[@0xTrapo](https://x.com/0xTrapo) — AI automation consultant building workflow systems for B2B operations.
