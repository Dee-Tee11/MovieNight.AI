# MovieNight.AI — Target Architecture (v2)

Status: proposed. Architecture only, no code. Companion file: `ROADMAP.md` (planned commits, TODO
list and open questions).

## Context

A movie recommender with a better recommendation engine and an agent (RAG + tools) that helps pick
the next film. It is a backend rewrite of CinemaWebApp (sibling folder).

Confirmed by you:
- **Auth:** Google sign-in. Not Supabase Auth, and Clerk is gone.
- **Backend:** FastAPI as in your diagram. No Supabase Edge Functions.
- **Database:** the old structure is a starting point, not a constraint. It changes wherever that
  makes things better.
- **Dataset:** Kaggle "The Ultimate 1Million Movies Dataset (TMDB + IMDb)".
- **Embedding model:** **Qwen3-Embedding-0.6B**.
- **LLM:** Groq Qwen **if it fits better** than gpt-oss-120b. I read "Qwen3/8B" as
  `qwen/qwen3.8-27b`, the only Qwen on Groq's current lists (preview). Say so if you meant another
  model. A scripted agent eval decides (Phase 0).
- **ML engine and recommender:** free to be redesigned.
- **Agent:** the design below stays (your diagram plus one tool).
- **Frontend:** out of scope for now. Same aesthetic later, but it needs a refactor against the new
  API.
- Also confirmed: FastAPI stays, clean start for users, hobby scale on free tiers, v1 = solo +
  friends as a signal (group "movie night" sessions are later).

## Target architecture

Changes vs your diagram (**Δ**): the model box is a chain of Groq models behind a limiter; a new tool
`get_recommendations`; quick actions that skip the LLM; tools sit on a catalogue and an engine, and
Postgres holds only user data.

```
Browser  (frontend later: same look, refactored against the OpenAPI contract)
   │  Google Sign-In → ID token ──► POST /auth/google ──► app session JWT
   │  Authorization: Bearer <session> on every call — the browser never touches the DB
   ▼
FastAPI — one container, one worker  (host ≥ 2 GB RAM)
   ├─ REST  /auth/google · /me · /me/movies · /me/recommendations · /friends/* · /movies/search
   └─ SSE   POST /agent/chat ─► loop ≤ 8 steps ─► llm.py ─► Groq: Qwen3.8-27B / gpt-oss-120b / -20b
                  │  quick actions skip the LLM          └ per-model limiter (8K TPM / 200K TPD)
                  ▼
              tools.py   8 read-only tools · user_id comes from the session
                  │
        ┌─────────┴──────────────────────────────┐
        ▼                                        ▼
   catalogue.py  (in RAM)                    engine/  generators → features → ranker
   E[N×d] + metadata + facets                taste profile · seed neighbours · friends
   Qwen3-Embedding-0.6B (query side)         · quality · personal feature weights
        ▲                                        │
        │ loaded at startup                      ▼
   catalogue artifact                       db.py (service key)
   (baked into the image,                        │
    or fetched from Storage)                     ▼
        ▲                                 Supabase Postgres — data only
        │ publish                         profiles · user_movies · friendships · movies (slim)
OFFLINE (GPU notebook, e.g. Kaggle/Colab):
   Kaggle CSV (~1M) → serving filter → embedding text → Qwen3-Embedding-0.6B on GPU
   → artifact + slim `movies` upsert → eval (weak-label retrieval, personas, agent eval)
```

Request flows:
- **"What should I watch tonight?"** (quick action) → `get_recommendations` directly, no LLM → cards.
- **Free-form chat** → `POST /agent/chat` → LLM calls tools → answer cites movie ids → SSE tokens +
  a `movies` event → cards come from the catalogue.
- **For You** → `GET /me/recommendations` → engine, computed on request (ms).
- **Rating a film** → `PUT /me/movies/{id}` → Postgres.

## Key decisions

| # | Decision | Why | Revisit when |
|---|---|---|---|
| 1 | **Browser → FastAPI only.** Supabase is just Postgres: no Edge Functions, no browser access, RLS on with no policies | One authorisation gate. Google tokens are not Supabase tokens. Ends the unwritten-RLS debt | Realtime features |
| 2 | **Google Sign-In; the backend exchanges the Google ID token for its own session JWT** | Standard pattern. The old `auth.py` JWKS code carries over | A second provider → add an identities table |
| 3 | **Catalogue artifact held in API RAM** (vectors + metadata + facets), exact search. Postgres keeps no vectors | The engine does many similarity operations per request, which is ms in numpy vs hundreds of ms of SQL round trips. Removes pgvector, ANN tuning and the RPC | > ~200k titles, or a host without RAM → pgvector HNSW |
| 4 | **Serving catalogue ≈ 50k titles** (up to ~100k at d=512 on a 2 GB host) filtered from ~1M | RAM and quality: obscure titles hurt recommendations | Bigger host |
| 5 | **Embedding model: Qwen3-Embedding-0.6B (decided).** Multilingual, instruction-aware, Matryoshka dims (32–1024) so `d` is a knob | Portuguese queries work directly against English plots. Costs: a GPU for the corpus, a ≥ 2 GB host, slower CPU queries (see Catalogue) | The host cannot carry it → `bge-small-en-v1.5` (the artifact manifest names the model; swap = rebuild + restart) |
| 6 | **Δ Engine = multi-generator retrieval → per-user feature ranker**, shrunk to a global prior and built in layers | Goes beyond one mean vector; explainable; works with few ratings | Real ratings data → learned global ranker |
| 7 | **DB changes, each earning its place** (see Data model) | You allowed it | — |
| 8 | **Agent as designed:** hand-rolled loop, stateless server; the client sends the last ≤ 12 text turns; `ask_user` ends the stream and the next message is the answer | No state, no resume logic | You want history across devices → persist (2 tables) |
| 9 | **Δ LLM chosen by a scripted agent eval** between `qwen/qwen3.8-27b` and `openai/gpt-oss-120b`. Ordered fallback chain across both plus `gpt-oss-20b`. Model names, effort settings and limits are config, behind one seam `llm.py` with a per-model token-bucket limiter | The lineup churns (finding 5). Limits are per model, so the chain also pools quota (up to ~3 × 200K tokens/day) | Free quota bites → a Claude Haiku 5.5 adapter (list price $0.10 in / $0.50 out per 1M tokens, ≈ $1–2 per 1,000 turns by my estimate) |
| 10 | **Δ Quick actions skip the LLM**, plus a per-user daily cap | The daily token cap is a few dozen agent turns per model pool | Paid LLM |
| 11 | **Explanations come from the ranker's feature contributions** ("because you liked A and B, same director as C, loved by Ana"), templated | No tokens, cannot hallucinate | Users want richer "why" |
| 12 | **Only numbers (never review text or emails) from other users reach the LLM**; overviews are truncated | Reviews and TMDB overviews are user-written → prompt-injection surface | — |
| 13 | **API-first.** FastAPI publishes OpenAPI; the later frontend refactor generates typed clients from it. Keep the card JSON shape of `db.to_card_item` so existing UI cards still render | The frontend is a separate, later effort | — |

**LLM selection rule (decision 9):** ≥ 95% valid tool calls first, then the fewest tokens per turn,
and a tie goes to the production model. My expectation, to be tested: Qwen wins on tokens (parallel
calls, `reasoning_effort=none` on routing steps) and gpt-oss wins on stability (Qwen is a preview that
can vanish at short notice). If Qwen wins it becomes primary, with gpt-oss as fallback.

## Auth (Google)

- **Flow (frontend later):** Google Identity Services returns an ID token → `POST /auth/google
  {credential}`.
- **Verification:** the API checks the signature against Google's JWKS
  (`https://www.googleapis.com/oauth2/v3/certs`, PyJWT's `PyJWKClient`, already a dependency), `iss`,
  `aud` = your OAuth client id, `exp` and `email_verified`. It then upserts `profiles` (id = Google
  `sub`) and returns a session JWT (HS256, `SESSION_SECRET`, ~14-day TTL, no refresh endpoint).
- **Every other endpoint:** `Authorization: Bearer <session>`. The user id comes only from the token,
  never from the path, query or body (same rule as today).
- **Revocation:** rotate `SESSION_SECRET` to log everyone out. A per-user `token_epoch` column comes
  later if needed.
- **Account deletion:** delete the profile row (cascades). The Google account is untouched. There is
  no external service to call any more.
- **Google console:** web OAuth client with authorised JS origins. Publish the consent screen to
  production (basic scopes should need no verification; confirm).
- **Stored:** only `sub`, email, name, picture.
- **Optional allowlist** (`ALLOWED_EMAILS`) to protect the free LLM quota while the app is
  invite-only.

## Data model

Live DB dump first, for reference. Clean start, so there is no data migration.

| Table | Key columns | Change vs CinemaWebApp, and why |
|---|---|---|
| `profiles` | `id` (Google sub, text pk), `email`, `name`, `picture`, `taguser` unique, `onboarding_status`, `created_at` | Renamed from `"User"` (the quoted capital-U name is a footgun). `picture` added |
| `movies` (slim) | `id` (TMDB id, pk), `title`, `year` | Vectors and display metadata move to the artifact (decision 3). The table stays for FKs and ad-hoc SQL |
| `user_movies` | `user_id`, `movie_id`, `status` ('saved'/'watching'/'seen'), `rating`, `review`, `created_at`, **`updated_at`** | `updated_at` for cache invalidation and friend-activity "when". CHECKs: statuses; rating range and `rating IS NULL` unless `seen` (finding 2 becomes impossible) |
| `friendships` | `user_a`, `user_b` (a < b, unique pair), `requested_by`, `status` ('pending'/'accepted'/'declined'), `created_at` | **Merges `friend_request` + `friendships`**. "Accepted friends" cannot drift (the old accept flow wrote two tables with no unique pair) |
| `rec_events` (optional) | `user_id`, `movie_id`, `surface`, `generator`, `position`, `score`, `created_at` | New append-only impression log. Outcomes are derived by joining `user_movies`. This is what lets the engine be measured and learned on real use later |

Dropped: `user_recommendations` (computed on request, in-process cache keyed by user +
`max(updated_at)`).

## Catalogue and embeddings

**Source:** Kaggle `alanvourch/tmdb-movies-daily-updates`. **You paste the CSV header and the
licence line, and I finalise the mapping.**

**Serving filter** (config): released, not adult, overview ≥ 50 chars, poster present, votes ≥ ~100,
then keep the top ~50k by a quality × popularity score.

**Artifact** (`catalogue-vN`, built offline, ~100–300 MB):
- `ids`
- `E`: L2-normalised float32, N × d (default d = 1024; Matryoshka-truncated and re-normalised if the
  eval says 768 or 512 is enough)
- Metadata arrays: title, year, `runtime_min`, genres, director(s), cast, keywords, overview (short),
  tagline, `poster_path` (→ `https://image.tmdb.org/t/p/w500`), rating, votes, language, country
- Facet indices for filters
- A manifest: version, model, d, N, template hash

The API refuses to start on a mismatch. Facets the dataset lacks (cast, keywords, …) are dropped and
the engine degrades gracefully.

**Embedding text (documents):** semantics only (title, year, tagline, overview, keywords, genres),
cut to ~256 tokens, no instruction prefix. Credits go to facets instead of being baked into the
vector, which is what the old template did. People searches become filters
(`search_movies(person=…)`).

**Queries:** `Instruct: {English task, e.g. "Given a movie mood or description, retrieve matching
films"}\nQuery:{query}`. The query can be in Portuguese or English. The model card says omitting the
instruction costs about 1–5% retrieval.

**What Qwen3-Embedding-0.6B costs** (facts from its model card; estimates are mine and to be
measured):
- **Model:** 0.6B params, 28 layers, last-token pooling with left padding, Apache-2.0, 100+
  languages. The card ships neither ONNX nor GGUF.
- **Corpus embedding needs a GPU.** 50k docs × ~200 tokens ≈ 10M tokens. I estimate tens of minutes
  on a free T4 (Kaggle notebook with the dataset attached, or Colab) vs about a day or more on a
  laptop CPU. The monthly refresh embeds only new titles.
- **Serving runtime:** an int8 ONNX export or llama.cpp GGUF (Q8_0), found or converted. It needs a
  parity test against the reference model (cosine ≈ 0.99 on ~200 texts, then hit@10 on the eval).
  Decide whether the corpus is embedded with the serving runtime or the reference one.
- **RAM:** weights ≈ 0.6 GB + catalogue 50k × 1024 × 4 B = 205 MB (102 MB at d=512) + ~40 MB metadata
  + ~120 MB app + runtime buffers ≈ 1.2–1.5 GB RSS. A **≥ 2 GB host** (open item 1).
- **Latency:** about 0.1–0.5 s per query on 2 vCPU. Only free-text `search_movies` embeds a query
  (quick actions and `get_recommendations` do not), and repeated queries hit an LRU cache.

**Eval before locking in dimension and template:**
- Weak-label retrieval for "more like X" (franchise/collection siblings, or same director + shared
  keywords, if the columns exist).
- ~30 hand-written golden queries in Portuguese and English.
- Metric: hit@10.

**Refresh:** the dataset updates daily. Ingestion is a manual monthly script that publishes a new
artifact version and upserts new ids into slim `movies` (never delete). Cron comes later.

**Terms:** TMDB requires attribution and restricts commercial use. IMDb's datasets are
non-commercial. That is fine for a portfolio; sort out licences, including the Kaggle one, before
monetising.

## Recommendation engine (the "better" part)

**Signal:** `status='seen'` rows with a rating. The scale (0–20 today) is config. Ratings are centred
on the user's own mean, so likes pull and dislikes push. Films already in `user_movies` are excluded.

**Stage 1 — candidate generators** (~100 each, union ≈ 300, all in numpy over the RAM catalogue):
1. Taste-vector scan: the signed profile against the whole catalogue (no blur: it scores everything,
   not the neighbours of a mean).
2. Per-seed neighbours of the user's top-rated films (preserves niche, multi-taste profiles).
3. Friends' high-rated films the user has not seen.
4. Same director / cast as the user's favourites (if the facets exist).

**Stage 2 — feature ranker** (per candidate j for user u):
- Similarity features: taste cosine, max cosine to a liked film, max cosine to a disliked film
  (penalty).
- Affinity features: genre, director, cast, decade (the user's mean centred rating per facet value,
  shrunk by count).
- Runtime fit.
- Quality prior (Bayesian-average rating) and popularity (log votes, a small weight for novelty
  control).
- Friends score (accepted friends' centred ratings, shrunk by count).

**The model:** a linear score. Weights start from a **global prior** in config. Once a user has ≥ 20
ratings the weights are refitted per user by ridge regression toward that prior, using
leave-one-out features (no label leakage): `w_u = w₀ + (XᵀX + λI)⁻¹ Xᵀ(y − X w₀)`. With few ratings
`w_u ≈ w₀` (cold-safe). With more, it learns whether this user is plot-driven, director-driven or
genre-driven.

**Build in layers**, each gated by the eval:
1. Generators 1–2 + hand-set prior weights.
2. Facet features.
3. Per-user refit. Ship it only if it beats the prior-only ranker on leave-one-out.

**MMR diversity:** only if personas show franchise clumping.

**Cold start:** < 5 rated films → quality prior. The onboarding flow seeds 5 films.

**Profile card for the agent:** computed in code from all ratings (counts, favourites, dislikes, top
genres, typical runtime), ≤ ~300 tokens.

**Proving "better"** (the dataset has no user ratings, so no offline benchmark):
- Personas: dislikes get demoted, ≥ 3 genres in the top 10, filters respected, no seen titles,
  watchlist rows never crash the profile (finding 2 becomes a regression test).
- Leave-one-out hit@10 vs a popularity baseline on real users (friends testing).
- Weak-label retrieval for embedding choices.
- `rec_events` turns real use into an eval set over time.

## Agent

```
POST /agent/chat {messages: last ≤ 12 text turns}      (session → user_id; server keeps no state)
  system + profile card + window
  repeat ≤ 8 steps and ≤ token ceiling per turn:
      llm.step(messages, tools) ── stream ──► SSE tokens
        ├─ text only            → done
        ├─ tool_call(s) ask_user → SSE `question`, END STREAM (next POST carries the answer)
        └─ tool_call(s) X…      → SSE `step`; validate args; run (in parallel if the model batches
                                  them); append results
  budget exhausted → one last step with tool_choice="none"
```

**Tools** (all read-only; the first 7 are your diagram's):

| Tool | Returns (compact) | Notes |
|---|---|---|
| `search_movies(query?, like_ids?, filters?, k≤8)` | cards `{id,title,year,runtime,genres,rating,line,seen}` | Semantic search, query in any language. Taste-blended by default. Filters: year range, max runtime, genres, person, `exclude_seen=true` |
| `get_recommendations(k≤8, filters?)` **Δ new** | cards + `because:[…]` | The link between the engine and the agent. Answers "what next?" in one call |
| `get_movie(id or title)` | runtime, director, top cast, overview ≤ 300 chars | Title resolved in the catalogue; ambiguous → candidates |
| `get_user_ratings(status?, sort, limit)` | `{id,title,rating,status}` | Numbers only |
| `get_friends()` | `{taguser}` | Accepted `friendships` only. Foldable into the other friend tools to save tokens |
| `get_friends_activity(days≤60, limit)` | `{friend,id,title,rating,when}` | `when` = `updated_at` |
| `get_friends_ratings(movie_ids[] or friend)` | `{friend,id,rating}` | Numbers only |
| `ask_user(question, options?)` | — | Pause and ask in the chat |

**SSE events:** `step{id,tool,label,status}` (UI chips) · `token{text}` · `movies{ids}` ·
`question{text,options?}` (stream ends) · `error{message,retry_after?}` · `done{usage}`.

**Guardrails:**
- `user_id` is never a tool parameter. It is bound from the session.
- Friend tools join on accepted friendships in SQL.
- Args are validated with pydantic and every tool has a timeout.
- The model cites films as `[[id]]`. The server validates the ids against this turn's tool results
  and sends only those in `movies{ids}`. The UI renders `[[id]]` only for validated ids, so invented
  films never become cards.
- Limiter: in-process sliding window per model (8K TPM, 30 RPM, 200K TPD as config). Calls wait up
  to ~15 s, then move down the fallback chain, then show "model busy".
- Per-user daily turn cap; user messages capped at ~1,000 chars.
- Malformed tool call from the provider → retry once with the error appended.

**Token budget (designed for 8K TPM per model):**
- Fixed overhead ≈ 1.3K tokens per call: system ≤ 400, profile card ≤ 300, tool schemas ≤ 600 (so
  one-line tool descriptions).
- Tool results ≤ ~500 tokens, and overviews are cut to ~150 chars.
- gpt-oss has no parallel tool calls, so a typical turn is 2–3 calls and ≈ 5–6K tokens. Qwen can
  batch tool calls and skip reasoning on routing steps, so it should need fewer. Both numbers are
  estimates to be measured in the Phase 0 agent eval.
- The daily cap (200K tokens per model) is only a few dozen turns, which is why quick actions bypass
  the LLM.

**Agent eval (a permanent asset, not a one-off):** ~30 scripted conversations in Portuguese and
English with assertions on tool choice, valid args, step count, no invented ids and tokens per turn.
Cases include a vague request that must trigger `ask_user`, a runtime filter, a friends question, and
adversarial prompts (e.g. "show Ana's email", an instruction planted in an overview). It runs against
any model in minutes and gates every model swap.

## Backend structure, ops, security, privacy

- **Layout:** `api/` (`main, auth, db, llm, agent, tools, catalogue, engine/{generators, features,
  ranker}, routers/{me, friends, movies}`) · `pipeline/` (ingest, embed, eval; its own GPU/torch
  requirements) · `supabase/` (schema, constraints, triggers) · `ARCHITECTURE.md`. The frontend
  returns later as `web/`.
- **Config, not code:** serving filter, prior weights, limits, model names and effort settings,
  rating scale.
- **Secrets:** service key, `SESSION_SECRET`, Groq key, Google client id (TMDB/Kaggle keys offline
  only). CORS locked to the frontend origin. Rate limit `/auth/google`.
- **Logs:** structured per agent step (tool, latency, prompt/completion tokens, status). This is also
  the debug trace.
- **Supabase free tier** pauses after inactivity → a daily health-check ping. Periodic export of
  `user_movies` and `friendships`.
- **RGPD:** ratings and chat go to a US LLM provider. Disclose it, send titles and ratings only
  (never email, name or ids), and expose "delete my data".
- **Counters** are in-process (single worker, which the resident model needs anyway). A second
  instance means moving them to Postgres.

## Frontend (later, separate effort)

Same visual identity, but a refactor against the new API. The backend's job now is a stable OpenAPI
contract (decision 13), the card shape, and the SSE event list above. Clerk and the direct-Supabase
hooks disappear in that refactor. Nothing else is decided here.

## Reuse / replace (paths under `/home/deetee/Documentos/Projetos/CinemaWebApp/fastapi/`)

**Reuse the pattern or the code:**
- `auth.py` (JWKS verification, now for Google's certs).
- `db.py` (`to_card_item` shape, service-key scoping rule).
- `routers/movies.py` (my-library listing contract).
- `routers/friends.py` (flows, adapted to the merged table).
- `routers/account.py` (without the Clerk call).
- `tests/conftest.py` and the CI shape.
- `regenerate_embeddings.py` (checkpoint/resume ideas) and the `populate_tmdb.py` TMDB fetchers, if
  the CSV lacks fields.

**Replace:**
- `main.py` recommendation internals
- `rag_service.py`
- `recommendation_system.py`
- `routers/recommendations.py`
- The ivfflat index, the `match_movies` RPC, the `user_recommendations` poll flow and the
  bge-large/sentence-transformers embedding step

The two contradicting READMEs go away; `ARCHITECTURE.md` becomes the source of truth.

## Phases (each ends with a checkable exit)

0. **Inspect and spike** (de-risk, then go/no-go):
   - Read the CSV header, size and licence, and count titles per vote threshold.
   - **Host:** pick one with ≥ 2 GB RAM and unbuffered SSE (open item 1).
   - **Embedding feasibility:** find or convert an int8 ONNX / GGUF of Qwen3-Embedding-0.6B. Exit:
     parity vs the reference, RSS < ~1.5 GB, ≤ ~0.5 s per query on the host, hit@10 within ~2
     points of the reference runtime. Choose d (1024/768/512) and run the GPU corpus job on a
     sample.
   - **LLM:** the agent eval on `qwen3.8-27b` and `gpt-oss-120b` (and `-20b` as fallback). Exit:
     ≥ 95% valid tool calls, ≤ ~6K tokens/turn, a chosen primary and chain. Also check whether
     tool-turn quality drops when gpt-oss reasoning is not echoed back.
   - **Engine:** layers 1–2 on personas vs the old weighted-mean baseline.
1. **Data plane:** ingest → filter → embed on GPU → artifact + slim `movies` → eval harness.
2. **Core backend:** Google auth/session, profiles, ratings, merged friends, catalogue, engine,
   `/me/recommendations`, OpenAPI. Exit: personas and leave-one-out beat the baseline.
3. **Agent:** `llm.py` (limiter, fallback chain), `tools.py`, loop, SSE, quick actions, caps. Exit:
   stub-LLM tests pass (step cap, `ask_user` turn boundary, id guard, user_id binding, friend
   scoping) and the agent eval passes on the chosen chain.
4. **Hardening and launch:** RGPD text, backups/keepalive, CI, deploy. The frontend refactor starts
   against the contract.
- **Later:** per-user refit if it did not make the cut · MMR · `rec_events`-driven learning ·
  collaborative signal if ratings data appears · group movie-night sessions · pgvector HNSW at scale
  · a Claude adapter in `llm.py`.

## Risks and open items

1. **Backend host and RAM.** Qwen3-Embedding-0.6B needs a **≥ 2 GB host with unbuffered SSE**, and
   the host is not recorded in the repo. Candidates to verify (terms change): Hugging Face Spaces
   (Docker, free CPU tier with a lot of RAM), Oracle Cloud Always Free ARM, Cloud Run, Fly.io.
   Render's free 512 MB tier cannot carry it. If no ≥ 2 GB host is workable, fall back to
   `bge-small-en-v1.5` (decision 5).
2. **Needed from you:** the CSV header and the licence line.
3. **CPU query latency and the ONNX/GGUF export** are unmeasured. Mitigations: the LRU cache, the
   spike's exit numbers, and quick actions that never embed a query.
4. **Preview-model risk:** Qwen3.8-27B can vanish at short notice, and the free LLM ceiling is a few
   dozen turns per model pool per day. Mitigations: fallback chain, quick actions, daily caps, the
   `llm.py` seam. All model names and limits are as of 2026-10-10; re-run the agent eval at build
   time.
5. **Per-user refit** may not beat the prior-only ranker with few ratings. That is why it is gated by
   the eval.
6. **No offline benchmark** because the dataset has no ratings. Personas, friends testing and
   `rec_events` substitute for it.

## Verification (how we'll know the architecture holds)

- Phase 0 spikes pass their exit numbers; otherwise revisit decisions 3, 5, 6 and 9.
- Personas and leave-one-out show the engine beats the popularity baseline and the old weighted-mean
  approach.
- Stub-LLM agent tests prove the guardrails (they need no network). The agent eval proves the
  chosen models.
- Auth tests: a forged or expired session, a wrong `aud` and a missing `email_verified` are all
  rejected, and no endpoint accepts a user id from the request.
- Confirm Groq's live model list (`/models`) before building on any model.
