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

- Course and lesson management with structured content
- Reading progress tracking per client
- Retrieval-Augmented Generation (RAG) pipeline:
  - Lesson text is chunked and embedded into 384-dimensional vectors using a local ONNX model (`all-MiniLM-L6-v2`)
  - Queries are embedded the same way and matched via cosine similarity
  - Top passages are fed to the LLM alongside the question for citation-grounded answers
- Chat history preserved per course
- Mobile-friendly reading interface

---

## Architecture

```
┌─────────────────────┐      ┌──────────────────────────────────┐
│   React + Vite SPA  │      │     Node.js + Express API        │
│   (Cloudflare)      │ ◄──► │     (Render)                     │
│                     │      │                                  │
│  CourseList         │      │  /api/courses   /api/lessons     │
│  CourseDetail       │      │  /api/progress  /api/tutor/chat  │
│  LessonReader       │      │  /api/health                      │
│  TutorChat          │      │                                  │
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
| **Retrieval service** | Brute-force cosine similarity search across all chunks of the current lesson. No vector DB needed at portfolio scale. |
| **LLM service** | Sends the retrieved passages + user question to DeepSeek V4 Flash Free via OpenCode API. Instructs the model to answer only from the provided text and cite passages. |
| **Tutor route** | Orchestrates embed -> retrieve -> ask -> return, preserving chat history per client + course. |

---

## Stack

- **Backend:** Node.js, Express, MongoDB (Atlas), @xenova/transformers (ONNX), OpenCode API (DeepSeek V4 Flash Free)
- **Frontend:** React 18, Vite 6, Tailwind CSS, React Router
- **Infrastructure:** Render (backend), Cloudflare Workers (frontend), MongoDB Atlas (database)

---

## Status

Live at [marginalia.hams69.workers.dev](https://marginalia.hams69.workers.dev). Sample courses on REST APIs and React Hooks are pre-loaded for demonstration.
