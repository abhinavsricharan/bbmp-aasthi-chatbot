# BBMP Aasthi Chatbot — Detailed Architecture

## Overview

A locally-run, privacy-first AI chatbot for the BBMP Aasthi e-Khata portal.
It uses **Retrieval-Augmented Generation (RAG)** to answer citizen questions
grounded in real BBMP knowledge — no cloud APIs, no data leaves your machine.

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CITIZEN'S BROWSER                           │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Any Website (BBMP Aasthi or other)                          │  │
│  │                                                              │  │
│  │  <script src="http://localhost:8000/static/widget.js">       │  │
│  │                                                              │  │
│  │   ┌─────────────────────────────────────────────────────┐   │  │
│  │   │              CHAT WIDGET (widget.js)                │   │  │
│  │   │                                                     │   │  │
│  │   │  ┌────────────┐  ┌───────────────┐  ┌──────────┐  │   │  │
│  │   │  │ Quick Chips│  │  Chat Panel   │  │ Input Box│  │   │  │
│  │   │  │ (6 topics) │  │  (messages)   │  │ + Send   │  │   │  │
│  │   │  └────────────┘  └───────────────┘  └──────────┘  │   │  │
│  │   │                                                     │   │  │
│  │   │         WebSocket  ws://localhost:8000/ws/chat      │   │  │
│  │   └───────────────────────┬─────────────────────────────┘   │  │
│  └───────────────────────────┼──────────────────────────────────┘  │
└──────────────────────────────┼──────────────────────────────────────┘
                               │ WebSocket (streaming tokens)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    FASTAPI BACKEND  (main.py)                       │
│                                                                     │
│   GET  /              → Demo HTML page                              │
│   GET  /health        → Health check                                │
│   GET  /static/*      → widget.js, widget.css                      │
│   WS   /ws/chat       → Streaming chat endpoint                     │
│                                                                     │
│   Per-connection state:                                             │
│   • conversation_history[]  (last 10 turns kept in memory)          │
│   • Streams tokens back to browser as they are generated            │
│                                                                     │
│   Message types sent to browser:                                    │
│   • { type: "bot_message" }   → welcome message                     │
│   • { type: "stream_start" }  → begin new response                  │
│   • { type: "stream_chunk" }  → one token of the response           │
│   • { type: "stream_end" }    → response complete                   │
│   • { type: "error" }         → something went wrong                │
└──────────────────┬──────────────────────────────────────────────────┘
                   │ calls rag.py
                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      RAG PIPELINE  (rag.py)                         │
│                                                                     │
│  Step 1 — RETRIEVE                                                  │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  User query → OllamaEmbeddings (nomic-embed-text)          │    │
│  │             → 768-dim vector                               │    │
│  │             → ChromaDB similarity search                   │    │
│  │             → Top 5 matching text chunks returned          │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  Step 2 — AUGMENT                                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  System Prompt  (BBMP-specific rules + helpline)           │    │
│  │  + Retrieved context chunks (top 5)                        │    │
│  │  + Conversation history (last 8 turns)                     │    │
│  │  + Current user question                                   │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  Step 3 — GENERATE                                                  │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  Ollama (mistral model) → streaming token generator        │    │
│  │  Yields tokens one-by-one back to main.py WebSocket        │    │
│  └────────────────────────────────────────────────────────────┘    │
└──────────────────┬────────────────────────┬─────────────────────────┘
                   │                        │
                   ▼                        ▼
┌──────────────────────────┐   ┌────────────────────────────────────┐
│   ChromaDB  (chroma_db/) │   │   Ollama  (local process)          │
│                          │   │                                    │
│  Collection:             │   │  Models:                           │
│  bbmp_knowledge          │   │  • mistral   → text generation     │
│                          │   │  • nomic-    → embeddings          │
│  24 embedded chunks      │   │    embed-    (768 dimensions)      │
│  from BBMP FAQ knowledge │   │    text                            │
│                          │   │                                    │
│  Stored on disk,         │   │  Runs as a local server on         │
│  no cloud needed         │   │  port 11434 (default)              │
└──────────────────────────┘   └────────────────────────────────────┘
```

---

## Data Flow — Step by Step

### When a User Asks a Question

```
1. User types question OR clicks quick chip
        │
        ▼
2. widget.js sends  { content: "What is e-Khata?" }
   over WebSocket to FastAPI
        │
        ▼
3. FastAPI /ws/chat receives the message
   Appends it to conversation_history[]
        │
        ▼
4. Calls  rag.stream_response(query, history)
        │
        ▼
5. RAG — retrieve_context():
   • Embeds the query using nomic-embed-text
   • Searches ChromaDB for the 5 most similar chunks
   • Returns relevant FAQ/knowledge text
        │
        ▼
6. RAG — build_messages():
   • Combines system prompt + context + history + question
   • Creates the full prompt list for the LLM
        │
        ▼
7. RAG — ollama.chat(stream=True):
   • Sends prompt to local mistral model
   • Receives tokens one by one
        │
        ▼
8. FastAPI streams each token back as:
   { type: "stream_chunk", content: "e-Khata " }
   { type: "stream_chunk", content: "is a..." }
   ...
        │
        ▼
9. widget.js appends each token to the bot bubble in real time
   (the answer "types itself" live on screen)
        │
        ▼
10. { type: "stream_end" } → input box re-enabled
    Question label shown above the bot reply
```

---

## One-Time Setup Data Flow (Ingestion)

```
scraper.py          ingest.py               ChromaDB
    │                   │                      │
    │  Save raw text     │                      │
    │─────────────────►  │                      │
    │  knowledge_base/   │  Split into chunks   │
    │  *.txt             │──────────────────►   │
    │                    │  (500 chars, 50       │
    │                    │   overlap)            │
    │                    │                      │
    │                    │  Embed each chunk     │
    │                    │  (nomic-embed-text)   │
    │                    │──────────────────►   │
    │                    │  768-dim vectors      │
    │                    │                      │
    │                    │  Store in ChromaDB    │
    │                    │──────────────────►   │
    │                    │  (persistent on disk) │
```

---

## File-by-File Breakdown

| File | Layer | Responsibility |
|---|---|---|
| `scraper.py` | Data | Scrapes BBMP Aasthi FAQs + hand-crafted knowledge base (300+ lines covering all services). Saves to `knowledge_base/*.txt` |
| `ingest.py` | Data | Loads text files → splits into 500-char chunks → embeds with `nomic-embed-text` → stores in ChromaDB at `chroma_db/` |
| `rag.py` | AI Core | Retrieves top-5 context chunks from ChromaDB → builds prompt with system rules + history → streams `mistral` LLM response token-by-token |
| `main.py` | Backend | FastAPI server. Handles WebSocket `/ws/chat`, maintains per-connection conversation memory, streams tokens to browser. Also serves static files |
| `static/widget.js` | Frontend | Self-contained chat widget. Connects via WebSocket, renders streaming tokens, shows quick chips, question labels, typing indicator. One `<script>` tag embeds it |
| `static/widget.css` | Frontend | BBMP blue (`#0d47a1`) themed styles. Floating button, slide-up panel, message bubbles, scrolling chips strip |

---

## Key Design Decisions

### 1. RAG (Not Pure LLM)
The LLM (`mistral`) alone would hallucinate incorrect BBMP-specific details.
RAG forces the model to answer **only from retrieved BBMP knowledge**, making responses accurate and grounded.

### 2. Local-Only (Ollama)
No data is sent to any cloud API. All processing — embedding and generation — happens on the local machine. Critical for government data privacy compliance.

### 3. WebSocket Streaming
Instead of waiting for the full response, tokens stream to the browser as they are generated. This makes the UI feel responsive even when the model takes 30–60 seconds to generate.

### 4. Per-Connection Memory
Each WebSocket connection maintains its own `conversation_history[]`, keeping the last 10 turns. This allows multi-turn conversations (e.g., follow-up questions) while preventing memory from growing unbounded.

### 5. Embeddable Widget
The entire frontend is a single `<script>` tag that injects itself. No changes to the host website's HTML structure are needed — it just floats in the bottom-right corner of any page.

### 6. Helpline Fallback
The system prompt instructs the LLM: if no relevant context is found or confidence is low, always suggest contacting the BBMP helpline **9480683695** rather than guessing.

---

## Technology Stack Summary

| Component | Technology | Why |
|---|---|---|
| LLM Runtime | Ollama | Local, free, easy model management |
| LLM Model | mistral | Best balance of speed + accuracy for 8GB RAM |
| Embedding Model | nomic-embed-text | High quality, local, 768-dim |
| Vector Store | ChromaDB | Zero-config, persistent, local |
| RAG Framework | LangChain + langchain-chroma | Standard Python RAG tooling |
| Backend | FastAPI + uvicorn | Fast async Python, built-in WebSocket support |
| Web Scraping | httpx + BeautifulSoup4 | Lightweight HTML parsing |
| Frontend | Vanilla JS + CSS | Zero dependencies, works on any website |

---

## Ports & Services

| Service | Port | Notes |
|---|---|---|
| FastAPI server | `8000` | `uvicorn main:app --port 8000` |
| Ollama | `11434` | Started automatically by Ollama |
| ChromaDB | *(in-process)* | Embedded, no separate server needed |

---

## How to Extend

### Add More Knowledge
1. Edit `scraper.py` → add more text to `MANUAL_KNOWLEDGE`
2. Re-run `python scraper.py` then `python ingest.py`
3. Restart the server

### Change the LLM Model
Edit `rag.py`:
```python
LLM_MODEL = "llama3.1:8b"   # or any model pulled via ollama pull
```

### Add Kannada Support
Modify the system prompt in `rag.py` to detect and respond in Kannada:
```python
"If the user writes in Kannada, respond in Kannada."
```
And pull a multilingual model like `aya:8b`.

### Deploy to a Server
Replace `ws://localhost:8000` in `widget.js` with your server's public address, and run uvicorn behind a reverse proxy (nginx) with SSL.
