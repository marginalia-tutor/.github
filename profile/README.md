# Marginalia

**Read the lesson. Question it in the margin.**

Try it at [marginalia.hams69.workers.dev](https://marginalia.hams69.workers.dev)

---

## What is Marginalia?

Marginalia is an AI tutor built on a simple idea: the tutor should only answer from the lesson you're currently reading. No general knowledge, no off-topic guesses, no hallucination. Every response is grounded in the source text and cites the exact passages it draws from.

Named after the ancient practice of writing notes in the margins of books, Marginalia turns the margin into a conversation.

---

## How it works

1. **Pick a course** -- browse the catalog of available lessons.
2. **Read a lesson** -- each lesson is displayed alongside a chat panel.
3. **Ask questions** -- type any question about the material.
4. **Get grounded answers** -- the backend retrieves the most relevant passages from the lesson, sends them to an LLM together with your question, and returns an answer with citations back to the source.

Because the LLM only sees the retrieved lesson text (not its full training data), the answers stay on-topic and verifiable.

---

## Features

- **Course reader** with structured lessons, progress tracking, and completion indicators
- **RAG pipeline** — lesson text is chunked and embedded (384-dim ONNX vectors), questions are retrieved via hybrid search (cosine similarity + BM25 scoring), and the LLM generates grounded, cited answers
- **SSE streaming** — tutor responses stream token-by-token instead of waiting for the full answer
- **Citation source highlighting** — hovering a citation highlights the exact source sentence in the lesson text
- **Multi-provider LLM fallback** — configure a secondary LLM endpoint; if the primary errors, the fallback is tried transparently
- **JWT auth** — real accounts with email/password login, co-existing with anonymous access (no migration needed)
- **Conversation memory** — last 6 turns sent as context; full history preserved per course
- **Responsive layout** — three-column reader collapses to mobile-friendly slide-over panels below tablet width

---

## Architecture

```
┌─────────────────────┐      ┌──────────────────────────────────┐
│   React + Vite SPA  │      │     Node.js + Express API        │
│   (Cloudflare)      │ ◄──► │     (Render)                     │
│                     │      │                                  │
│  CourseList         │      │  /api/courses   /api/lessons     │
│  CourseDetail       │      │  /api/progress  /api/auth/*      │
│  LessonReader       │      │  /api/tutor/chat                 │
│  TutorChat          │      │  /api/tutor/chat/stream          │
│  Auth (login/reg)   │      │  /api/health                     │
└─────────────────────┘      └──────────┬───────────────────────┘
                                        │
                         ┌──────────────┴──────────────┐
                         │         MongoDB Atlas        │
                         │                              │
                         │  courses │ lessons           │
                         │  progress │ chat_history     │
                         │  embeddings (raw vectors)    │
                         └──────────────────────────────┘
```

Key components in the backend:

| Component | Role |
|-----------|------|
| **Embeddings service** | Runs a quantized ONNX model locally via @xenova/transformers. Produces 384-dim vectors for both lesson chunks and user queries. No external API calls. |
| **Retrieval service** | Hybrid search combining cosine similarity (env-configurable weight, default 0.6) with BM25 Okapi scoring (weight 0.4). Scores are min-max normalized and the top-K chunks are returned. No vector DB needed at portfolio scale. |
| **LLM service** | Multi-provider chat completions via plain `fetch()` to any OpenAI-compatible endpoint. Supports a secondary fallback provider on error, configurable timeout per provider, and SSE streaming. Temperature 0.3, max 2000 tokens. |
| **Auth middleware** | Optional JWT verification — attaches `req.user` if a valid Bearer token is present, silently continues for anonymous users. Both `userId` and `clientId` paths are supported throughout the data model. |
| **Tutor route** | Orchestrates embed -> hybrid retrieve -> LLM generate, with both `/chat` (full response) and `/chat/stream` (SSE tokens) endpoints. Persists conversation history. |

---

## Stack

- **Backend:** Node.js, Express, MongoDB (Atlas), @xenova/transformers (ONNX), OpenCode API (DeepSeek V4 Flash Free), bcryptjs, jsonwebtoken
- **Frontend:** React 18, Vite 6, Tailwind CSS, React Router
- **Infrastructure:** Render (backend), Cloudflare Workers (frontend), MongoDB Atlas (database)

---

## Status

Live. All six planned enhancements are implemented: multi-provider LLM fallback, hybrid retrieval, SSE streaming, citation source highlighting, JWT auth, and responsive layout. Sample courses on REST APIs and React Hooks are pre-loaded for demonstration.
