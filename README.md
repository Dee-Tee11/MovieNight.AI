# MovieNight.AI

A movie recommender that learns your taste from your ratings, plus an assistant that helps you pick
tonight's film using your profile, your ratings and what your friends watched.

> **Status: planning.** The architecture and the roadmap are written; there is no code yet. v2 is a
> backend rewrite of CinemaWebApp (the previous version, kept next to this folder).

## What it will do

- **Learn your taste.** Every rating shapes a profile: likes pull, dislikes push. Recommendations
  come with a reason ("because you liked A and B, same director as C, loved by Ana").
- **Answer in chat.** An agent with read-only tools can search by mood or description (Portuguese or
  English), suggest what to watch tonight, check what your friends watched, and ask a clarifying
  question when your request is vague. It only cites films that exist in the catalogue.
- **Use your friends.** Accepted friends' ratings and activity feed both the recommendations and the
  assistant.

## How it works (short)

1. **Offline pipeline.** A ~1M-title Kaggle dataset is filtered to a ~50k-title serving catalogue
   and embedded on a GPU with Qwen3-Embedding-0.6B. The result is a versioned catalogue file.
2. **API.** A FastAPI service loads the catalogue into RAM and does exact search. Users sign in with
   Google, and the API issues its own session token. Ratings and friendships live in Supabase
   Postgres, which only the API talks to (no Edge Functions).
3. **Recommendation engine.** Several candidate generators (your taste vector, neighbours of your
   favourites, friends' picks, same director or cast) feed a feature ranker. Its weights start from
   a global prior and personalise as you rate.
4. **Agent.** `POST /agent/chat` streams Server-Sent Events. A tool-calling loop runs on a chain of
   Groq models, picked by a scripted agent eval. Quick actions such as "what should I watch
   tonight?" skip the LLM entirely.

The full design, with the reasoning behind every decision, is in [ARCHITECTURE.md](ARCHITECTURE.md).

## Stack

| Layer | Choice | State |
|---|---|---|
| Backend | Python, FastAPI, SSE | decided |
| Auth | Google Sign-In → backend session JWT | decided |
| Data | Supabase Postgres (users, ratings, friends only) | decided |
| Embeddings | Qwen3-Embedding-0.6B (multilingual, needs a ≥ 2 GB host) | decided, runtime to be proven in Phase 0 |
| Catalogue | in-RAM artifact, exact search | decided |
| Engine | multi-generator retrieval → per-user feature ranker | decided, built in layers |
| LLM | Groq: `qwen/qwen3.8-27b`, `openai/gpt-oss-120b`, `gpt-oss-20b` as fallback | chosen by the agent eval in Phase 0 |
| Frontend | React, same aesthetic as before | later |

## Planned layout

```
MovieNight.AI/
├── api/            FastAPI app: auth, db, llm, agent, tools, catalogue, engine/, routers/
├── pipeline/       offline: ingest, embed (GPU), eval
├── supabase/       schema, constraints, triggers
├── docs/           dataset notes and spike results
├── web/            frontend, later (refactored against the OpenAPI contract)
├── ARCHITECTURE.md
├── ROADMAP.md
└── README.md
```

Only the three Markdown files exist today.

## Docs

| File | What is in it |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Decisions, diagram, auth, data model, catalogue, engine, agent, risks |
| [ROADMAP.md](ROADMAP.md) | The 39 planned commits in order, the TODO list and the open questions |

## Where we are

- [x] Architecture written
- [x] Roadmap written (commits, TODO, open questions)
- [ ] Blocking items before the first commit: backend host (≥ 2 GB RAM), the Kaggle CSV header and
  licence line, a check of Groq's live model list, a Google OAuth client id, a Supabase project, a
  GitHub repo, a GPU notebook. See [ROADMAP.md](ROADMAP.md) section 2.
- [ ] Phase 0 spikes: embedding runtime, LLM choice, engine prototype

## Running it

Nothing to run yet. The first commits (see the roadmap) initialise the repository and scaffold the
API. Configuration will come from environment variables (Supabase service key, `SESSION_SECRET`,
Groq key, Google client id). Secrets and generated artifacts are never committed.

## Conventions

Conventional Commits. Every commit leaves lint and tests green and updates `ARCHITECTURE.md` when it
changes a decision.

## Data and attribution

- Dataset: Kaggle "The Ultimate 1Million Movies Dataset (TMDB + IMDb)"
  (`alanvourch/tmdb-movies-daily-updates`). Its licence still has to be checked.
- Movie metadata and posters come from TMDB. This product uses the TMDB API but is not endorsed or
  certified by TMDB.
- IMDb's datasets are for non-commercial use. Sort out all licences before monetising.

## Licence

Not chosen yet.
