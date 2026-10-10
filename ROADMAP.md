# MovieNight.AI — Roadmap

Companion to `ARCHITECTURE.md`: the commits we will make (in order), a TODO list, and the questions
still open. Commit subjects follow Conventional Commits. Every commit leaves lint and tests green and
updates `ARCHITECTURE.md` when it changes a decision. Commits made with Claude Code end with
`Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`. Suggested flow: one branch per phase,
merged without squashing so these commits stay visible.

## 1. Planned commits

### Phase 0 — Bootstrap and spikes (exit: go/no-go on host, embedding runtime, LLM, engine)

1. `chore: init repository with .gitignore, README and the two docs`. The folder is not a git repo
   yet. Ignore `.env*`, `*.npz`, `*.onnx`, `*.gguf` and `catalogue-*` (the old repo committed 93 MB
   of caches).
2. `chore(api): scaffold FastAPI app with config loader, /health, pytest and ruff`.
3. `ci: lint and test on push`.
4. `spike(pipeline): inspect the Kaggle CSV`. Header, row counts per vote threshold, licence note →
   `docs/dataset.md`.
5. `spike(embed): Qwen3-Embedding-0.6B runtime parity, latency and RSS`. int8 ONNX vs GGUF Q8_0 vs
   the reference → `docs/spikes/embedding.md`.
6. `feat(eval): scripted agent eval harness`. ~30 PT/EN conversations with canned tool results and
   assertions; first run on qwen3.8-27b, gpt-oss-120b and gpt-oss-20b → `docs/spikes/llm.md`.
7. `spike(engine): personas and weighted-mean baseline vs layer 1–2 prototype`.
8. `docs: record the host, model and dimension decisions`. Updates decisions 3, 5 and 9. This is the
   go/no-go.

### Phase 1 — Data plane (exit: artifact + slim `movies` published; eval harness green)

9. `feat(pipeline): ingest the Kaggle CSV and apply the config-driven serving filter`.
10. `feat(pipeline): document-text and query-instruction builders`. Unit-tested.
11. `feat(pipeline): GPU embedding job with checkpoint/resume (script + notebook)`.
12. `feat(pipeline): build and publish the catalogue artifact and manifest`.
13. `feat(eval): weak-label retrieval and golden-query hit@10`.
14. `feat(supabase): initial schema`. profiles, slim movies, user_movies, friendships, CHECKs,
    `updated_at` trigger, RLS on with no policies. Applies cleanly to an empty project.
15. `feat(pipeline): upsert slim movies into Postgres`.

### Phase 2 — Core backend (exit: personas and leave-one-out beat the baseline)

16. `feat(api): Google ID-token verification and POST /auth/google`. Tests: forged, expired, wrong
    `aud`, unverified email, allowlist.
17. `feat(api): Bearer session dependency, profile upsert and GET /me`. No endpoint takes a user id
    from the request.
18. `feat(api): library endpoints`. `PUT/DELETE /me/movies/{id}`, `GET /me/movies` with status
    counts.
19. `feat(api): friends on the merged table`. Requests, accept/decline, list.
20. `feat(api): catalogue module`. Load artifact, manifest check, filters, card shape.
21. `feat(api): query embedder with LRU cache and GET /movies/search`.
22. `feat(engine): signed taste profile and candidate generators 1–2`.
23. `feat(engine): feature ranker layer 1 with templated explanations`.
24. `feat(api): GET /me/recommendations`. Computed on request, cached by `updated_at`; includes the
    watchlist-row regression test.
25. `feat(engine): facet features (layer 2)`. Genre, director, cast, decade, runtime. Dropped if
    personas do not improve.
26. `feat(engine): per-user shrunk ridge refit (layer 3)`. Only if it beats prior-only on
    leave-one-out.
27. `docs(api): export OpenAPI and pin the card and SSE contract for the frontend`.

### Phase 3 — Agent (exit: stub-LLM tests and the agent eval pass on the chosen chain)

28. `feat(llm): provider seam, per-model limiter and fallback chain`.
29. `feat(agent): read-only tools with pydantic validation and user_id binding`.
30. `feat(agent): SSE loop with step/token ceilings and the ask_user turn boundary`.
31. `feat(agent): profile card, system prompt and the [[id]] guard with the movies event`.
32. `feat(agent): quick actions and per-user daily cap`.
33. `test(agent): stub-LLM tests for the guardrails`.
34. `feat(eval): run the agent eval on the chain and record the chosen primary`.

### Phase 4 — Hardening and launch

35. `feat(api): DELETE /me (delete-my-data)`.
36. `feat(api): rec_events logging` (optional, see question 7).
37. `chore(ops): Dockerfile, keepalive ping and export script`.
38. `ci: build the image and deploy to the chosen host`.
39. `docs: RGPD notes and runbook`. Monthly catalogue refresh, `SESSION_SECRET` rotation, re-running
    the agent eval after any model swap.

The frontend refactor is its own roadmap, started against the contract from commit 27.

## 2. TODO

### Blocking, before commit 1
- [ ] (you) Pick the backend host: ≥ 2 GB RAM, unbuffered SSE.
- [ ] (you) Paste the Kaggle CSV header and the licence line, or download the file and tell me where.
- [ ] (you) Run `GET https://api.groq.com/openai/v1/models` with your key and tell me which models it
  sees. Also read your org's rate-limits page for the free-plan numbers.
- [ ] (you) Google Cloud: create a Web OAuth client and consent screen (publish to production). I
  only need the client id.
- [ ] (you) Supabase: create the v2 project (or tell me to reuse the old one), and dump the live
  CinemaWebApp schema for reference.
- [ ] (you) GitHub: create the repo and decide its visibility.
- [ ] (you) A GPU for the embedding job: a Kaggle (phone-verified) or Colab notebook.
- [ ] (you) A TMDB API key, only if the CSV lacks fields, plus the attribution text.

### During the build
- [ ] (me) Draft 6–8 personas for the engine eval (you review and add real tastes).
- [ ] (me) Draft the ~30 agent-eval conversations (you add real-world prompts, in PT and EN).
- [ ] (you) Rate a few dozen films in the v2 app so leave-one-out has data. Friends testing helps.
- [ ] (you) Put the secrets in the host and in GitHub Actions secrets: service key,
  `SESSION_SECRET`, Groq key, Google client id.
- [ ] (you) Write the privacy-policy text (RGPD): data processed, US LLM provider, deletion path.
- [ ] (me) Re-run the agent eval whenever a model name changes.

### Backlog (not in v1)
- [ ] Per-user refit if it did not make the cut · MMR diversity · `rec_events`-driven learning.
- [ ] Collaborative signal if a ratings dataset appears.
- [ ] Group movie-night sessions (profiles compose).
- [ ] pgvector HNSW or a RAM cache at scale.
- [ ] A Claude adapter in `llm.py` if the free quota bites.
- [ ] Cron for the catalogue refresh.
- [ ] Per-user session revocation (`token_epoch`).
- [ ] Server-side chat history if you want it across devices.

## 3. Open questions (my default in brackets; "ok" means accept all defaults)

1. Which backend host? [Hugging Face Spaces or Oracle Always Free ARM; verify current terms]
2. CSV header and licence line? [needed; no default]
3. Is "Qwen3/8B" `qwen/qwen3.8-27b`? [yes]
4. Keep the 0–20 rating scale? [yes, as a config value]
5. New Supabase project or reuse the old one? [new project; the old one stays as reference until v2
   is live]
6. GitHub repo name and visibility? [`MovieNight.AI`, private]
7. Log `rec_events` in v1? [yes, commit 36]
8. Agent replies in the user's language (PT/EN)? [yes, mirror the user]
9. Delete the placeholder `MovieNightAI ARCHiTECTURE` once `ARCHITECTURE.md` exists? [leave it]
10. Group movie-night sessions stay out of v1? [yes]
11. Open sign-up, or invite-only (`ALLOWED_EMAILS`) while on the free LLM tier? [invite-only]
12. May the embedding dimension be truncated (1024 → 768 or 512) if the eval shows ≤ 1 point loss?
    [yes]
