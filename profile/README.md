# Marginalia 🧠📝

**Read the lesson. Question it in the margin.**

Marginalia is an AI tutor that answers questions strictly from the lesson text you're reading — no general knowledge, no hallucination. Every response is grounded and cited.

## Repositories

| Repo | Description |
|------|-------------|
| [backend](https://github.com/marginalia-tutor/backend) | Node.js + Express API, MongoDB, local ONNX embeddings, LLM-powered RAG |
| [frontend](https://github.com/marginalia-tutor/frontend) | React + Vite + Tailwind SPA |

## Stack

- **Backend:** Node.js, Express, MongoDB (Atlas), @xenova/transformers (ONNX), OpenCode API (DeepSeek V4 Flash Free)
- **Frontend:** React 18, Vite 6, Tailwind CSS, React Router
- **Infrastructure:** Render (backend), Cloudflare Pages/Workers (frontend)

## Philosophy

Marginalia takes its name from the ancient practice of writing notes in the margins of books. The tutor doesn't answer from its general knowledge — it only answers from the lesson you're currently reading, with direct citations back to the source text.
