# OmniChat

> **One chat window. Many open models. No dead ends.**

OmniChat is an open-source AI chat application that runs on free and open-weight
models — NVIDIA Nemotron, Qwen, GLM, DeepSeek, Muse Glimmer and others. The
conversation lives in OmniChat's database, not inside any one model, so when a
provider hits a rate limit or its context window runs out, the backend **Relay**
hands the conversation to another model automatically and visibly. Your chat
never stops because a limit was reached.

**Status:** early development (v0.1 scaffold). The complete product specification
lives in [`docs/DESIGN.md`](docs/DESIGN.md).

---

## Why OmniChat?

Anyone who relies on AI chat daily hits three walls:

1. **Usage limits** — a free plan ends mid-task and you start over somewhere else.
2. **Context limits** — long chats degrade or refuse when the window fills up.
3. **Fragmentation** — every assistant keeps its own history, files and memory.

OmniChat answers with a single idea: **the conversation belongs to you, not to
the model.** Because model APIs are stateless, the same history can be sent to a
different model at any moment — so a rate limit becomes a brief handoff, not a
dead end.

## What makes it different

- **The Relay** — automatic, visible, undoable handoff between models. A thin
  thread in the UI marks every switch, with the reason, and one click takes you
  back. No silent model swaps, ever.
- **Eval-driven routing** — a built-in evaluation harness (`evals/`) continuously
  measures which open models are actually good at what (code, maths, Hindi,
  long docs…), and those scorecards feed the router. Routing decisions are
  explainable: every reply can show *why* a model was chosen.
- **Privacy-first** — memory is opt-in and editable, you always see which
  provider processed which message, and routes that train on your data can be
  excluded entirely. Local-only mode for sensitive chats.
- **BYOK-first** — bring your own provider keys and per-user limits apply instead
  of a shared pool. No account juggling, no re-explaining.
- **Open source** — the whole stack, self-hostable with one `docker compose up`.

## Features

**Chat**
- Streaming responses, stop / regenerate / edit with branching, auto-titles,
  drafts, incognito chats, message feedback
- Rich Markdown: code blocks with run buttons, KaTeX math, Mermaid diagrams,
  tables, citations with a Sources panel
- File Q&A: PDF, DOCX, spreadsheets, images, audio, zip — with hybrid
  retrieval (BM25 + vector) and page-level citations
- Projects: shared instructions, knowledge bases, per-project memory
- Artifacts: HTML pages, React components, documents and diagrams rendered in a
  sandboxed side panel with versioning and one-click publishing

**Relay & routing**
- Multi-provider gateway over OpenAI-compatible endpoints (NVIDIA NIM,
  OpenRouter, Groq-style providers, local Ollama, …)
- Health tracking + circuit breaker per route; token-bucket quota accounting
- Mid-stream failure continuation, context-aware compaction, 1M-token routes
- Per-conversation model pinning, preferred fallback chains, quality guards

**Control**
- Model picker with live health dots, compare mode (2–3 models side by side)
- Context meter showing exactly what fills the window, with one-click compact
- Full settings: themes, density, fonts, Hindi/English UI, keyboard shortcuts,
  usage dashboard ("you were rescued from N rate limits this month")

**For developers**
- OpenAI-compatible API (`/v1/chat/completions`) exposing the router as one model
- MCP client + native connectors (Drive, Gmail, Calendar, GitHub, Notion)
- Typed SSE event protocol, idempotent writes, cursor pagination

## Architecture

```
                    ┌──────────────────────────────┐
                    │   Browser / PWA (Next.js)    │  web/
                    └──────────────┬───────────────┘
                       HTTPS (REST + SSE)
                    ┌──────────────▼───────────────┐
                    │  Edge: CDN + WAF + rate limit │
                    └──────────────┬───────────────┘
                    ┌──────────────▼───────────────┐
                    │       API service (FastAPI)   │  api/
                    │  auth · chats · files · admin │
                    └───┬────────┬─────────┬───────┘
                        │        │         │
        ┌───────────────▼┐  ┌────▼─────┐  ┌▼────────────────┐
        │ LLM Gateway     │  │ Workers  │  │ Realtime hub     │  api/app/gateway/
        │ router+adapters │  │ (queue)  │  │ (WS/SSE fan-out) │  worker/
        └───┬───┬───┬────┘  └────┬─────┘  └──────────────────┘
            │   │   │            │
   NVIDIA NIM  │  Local Ollama   ├─ ingest (parse, OCR, chunk, embed)
      OpenRouter  Groq/others    ├─ summariser / title / memory extractor
                                 ├─ code sandbox runner
                                 ├─ email, webhooks, exports
   ┌──────────────┬──────────────┴───┬──────────────┬──────────────┐
   │ PostgreSQL    │ Redis            │ Object store  │ Vector index  │
   │ (+pgvector)   │ cache/queue/     │ (S3/MinIO/R2) │ (pgvector)    │
   │               │ rate limits      │               │               │
   └──────────────┴──────────────────┴──────────────┴──────────────┘
```

## Quickstart

**Prerequisites:** Docker + Docker Compose, Git.

```bash
git clone https://github.com/YOUR_USERNAME/omnichat.git
cd omnichat
cp .env.example .env      # add provider keys later; not required to start
docker compose up --build
```

| Service | URL |
|---|---|
| Web app | http://localhost:3000 |
| API | http://localhost:8000 |
| API docs (OpenAPI) | http://localhost:8000/docs |
| MinIO console | http://localhost:9001 |

Without provider keys the gateway runs against a **mock provider with
programmable failures** (429s, slow tokens, mid-stream death) — enough to develop
and demo the Relay end to end. Add `NVIDIA_API_KEY` / `OPENROUTER_API_KEY` to
`.env` when you want live models. Keys never leave the server; see
[`docs/DESIGN.md`](docs/DESIGN.md) §B4.3.

> **Note on free tiers:** provider rate limits, model availability and terms of
> service change often — and some free tiers restrict how they may be used.
> Model IDs and limits live in config (`api/app/gateway/registry/`), never in
> code, and must be re-verified before any public launch.

## Project structure

| Path | Contents |
|---|---|
| `web/` | Next.js frontend — spec Section A (UI/UX) |
| `api/` | FastAPI backend: auth, chats, files, projects, admin |
| `api/app/gateway/` | The heart: router policy, provider adapters, model registry, health/circuit breaker, Relay engine — spec §B4–B5 |
| `worker/` | Background jobs: ingest, summarise, memory, research, email |
| `evals/` | Model evaluation harness + datasets — feeds the router's quality scores (§B28.4) |
| `contracts/` | API contracts; the TS client is generated from the OpenAPI spec |
| `infra/` | Dockerfiles, deployment manifests |
| `docs/` | Full product spec (`DESIGN.md`) and architecture decision records (`adrs/`) |

## Roadmap

| Phase | Focus |
|---|---|
| P0 | Foundations: repo, tokens, auth skeleton, DB schema, CI, Compose |
| P1 | MVP chat: streaming, sidebar, history, composer, markdown, themes |
| P2 | **Router & Relay**: adapters, registry, health/breaker, fallback, model picker, context meter |
| P3 | Files & knowledge: uploads, parsing, RAG with citations, projects |
| P4 | Tools & artifacts: web search, code sandbox, artifact panel, deep research |
| P5 | Personalisation: memory, personas, BYOK, usage dashboard |
| P6 | Integrations: Drive/GitHub connectors, MCP client |
| P7 | Polish & launch: PWA, a11y audit, Hindi i18n, sharing, admin, security review |
| v2 | Voice mode, Code tab, team workspaces, billing, E2EE vault chats |

The 30-second demo this project is built around: kill a provider mid-stream and
watch the answer continue on the next model with a Relay marker. That is the
whole pitch.

## Contributing

Behaviour changes start in `docs/DESIGN.md`, not in code. Significant
architectural decisions get a short ADR in `docs/adrs/`. Keep PRs small, add
tests (unit, contract tests against the mock provider, Playwright for UI flows),
and keep CI green. Full guide: [`CONTRIBUTING.md`](CONTRIBUTING.md).

## Security

Do not open public issues for vulnerabilities — see [`SECURITY.md`](SECURITY.md).

## License

TBD — MIT or Apache-2.0, to be chosen before the first public release.
