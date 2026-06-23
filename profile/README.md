# Marginalia

**Read the lesson. Question it in the margin.**

Try it at [marginalia.hams69.workers.dev](https://marginalia.hams69.workers.dev)

---

## What is Marginalia?

Marginalia is an AI tutor built on a simple idea: the tutor should only answer from the lesson you're currently reading. No general knowledge, no off-topic guesses, no hallucination. Every response is grounded in the source text and cites the exact passages it draws from.

Named after the ancient practice of writing notes in the margins of books, Marginalia turns the margin into a conversation.

---

## How it works

1. **Pick a course** — browse the catalog of available lessons.
2. **Read a lesson** — each lesson is displayed alongside a chat panel.
3. **Ask questions** — type any question about the material.
4. **Get grounded answers** — the backend retrieves the most relevant passages from the lesson, sends them to an LLM together with your question, and returns an answer with citations back to the source.

Because the LLM only sees the retrieved lesson text (not its full training data), the answers stay on-topic and verifiable.

---

## Architecture decisions

A few choices worth calling out, since the reasoning matters more than the code for a portfolio piece:

**Embeddings stored as plain arrays in MongoDB, similarity computed in app code.** No Atlas Vector Search, no Pinecone/Chroma. At course-library scale (low thousands of chunks) brute-force cosine similarity runs in well under 50ms and keeps the entire stack to "just MongoDB" — one connection string, nothing extra to provision. The migration path if a library grew large is narrow and well-defined: `findRelevantChunks()` in `backend/src/services/retrieval.js` is the only place that would change, to an Atlas `$vectorSearch` aggregation stage.

**No LLM SDK.** `services/embeddings.js` and `services/llm.js` are plain `fetch()` calls against `LLM_BASE_URL`. OpenAI, Groq, DeepSeek, and most others expose an OpenAI-compatible `/chat/completions` and `/embeddings` shape, so the provider is an environment variable, not a dependency.

**Auth is optional.** The app started with anonymous `clientId` in `localStorage` (no login system) to get the core RAG demo working first. Real JWT accounts were layered on in Phase 4, with the data model accepting both `userId` and `clientId` so both paths coexist without migration.

**Hand-rolled chunking, not a text-splitter package.** `chunking.js` is a ~30-line paragraph-aware splitter with overlap. Worth owning outright rather than importing for something this size — and it's a better interview answer than "I imported langchain's splitter."

**Citations carry a snippet and a score, not just a lesson name.** The margin panel shows the actual excerpt the model was given, so you can sanity check whether the retrieval — not just the generation — got it right. Faithfulness you can eyeball per-answer instead of measuring after the fact.

---

## Project structure

```
marginalia/
  backend/
    src/
      models/        Course, Lesson, Chunk, Progress, ChatMessage, User
      services/      chunking, embeddings, llm, retrieval, bm25
      routes/        courses, lessons, progress, tutor, auth
      scripts/seed.js loads sample courses + embeds them
  frontend/
    src/
      context/       AuthContext
      pages/         CourseList, CourseDetail, LoginPage, RegisterPage
      components/    MarginTutor, LessonNav, Citation, Header, CourseCard
```

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

Live. Sample courses on REST APIs and React Hooks are pre-loaded for demonstration.

### What's been built

The six enhancements originally planned are now all implemented:

| Enhancement | Phase | Status |
|---|---|---|
| Multi-provider LLM fallback | 1 | Done |
| Hybrid retrieval (BM25 + vector cosine) | 1 | Done |
| Streaming tutor responses over SSE | 2 | Done |
| Citation source highlighting in lesson text | 2 | Done |
| Real accounts with JWT auth | 4 | Done |
| Responsive three-column reader layout | 3 | Done |
