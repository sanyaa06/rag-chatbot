# RAG Chatbot — Archive

A retrieval-augmented generation (RAG) chatbot built on n8n, Qdrant, and the Gemini API. Users upload PDFs into a shared knowledge base, then ask questions answered using only that knowledge base's content.

## Architecture

The system is split into a static frontend and four independent n8n workflows.

```
                         ┌─────────────────────┐
                         │   index.html         │
                         │  (upload + chat UI)  │
                         └──────────┬───────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │ upload webhook      │ chat webhook
              ▼                     ▼
     ┌──────────────────┐   ┌──────────────────────────────┐
     │ upload-workflow    │   │ chat-workflow                │
     │ Webhook → Drive     │   │ Webhook → Embed query        │
     └────────┬───────────┘   │       → Search Qdrant         │
              │                │       → Build context         │
              ▼                │       → Generate answer (Gemini)│
   ┌────────────────────────┐  │       → Respond to Webhook     │
   │ ingestion-workflow       │  └──────────────────────────────┘
   │ Drive Trigger → Download │                 ▲
   │  → Convert PDF → Chunk    │                 │
   │  → Embed chunks → Upsert  │─────────────────┘
   └───────────┬────────────┘        reads from
               │ writes to
               ▼
     ┌───────────────────────┐
     │ Qdrant collection        │
     │ "knowledge_base"          │
     │ (created once via         │
     │ create-collection-workflow)│
     └───────────────────────┘
```

## Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| `create-collection-workflow.json` | Manual (run once) | Creates the Qdrant `knowledge_base` collection with the correct vector size and distance metric. Must run before ingestion. |
| `upload-workflow.json` | Webhook (POST, from frontend upload) | Receives a PDF from the frontend and saves it to Google Drive. |
| `ingestion-workflow.json` | Google Drive Trigger (fileCreated) | Downloads the new file, converts it via PDF.co, splits it into chunks, embeds each chunk with Gemini, and upserts the vectors + payload into Qdrant. |
| `chat-workflow.json` | Webhook (POST, from frontend chat) | Embeds the incoming question, searches Qdrant for relevant chunks, builds a context string, asks Gemini to answer using only that context, and returns the answer to the frontend. |

## Setup

### 1. Prerequisites

- An n8n instance (cloud or self-hosted)
- A Qdrant Cloud cluster (or self-hosted Qdrant)
- A Gemini API key ([Google AI Studio](https://aistudio.google.com))
- A PDF.co API key (used for PDF-to-text conversion during ingestion)
- A Google Drive account, connected in n8n

### 2. Import the workflows

In n8n: **Workflows → Import from File**, and import each of the four `.json` files in `/workflows`.

### 3. Configure credentials

None of the exported workflow files contain real API keys — every request that needs one currently has a `YOUR_..._API_KEY` placeholder. In each relevant HTTP Request node, either:
- Replace the placeholder with your key directly (fine for local/testing use), or
- Store the key as an n8n credential and reference it via the node's Authentication settings (recommended for anything shared or deployed)

Keys/config needed:
- **Gemini API key** — used in the embed and generate-answer nodes across `ingestion-workflow.json` and `chat-workflow.json`
- **Qdrant cluster URL + API key** — update the hardcoded cluster URL in `create-collection-workflow.json`, `ingestion-workflow.json`, and `chat-workflow.json` to match your own cluster
- **PDF.co API key** — used in `ingestion-workflow.json`'s conversion step
- **Google Drive OAuth credential** — connected via n8n's credential manager, used in both `upload-workflow.json` and `ingestion-workflow.json`

### 4. Run order

1. Run `create-collection-workflow.json` once, manually, to create the Qdrant collection.
2. Activate `upload-workflow.json` and `ingestion-workflow.json` (these should run continuously).
3. Activate `chat-workflow.json`.
4. Update `index.html`'s `UPLOAD_WEBHOOK_URL` and `CHAT_WEBHOOK_URL` constants with your n8n **production** webhook URLs (not the `/webhook-test/` ones — those only work while the n8n editor is open).
5. Open `index.html` in a browser, upload a PDF, wait for ingestion to complete, then ask a question in the chat panel.

## Models used

- **Embeddings**: `gemini-embedding-001` (Google's current embedding model — the older `text-embedding-004` was shut down January 2026), output dimension explicitly set to `768` to match the Qdrant collection config
- **Generation**: `gemini-3-flash-preview` (Google's current recommended free-tier chat model as of mid-2026)

Google has deprecated Gemini models fairly aggressively over the past several months — if either model above returns a 404, check [ai.google.dev/gemini-api/docs/changelog](https://ai.google.dev/gemini-api/docs/changelog) for the current replacement and update the model name in the relevant HTTP Request node(s).

## Notes on retrieval tuning

The Qdrant search step uses a `score_threshold` to filter out irrelevant matches. If the chatbot starts returning "I don't know" for questions it should be able to answer, the threshold may be set too high — lower it. If it starts answering from unrelated chunks, raise it. See the search node in `chat-workflow.json`.

## Security

- No workflow file in this repo contains real API keys — all committed as placeholders.
- Rotate any key that was ever typed directly into a URL/body during development rather than stored as an n8n credential, since it may have been logged, screenshotted, or shared during debugging.
- `index.html` exposes the webhook URLs publicly (they're not secrets, but they do reveal the n8n instance's endpoint) — consider adding auth to the webhook nodes if this is deployed beyond local/personal use.
