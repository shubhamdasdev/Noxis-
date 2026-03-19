# Architecture Task: Mem0 + OpenAI Integration Setup

---

**ID:** AT-001-004
**Release:** R-001
**Stage:** Backlog
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Extends:** None
**Depends On:** AT-001-002
**Version:** 1.0

---

## What

Establishes verified, working connections to Mem0 (hosted) and OpenAI GPT-4.1, plus the decision policy layer that sits between them. Both services are external dependencies that multiple stories in R-001 rely on — S-003-001 (memory storage), S-004-005 (orchestration pipeline), and S-006-001 (daily brief engine) all call both services. Without this AT, coding agents would independently set up API clients, choose SDK versions, and invent service wrapper patterns — creating duplicate code and connection bugs. This AT produces a `memory_service.py` stub with Mem0 connected and a `openai_client.py` singleton with GPT-4.1 configured, plus the `apply_decision_policy()` function the orchestration pipeline will call.

## Artifact Location

```
backend/
  services/
    memory_service.py             ← Mem0 client + add_memory() + search_memory() + apply_decision_policy()
    openai_client.py              ← OpenAI client singleton, model config, stream helper
  tests/
    test_memory_service.py        ← integration smoke test: add + retrieve a test memory
    test_openai_client.py         ← integration smoke test: single non-streaming completion
```

## Steps

1. `[User]` Go to mem0.ai, create an account, and generate an API key. Add `MEM0_API_KEY=<your-key>` to your `.env` file. Tell me when done.

2. `[User]` Go to platform.openai.com, generate an API key with access to `gpt-4.1`. Add `OPENAI_API_KEY=<your-key>` to your `.env` file. Tell me when done.

3. `[Agent]` `pip install mem0ai openai` added to `requirements.txt`; `memory_service.py` initialises a `MemoryClient` from the `mem0ai` SDK using `MEM0_API_KEY` from environment; connection verified by calling `client.get_all(user_id="architect-test")` — returns without error

4. `[Agent]` `memory_service.py` implements the three public functions that stories will call:
   - `add_memory(user_id: str, content: str, metadata: dict) -> str` — calls `client.add()`, returns memory ID
   - `search_memory(user_id: str, query: str, limit: int = 50) -> list[dict]` — calls `client.search()`, returns raw memory list
   - `apply_decision_policy(memories: list, module: str, query: str) -> list` — exact implementation from `arch.md` Decision Policy section (relevance filter → recency weight → quality filter at 0.5 confidence → sort → cap at 20)

5. `[Agent]` `openai_client.py` initialises an `openai.AsyncOpenAI` client using `OPENAI_API_KEY`; model is set via `MODEL_ID = os.getenv("OPENAI_MODEL_ID", "gpt-4.1")`; exports two functions:
   - `complete(messages: list, max_tokens: int = 1000) -> str` — non-streaming completion
   - `stream(messages: list) -> AsyncGenerator[str, None]` — async generator yielding text chunks as SSE tokens

6. `[Agent]` Integration smoke tests pass: `test_memory_service.py` adds a memory entry `{"user_id": "test-user", "content": "test memory", "metadata": {"category": "general"}}` and retrieves it; `test_openai_client.py` calls `complete([{"role": "user", "content": "Say hello"}])` and receives a non-empty string

## Acceptance Criteria

- [ ] `from services.memory_service import add_memory, search_memory, apply_decision_policy` imports without error
- [ ] `add_memory("test-user", "test content", {"category": "general"})` returns a non-empty string (the memory ID)
- [ ] `search_memory("test-user", "test content")` returns a list containing the added memory
- [ ] `apply_decision_policy` exactly implements the 4-step logic from `arch.md`: relevance filter (module match or 'general') → recency score (fact: 1.0, others: decay over 90 days) → quality filter (confidence ≥ 0.5) → sort by composite score → cap at 20 results
- [ ] `apply_decision_policy` returns a list of ≤ 20 items under all inputs
- [ ] `from services.openai_client import complete, stream` imports without error
- [ ] `complete([{"role": "user", "content": "Say hello"}])` returns a non-empty string without raising an exception
- [ ] `OPENAI_MODEL_ID` env var overrides the default model — calling `complete()` uses the configured model ID, not a hardcoded string
- [ ] Both API keys are read from environment variables only — never hardcoded in any source file

## Depends On

AT-001-002 (backend scaffold must exist before services/ can be populated)

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |
