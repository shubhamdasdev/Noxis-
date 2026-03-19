# Architecture Task: FastAPI Backend Scaffold + Supabase Schema

---

**ID:** AT-001-002
**Release:** R-001
**Stage:** Backlog
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Extends:** None
**Depends On:** None
**Version:** 1.0

---

## What

Establishes the FastAPI backend project and creates all Supabase database tables, auth configuration, and storage buckets. Every backend story in R-001 (auth, chat, modules, briefs) depends on this infrastructure existing. Without it, coding agents would each independently create database connections, invent table schemas, and produce SQL that conflicts. This AT produces a runnable FastAPI server with all routers present (empty handlers), all 9 Supabase tables created and verified, and a Railway deployment pipeline ready to receive code.

## Artifact Location

```
backend/
  main.py                         ← FastAPI app factory, CORS, router registration
  routers/
    auth.py                       ← empty router, correct route stubs
    chat.py
    modules.py
    briefs.py
    memory.py
  services/
    supabase_client.py            ← shared Supabase client singleton
  schemas/
    user.py                       ← Pydantic models for user + auth requests/responses
    chat.py
    modules.py
  supabase/
    migrations/
      001_initial_schema.sql      ← all 9 tables from arch.md Data Model section
  .env.example                    ← all required env vars, no values
  requirements.txt
  Procfile                        ← Railway process definition
```

## Steps

1. `[User]` Create a Supabase project at supabase.com. Copy the Project URL and `anon` key and the `service_role` key. Tell me when this is done and provide the Project URL.

2. `[User]` Create a Railway project at railway.app connected to the Noxis GitHub repo. Tell me when this is done.

3. `[Agent]` FastAPI project is scaffolded — `main.py`, all routers registered, `requirements.txt` includes `fastapi`, `uvicorn`, `supabase`, `python-dotenv`, `pydantic`; server starts with `uvicorn main:app --reload` with no errors

4. `[Agent]` `001_initial_schema.sql` migration contains exactly the 9 tables from `arch.md` Data Model section — `users`, `threads`, `messages`, `daily_briefs`, `wardrobe_items`, `workout_logs`, `meal_logs`, `purchase_logs`, `habits`, `habit_completions` — with all constraints, foreign keys, and `CHECK` clauses specified in `arch.md`; migration is applied to the Supabase project (e.g. via `supabase db push` or the Supabase SQL editor)

5. `[Agent]` Supabase Auth is configured: email/password provider enabled; Apple Sign-In provider enabled with the correct `redirect_url` (to be updated with production URL post-launch); JWT expiry set to 7 days

6. `[Agent]` Supabase Storage bucket `media` created (private, max 10MB per file); bucket policy allows authenticated users to read/write their own files only

7. `[Agent]` `.env.example` lists all required env vars: `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `OPENAI_API_KEY`, `MEM0_API_KEY`, `RAILWAY_ENVIRONMENT`; `.env` added to `.gitignore`

8. `[Agent]` All route stubs in every router return `{"status": "not_implemented"}` with correct HTTP methods matching `arch.md` API Routes section exactly; `GET /health` returns `{"status": "ok"}`; server deploys to Railway without errors

## Acceptance Criteria

- [ ] `uvicorn main:app` starts without errors on Python 3.12
- [ ] `GET /health` returns `{"status": "ok"}` with 200
- [ ] All 9 tables from `arch.md` exist in Supabase with exact column names, types, and constraints — verifiable via `SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'`
- [ ] `users.tone_mode` column has CHECK constraint: `tone_mode IN ('brother', 'consultant', 'peer')`
- [ ] `messages.role` column has CHECK constraint: `role IN ('user', 'assistant')`
- [ ] `daily_briefs` has UNIQUE constraint on `(user_id, date)`
- [ ] `wardrobe_items.source`, `workout_logs.source`, `meal_logs.source`, `purchase_logs.source`, `habits.source` all have CHECK constraint: `source IN ('manual', 'chat')` (habits also includes `'noxis'`)
- [ ] Supabase Auth email/password and Apple Sign-In providers are enabled
- [ ] Supabase Storage `media` bucket exists with private access policy
- [ ] All API routes in `arch.md` have a corresponding stub handler in the correct router file; no routes are missing
- [ ] `.env.example` is committed; `.env` is in `.gitignore` and not committed

## Depends On

None — this task is parallel with AT-001-001.

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |
