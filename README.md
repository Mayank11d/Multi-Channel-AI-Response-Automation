# Multi-Channel AI Response Automation

An n8n workflow that unifies customer messages from **Email**, **WhatsApp**, and a **Chatbot** into a single AI-powered pipeline — automatically handling bookings, cancellations, reschedules, pricing queries, and general support.

---

## Overview

```
Gmail / WhatsApp / Chatbot
        ↓
  Normalize & Merge
        ↓
  Extract Contact Details
        ↓
  MongoDB (User Profile)
        ↓
  AI Agent (GPT-4o-mini)
  ├── Short-term: Buffer Memory
  ├── Long-term: Qdrant Vector Store
  └── Tools: Create / Cancel / Reschedule / Check Slot / List Bookings
        ↓
  Format & Route Response
        ↓
Gmail / WhatsApp Webhook / Chatbot
```

---

## Features

- **Multi-channel ingestion** — handles Gmail, WhatsApp webhooks, and n8n chatbot triggers in a unified pipeline
- **AI customer service agent** — powered by GPT-4o-mini with a detailed system prompt covering booking, pricing, complaints, and general enquiries
- **Dual memory system** — short-term buffer window memory (per session) + long-term Qdrant vector store for cross-session recall
- **MongoDB user profiles** — upserts user contact, name, address, and email on every message
- **5 booking tools** — Create Booking, Cancel Booking, Reschedule Booking, Check Slot Availability, Get Booking List (each calls a separate sub-workflow)
- **Channel-aware formatting** — HTML email formatting for Gmail, WhatsApp markdown (`*bold*`), plain text for chatbot
- **Automatic response routing** — Switch node routes the AI reply back to the correct output channel

---

## Nodes Reference

| Node | Type | Purpose |
|---|---|---|
| `Gmail Incoming Trigger` | Gmail Trigger | Polls inbox every minute for new messages |
| `WhatsApp Incoming Webhook` | Webhook (POST) | Receives WhatsApp Cloud API payloads |
| `When chat message received` | Chat Trigger | Receives messages from n8n chatbot UI |
| `Get Gmail Message Details` | Gmail | Fetches full email body/headers by message ID |
| `Normalize Gmail / WhatsApp / Chatbot Data` | Code | Extracts `from`, `text`, `channel` into a standard shape |
| `Merge Channels Input` | Merge (3 inputs) | Combines all three channel streams |
| `Extract Contact & Address Details` | Code | Regex-extracts name, phone, email, address from message text |
| `Manage User` | MongoDB | Upserts user record into `user_details` collection |
| `Prepare AI Agent Context` | Code | Merges stored user profile with current message data |
| `Build AI Agent Payload` | Set | Maps fields to `to`, `body`, `channel`, `subject`, etc. |
| `AI Agent` | LangChain Agent | Core reasoning node — GPT-4o-mini with tools and memory |
| `OpenAI Model for AI Agent` | OpenAI Chat | `gpt-4o-mini` language model |
| `Simple Memory` | Buffer Window Memory | Short-term in-session memory keyed by `$json.to` |
| `Qdrant Vector Store` | Qdrant (retrieve-as-tool) | Long-term vector memory from `chat_history` collection |
| `OpenAI Embeddings for Qdrant` | OpenAI Embeddings | Embeds queries for Qdrant retrieval |
| `Get Booking List` | Tool Workflow | Retrieves all bookings for a customer ID |
| `Create Booking` | Tool Workflow | Creates a new booking with full details |
| `Cancel Booking` | Tool Workflow | Cancels a booking by ID |
| `Reschedule Booking` | Tool Workflow | Reschedules an existing booking |
| `Check Slot Availability` | Tool Workflow | Checks if a date/time slot is free |
| `Format AI Final Response` | Code | Parses agent JSON output, applies channel formatting |
| `Check If Email Channel` | IF | Branches email vs other channels |
| `Format Email HTML` | Code | Wraps email body in `<p>` tags |
| `Route To Channel Output` | Switch | Routes to Email / WhatsApp / Chatbot output |
| `Send Email Response` | Gmail | Sends formatted reply via Gmail OAuth |
| `Build WhatsApp Response` / `Send WhatsApp Response` | Set + Respond to Webhook | Returns WhatsApp reply |
| `Build Chatbot Response` / `Send Chatbot Response` | Set + Chat | Sends chatbot reply |
| `Build Memory Payload` | Code | Packages user + assistant messages for Qdrant |
| `Save Memory To Qdrant` | Execute Workflow | Fires sub-workflow to persist conversation to Qdrant |

---

## Sub-Workflows Required

This workflow calls five external sub-workflows by ID. You must have these configured in your n8n instance:

| Sub-workflow name | Called by |
|---|---|
| `booking list` (`bxFuiMNUbu56Slaj`) | Get Booking List tool |
| `create booking` (`U5asKGLkNl29QLqt`) | Create Booking tool |
| `Cancel Booking` (`iXu52au5Xks72mgL`) | Cancel Booking tool |
| `Reschedule Booking` (`uOt6lFcsADJaZuPo`) | Reschedule Booking tool |
| `Check Slot` (`PbWlcPXvFvNJTOMB`) | Check Slot Availability tool |
| `Retrive` (memory, `AHAIGw5lIqmOyiaC`) | Save Memory To Qdrant |

---

## Required Credentials

| Credential | Used by |
|---|---|
| `Gmail OAuth2` | Gmail Incoming Trigger, Get Gmail Message Details, Send Email Response |
| `OpenAI API` | OpenAI Model for AI Agent, OpenAI Embeddings for Qdrant |
| `Qdrant API` | Qdrant Vector Store |
| `MongoDB` | Manage User |

---

## Data Flow in Detail

### 1. Ingestion & Normalization
Each channel produces a normalized object:
```json
{
  "from": "<sender_id_or_email>",
  "text": "<message_body>",
  "channel": "email | whatsapp | chatbot"
}
```

### 2. Contact Extraction
The `Extract Contact & Address Details` node regex-parses the message for:
- Name (`my name is ...`, `hi <Name>`)
- Phone (Indian mobile: `[6-9]XXXXXXXXX`)
- Email address
- Address/location keywords

### 3. User Profile (MongoDB)
The `user_details` collection is upserted with `user_id` as the key. Existing fields from prior messages are preserved via the `Find chat history` merge logic.

### 4. AI Agent
The agent receives a fully populated payload including user profile, channel, and message body. It reasons with:
- **Buffer window memory** for the current session
- **Qdrant vector store** for older cross-session context
- **5 tool workflows** for live booking operations

The system prompt enforces:
- Always check slot availability before creating a booking
- Phone number required for new users
- Never ask again for details already in memory
- JSON-only output format

### 5. Response Output Format
The agent always outputs valid JSON:
```json
{
  "to": "<recipient>",
  "subject": "<email subject or empty>",
  "body": "<response message>",
  "channel": "email | whatsapp | chatbot",
  "category": "<Booking | Complaint | Pricing | ...>"
}
```

### 6. Channel Routing
- **Email** → HTML-formatted body → Gmail Send
- **WhatsApp** → Plain text with `*bold*` markdown → Webhook respond
- **Chatbot** → Plain text → n8n Chat respond

---

## Message Categories

The AI agent classifies every message into one of:

- Enquiry
- Booking / Confirmation
- Complaint / Issue
- Reschedule / Cancellation
- Quotation / Price Request
- Follow-up / Status Check
- Feedback / Suggestions
- Emergency
- Others

---

## Memory Architecture

```
Short-term (Buffer Window)      Long-term (Qdrant)
──────────────────────────      ──────────────────
In-memory, current session      Persisted across sessions
Keyed by session_id (to)        Embedded with OpenAI embeddings
Auto-managed by LangChain       Stored via "Retrive" sub-workflow
Used for recent context         Used for cross-session recall
```

The agent is instructed to reconstruct memory from Qdrant by matching `session_id` in the `pageContent` JSON before asking the user for previously provided details.

---

## Booking Tool Logic

```
User requests a booking
        ↓
  All fields present? ──No──→ Ask for missing field
        ↓ Yes
  Check Slot Availability
        ↓
  Available? ──No──→ Show alternatives, ask user to choose
        ↓ Yes
  Create Booking
        ↓
  Confirm booking_id to user
```

For **cancel** or **reschedule**, `booking_id` is required. If not in memory, the agent calls `Get Booking List` first.

---

## Setup Checklist

- [ ] Import this workflow JSON into your n8n instance
- [ ] Connect all credentials (Gmail, OpenAI, Qdrant, MongoDB)
- [ ] Create the six sub-workflows listed above and update their IDs if different
- [ ] Create a `chat_history` collection in your Qdrant instance
- [ ] Create a `user_details` collection in your MongoDB instance
- [ ] Activate the workflow
- [ ] Test with a WhatsApp webhook test, a Gmail message, and the n8n chatbot UI

---

## Notes

- The workflow is currently **inactive** (`"active": false`) — enable it after completing setup.
- The `Normalize Gmail Data` node strips HTML tags from the email body before passing to the agent.
- The `Prepare AI Agent Context` node contains commented-out code for fetching `afinity_customer_id` — this can be re-enabled if your MongoDB stores customer IDs from an external system.
- WhatsApp payloads are parsed for both the Meta Cloud API format (`entry[0].changes[0].value.messages[0]`) and a simpler flat format (`{from, message}`).
