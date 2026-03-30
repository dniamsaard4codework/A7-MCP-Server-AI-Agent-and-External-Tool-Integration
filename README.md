# A7: MCP-Server, AI Agent, and External Tool Integration

**NLU Assignment 7** — MCP-Server, AI Agent, and External Tool Integration
**Student**: Dechathon Niamsa-ard [st126235]

---

## Overview

This project builds an integrated AI Agent ecosystem using the **Model Context Protocol (MCP)** on n8n, deployed locally via **Docker** (with PostgreSQL backend) and exposed to the internet using a **Cloudflare Quick Tunnel** — replacing ngrok as the tunneling solution. The agent is connected to a **Telegram Bot** for real-time messaging and to **Google Calendar** for full **CRUD** event management (Create, Read, Update, Delete), demonstrating a practical NLU-powered scheduling assistant capable of understanding natural language commands and taking real-world actions.

> **Infrastructure note:** Instead of ngrok (as described in the guide), this setup uses `cloudflared tunnel --url http://localhost:5678` which generates a temporary public HTTPS URL with no account or token required. The URL is used as the n8n Webhook/Production URL for all MCP and Telegram callbacks.

---

## Task 1: MCP Infrastructure & Server Setup (3 Points)

### 1.1 Server Deployment — Docker + Cloudflare Tunnel (0.5 pts)

n8n is self-hosted via **Docker Compose** backed by a **PostgreSQL 16** database for persistent workflow and credential storage. The local instance is exposed publicly using **Cloudflare Quick Tunnel**:

```bash
cloudflared tunnel --url http://localhost:5678
```

This creates a stable HTTPS endpoint (e.g., `https://twiki-council-appointments-injuries.trycloudflare.com`) that n8n uses as its `WEBHOOK_URL`, making all webhooks and MCP SSE endpoints reachable from the internet.

![Cloudflare Tunnel Terminal](assets/cloudflare_tunnel.png)
*Cloudflare Quick Tunnel running in the terminal — the generated public HTTPS URL is printed and used as the n8n Webhook URL throughout the entire assignment.*

![n8n Accessed via Cloudflare URL](assets/1.1.png)
*n8n Welcome page loaded in the browser via the public Cloudflare URL, confirming the tunnel is active and the instance is reachable from the internet.*

![n8n Workflows Overview](assets/1.1_home.png)
*n8n workflow dashboard showing all workflows. Both **MCP Server** and **Client Side** are marked **Published** (green badge), confirming they are active and accessible via the public tunnel URL.*

---

### 1.2 MCP Server Workflow — 3 Internal Tools (1 pt)

A dedicated workflow named **"MCP Server"** acts as the tool provider. An **MCP Server Trigger** node is the entry point; three internal tool nodes branch from it:

| Tool | Node Type | Function |
| ---- | --------- | -------- |
| **Calculator** | Built-in Calculator | Evaluates arithmetic expressions |
| **Date & Time** | Code Tool (`Date_and_Time`) | Returns current date/time in Asia/Bangkok (UTC+7) |
| **Get_Latest_News** | HTTP Request Tool | Fetches BBC News RSS via `api.rss2json.com` — no API key needed |

The workflow is toggled **Active** (Published). The resulting **Production URL** (ending in `/sse`) is the SSE endpoint that the AI Agent Client connects to.

![MCP Server Workflow](assets/1.2.png)
*MCP Server workflow: MCP Server Trigger connected to Calculator, Date & Time, and HTTP Request (Get_Latest_News). The **↑ Published** indicator at the top-right confirms the workflow is active, exposing all three tools over the SSE endpoint.*

---

### 1.3 AI Agent Client (1.5 pts)

A second workflow named **"Client Side"** contains the AI Agent. It is configured as a **Tools Agent** with the following components:

| Component | Configuration |
| --------- | ------------- |
| **Trigger** | When chat message received (Chat Trigger) |
| **AI Agent type** | Tools Agent |
| **Chat Model** | OpenAI Chat Model node → Groq API (`llama-3.3-70b-versatile`), Base URL: `https://api.groq.com/openai/v1` |
| **Memory** | Simple Memory — Window Buffer Memory, context window length: 10 |
| **Tool** | MCP Client — SSE Endpoint set to the MCP Server Production URL |

The agent is verified by chatting directly in n8n. The execution log shows the MCP Client discovering and invoking tools on the MCP Server.

![Client Side — MCP News Query](assets/1.3.png)
*Client Side workflow executing a news query. The user asks for the latest BBC news; the agent calls the MCP Client → HTTP Request tool and returns structured BBC headlines. The execution log on the right confirms the full tool call chain: AI Agent → Simple Memory → Groq Chat Model → MCP Client → HTTP Request.*

---

## Task 2: Telegram & Google Calendar Integration (3 Points)

### 2.1 Telegram Bot API (0.5 pts)

The Client Side workflow is extended to use **Telegram** as the messaging channel:

- The **When chat message received** trigger is replaced with a **Telegram Trigger** node (listening for `message` updates, credential: Bot Token from BotFather).
- A **Send a text message** node is added after the AI Agent output, routing the agent's reply back to the user's Telegram chat.
- The bot is named **"ST126235 NLP A7 Schedule Bot"**.

Flow: `Telegram Trigger → AI Agent → Send a text message`

![Telegram Workflow in n8n](assets/2.1.png)
*Client Side workflow updated with Telegram Trigger (left) and Send a text message node (right). The execution log shows a successful Telegram send, with the output confirming the `message_id` and chat details.*

![Telegram Bot — Date & News](assets/2.1_telegram.png)
*Live Telegram conversation: the bot correctly reports today's date as **March 30, 2026** (Bangkok timezone) using the Date & Time MCP tool, and fetches BBC News front-page headlines using the Get_Latest_News MCP tool — both tools invoked transparently via the MCP Server.*

---

### 2.2 Google Calendar Tool (0.5 pts)

Five **Google Calendar** nodes are added to the AI Agent's tool set via OAuth 2.0 credentials (Google Cloud Console — Calendar API enabled, redirect URI set to the Cloudflare tunnel URL), providing full **CRUD** capability:

| Tool Node | Action | Purpose |
| --------- | ------ | ------- |
| **Create an event** | CREATE | Creates new calendar events with title, date range, all-day flag |
| **Get many events** | READ | Lists upcoming events within a date range |
| **Get an event** | READ | Retrieves a single event by ID or by querying a specific date |
| **Update an event** | UPDATE | Modifies event title, dates, or description |
| **Delete an event** | DELETE | Permanently removes an event from the calendar |

The agent's system prompt is configured to treat Asia/Bangkok (UTC+7) as the default timezone and always use ISO 8601 date formatting.

![Google Calendar Tools — Full CRUD Workflow](assets/2.2.png)
*AI Agent with all five Google Calendar tools visible. The execution log shows the agent confirming all four project phase events with Google Calendar direct links.*

![Final Client Side Workflow — All Tools](assets/2.3_flow.png)
*Final Client Side workflow state: Telegram Trigger → AI Agent → Send a text message. The AI Agent has 8 tools: Groq Chat Model, Simple Memory, MCP Client, and all five Google Calendar tools (Create, Get Many, Get One, Update, Delete). Execution log confirms successful calendar event creation.*

---

### 2.3 Automated Project Scheduling (1 pt)

A natural language scheduling command is sent to the bot via Telegram, requesting creation of four all-day project phase events:

> *"Please create 4 Google Calendar events for my project:*
> *1. '1st Phase: Literature Review' April 7 to April 20, 2026*
> *2. '2nd Phase: Project Proposal' April 21 to May 4, 2026*
> *3. '3rd Phase: Update Progress' May 5 to May 18, 2026*
> *4. '4th Phase: Final (Presentation)' May 19 to May 25, 2026*
> *Each should be an all-day event."*

The agent parses the intent, calls **Create an event in Google Calendar** four times, and replies with a confirmation summary.

| # | Event | Start | End |
| - | ----- | ----- | --- |
| 1 | 1st Phase: Literature Review | 2026-04-07 | 2026-04-20 |
| 2 | 2nd Phase: Project Proposal | 2026-04-21 | 2026-05-04 |
| 3 | 3rd Phase: Update Progress | 2026-05-05 | 2026-05-18 |
| 4 | 4th Phase: Final (Presentation) | 2026-05-19 | 2026-05-25 |

![Telegram — Scheduling Command & Confirmation](assets/2.3telegram.png)
*Telegram conversation: after verifying the date and fetching news via MCP tools, the user sends the project scheduling command. The bot confirms all four phases are created in Google Calendar with their exact date ranges.*

**Google Calendar — April 2026 (after creation):**

![Google Calendar April 2026](assets/2.3Apr.png)
*April 2026 calendar view confirming **"1st Phase: Literature Review"** (Apr 7–20) and the start of **"2nd Phase: Project Proposal"** (Apr 21) are created as all-day events.*

**Google Calendar — May 2026 (after creation):**

![Google Calendar May 2026](assets/2.3May.png)
*May 2026 calendar view confirming **"2nd Phase: Project Proposal"** (ending May 4), **"3rd Phase: Update Progress"** (May 5–18), and **"4th Phase: Final (Presentation)"** (May 19–25) — all four phases visible on the calendar.*

---

### 2.4 Interaction Verification — Full CRUD (1 pt)

The agent is verified through a complete **Create → Read → Update → Delete** cycle, all triggered via natural language in Telegram.

#### Read — Get Many Events

> *"What are my upcoming project phases? Can you list all events on my calendar for the next month?"*

![n8n — Get Many Execution](assets/2.4_GetMany.png)
*n8n execution log: AI Agent calls **Get many events in Google Calendar**, retrieves all four project phases, and returns them to the Telegram user.*

![Telegram — Get Many](assets/2.4_GetMany_telegram.png)
*Bot lists all four project phase events with exact date ranges, confirming the events are live and readable from Google Calendar.*

---

#### Read — Get One Event by Date

> *"What event I have on 12 April 2026?"*

![n8n — Get One Execution](assets/2.4_GetOne.png)
*n8n execution log: **Get an event in Google Calendar** returns the "1st Phase: Literature Review" event details (ID, summary, start/end times) for April 12, 2026.*

![Telegram — Get One](assets/2.4_GetOne_telegram.png)
*Bot correctly identifies that on April 12, 2026, the user has **"1st Phase: Literature Review"** — demonstrating single-event lookup by date.*

---

#### Update Event

> *"Update literature review to Start April 10 2026 and End with April 21 2026"*

![n8n — Update Execution](assets/2.4_Update.png)
*n8n execution log: **Update an event in Google Calendar** successfully modifies the "1st Phase: Literature Review" event — new start: 2026-04-10, new end: 2026-04-21. The output confirms the updated event ID and timestamps.*

![Telegram — Update Confirmation](assets/2.4_Update_telegram.png)
*Bot confirms the update: "The '1st Phase: Literature Review' event has been updated to start on April 10, 2026, and end on April 21, 2026."*

![Google Calendar — After Update](assets/2.4_Update_calendar.png)
*April 2026 calendar showing **"1st Phase: Literature Review"** now starts on April 10 (shifted from April 7), reflecting the update command from Telegram.*

---

#### Delete Event

> *"Delete 1st Phase: Literature Review"*

![Google Calendar — Before Delete](assets/2.4_Before_delete.png)
*April 2026 calendar showing the state before deletion — "1st Phase: Literature Review" (Apr 10–21) and "2nd Phase: Project Proposal" are visible. "Event saved" toast confirms the calendar is live.*

![n8n — Delete Execution](assets/2.4_Delete.png)
*n8n execution log: **Delete an event in Google Calendar** returns `success: true` — the event has been permanently removed.*

![Telegram — Delete Confirmation](assets/2.4_Delete_telegram.png)
*Bot confirms: "The event '1st Phase: Literature Review' has been deleted from your Google Calendar."*

![Google Calendar — After Delete](assets/2.4_After_delete.png)
*April 2026 calendar after deletion — "1st Phase: Literature Review" is gone. Only "2nd Phase: Project Proposal" (Apr 21+) and an unrelated event remain, confirming the delete was successful.*

---

#### Final State

After the CRUD cycle, the event is re-added to restore the original schedule:

> *"Add '1st Phase: Literature Review' April 7 to April 20, 2026"*

![Telegram — Full CRUD Conversation](assets/2.4_final_telegram.png)
*Complete Telegram conversation thread showing the full CRUD cycle: Create (×4) → Get Many → Update → Get One → Delete → Re-create. The bot handles every operation correctly through natural language, demonstrating end-to-end Google Calendar management.*

---

## Architecture Summary

```text
                   ┌─────────────────────────────────────────────────┐
                   │              MCP Server Workflow                │
                   │  MCP Server Trigger ──► Calculator              │
                   │                    ──► Date & Time (Code Tool)  │
                   │                    ──► Get_Latest_News (HTTP)   │
                   └───────────────────────┬─────────────────────────┘
                                           │ Production URL (SSE)
                   ┌───────────────────────▼─────────────────────────┐
                   │             Client Side Workflow                │
                   │  Telegram Trigger                               │
                   │       ↓                                         │
                   │  AI Agent  (Groq llama-3.3-70b-versatile)      │
                   │    ├── Simple Memory (Window Buffer, k=10)      │
                   │    ├── MCP Client ──► MCP Server (SSE)         │
                   │    ├── Create an event in Google Calendar       │
                   │    ├── Get many events in Google Calendar       │
                   │    ├── Get an event in Google Calendar          │
                   │    ├── Update an event in Google Calendar       │
                   │    └── Delete an event in Google Calendar       │
                   │       ↓                                         │
                   │  Send a text message (Telegram)                 │
                   └─────────────────────────────────────────────────┘
                                           ↑
                   cloudflared tunnel --url http://localhost:5678
                   (Cloudflare Quick Tunnel — public HTTPS, no account needed)
```

| Component | Details |
| --------- | ------- |
| **n8n** | Self-hosted via Docker, PostgreSQL 16 backend, port 5678 |
| **Tunnel** | Cloudflare Quick Tunnel (`cloudflared`) — no account or token required |
| **LLM** | Groq API — `llama-3.3-70b-versatile` (free tier, OpenAI-compatible) |
| **Memory** | Window Buffer Memory — context window 10 turns |
| **MCP Tools** | Calculator, Date & Time (Code), Get_Latest_News (BBC RSS via HTTP) |
| **Messaging** | Telegram Bot API — ST126235 NLP A7 Schedule Bot |
| **Calendar** | Google Calendar API via OAuth 2.0 — full CRUD (Create, Get Many, Get One, Update, Delete) |

---

## Setup

### 1. Configure `local_n8n/.env`

The `.env` file holds database credentials and the public tunnel URL used by n8n as its `WEBHOOK_URL`:

```env
# Database Config
DB_USER=n8n_admin
DB_PASSWORD=your_password
DB_NAME=n8n_db

# Tunnel URL (update this each time cloudflared generates a new URL)
NGROK_URL=https://<your-tunnel>.trycloudflare.com
```

> The variable is named `NGROK_URL` for compatibility with the `docker-compose.yaml` template, but the value is a **Cloudflare Quick Tunnel** URL.

### 2. Start the stack

```bash
cd local_n8n
docker compose up -d
```

### 3. Start the Cloudflare tunnel

```bash
cloudflared tunnel --url http://localhost:5678
# → Copy the generated HTTPS URL into local_n8n/.env as NGROK_URL, then restart:
docker compose restart n8n
```

### 4. Configure n8n workflows

```text
- Create / import the MCP Server workflow, publish it, note the Production URL (ends in /sse)
- Create the Client Side workflow:
    - Configure Groq API credentials (llama-3.3-70b-versatile)
    - Set MCP Client SSE Endpoint to the MCP Server Production URL
    - Configure Telegram Bot credentials (Bot Token from BotFather)
    - Configure Google Calendar OAuth2 credentials (5 nodes: Create, Get Many, Get One, Update, Delete)
- Activate the Client Side workflow
```

---

## References

- n8n Documentation — [docs.n8n.io](https://docs.n8n.io)
- Cloudflare Quick Tunnels — [developers.cloudflare.com/cloudflare-one/connections/connect-networks/do-more-with-tunnels/trycloudflare](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/do-more-with-tunnels/trycloudflare/)
- Groq API — [console.groq.com](https://console.groq.com)
- Google Calendar API — [developers.google.com/calendar](https://developers.google.com/calendar)
- Telegram Bot API — [core.telegram.org/bots/api](https://core.telegram.org/bots/api)
- Assignment reference environment — [github.com/chaklam-silpasuwanchai/Python-fo-Natural-Language-Processing/tree/main/Code/11%20-%20Agentic%20AI/local_n8n](https://github.com/chaklam-silpasuwanchai/Python-fo-Natural-Language-Processing/tree/main/Code/11%20-%20Agentic%20AI/local_n8n)

---
