# Story: Orchestration Pipeline — Personality + Memory + Context

---

**ID:** S-004-005
**Epic:** E-004
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P0
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **every message I send to trigger a response that reflects who I am, what I care about, and Noxis's fixed personality**, so that **the conversation never feels generic — it always feels like Noxis knows me and has opinions**.

## Goal

The orchestration pipeline is the server-side engine that assembles every Noxis response. It runs in sequence: classify the message → retrieve relevant memory → filter memory through the decision policy → build the full system prompt → call the AI provider → stream the response → extract new memory candidates → flag dashboard data for sync. Every single Noxis response passes through this pipeline.

## Why / Rationale

This is the backbone of the entire product. Without a disciplined pipeline, memory and personality become inconsistent — sometimes applied, sometimes not. The pipeline must be a single, explicit, auditable code path so that behavior is predictable and testable. Every part of Noxis's intelligence runs through here.

## Functional Requirements

**Step 1 — Message Classification:**
- Determine the message type: `text_only`, `text_with_image`, `image_only`
- If image present, trigger image classification (S-004-003) to get `image_category`
- Determine the relevant life module from message content: wardrobe / gym / food / spending / routines / general

**Step 2 — Memory Retrieval:**
- Query Mem0 (S-003-001) with the classified module and the user's message as the search query
- Retrieve up to 30 candidate memory items ranked by relevance score
- Include the user's USER.md core profile fields as structured context alongside retrieved memories

**Step 3 — Decision Policy Filter (S-003-004):**
- Apply relevance filter: keep only memory items with module match + recency score above threshold
- Apply quality filter: drop items below confidence 0.5; flag contradictions for resolution
- Apply capacity limit: max 20 memory items proceed to the prompt

**Step 4 — System Prompt Assembly:**
- Layer 1 (fixed, always first): SOUL.md full content
- Layer 2: tone mode instruction based on user's current tone selection (S-002-002)
- Layer 3: module context (judgment lens if image, or module-specific framing if message is module-relevant)
- Layer 4: filtered memory items, structured as "What I know about this user:" block
- Layer 5: conversation history (last 10 message pairs from the active thread)
- Layer 6: current user message (with image URL if applicable)

**Step 5 — AI Provider Call:**
- Send assembled prompt to the configured AI provider (OpenAI GPT-4o or Anthropic Claude)
- Use streaming mode; surface tokens to the client in real time via Server-Sent Events (SSE)
- Record: total tokens used, model version, latency, SOUL.md version hash

**Step 6 — Response Streaming:**
- Stream tokens to the client via SSE; client (S-004-006) renders text word by word
- On stream completion, store the full response text as a `ChatMessage` record

**Step 7 — Memory Extraction:**
- After response completes, run a lightweight post-generation pass (S-003-003) to extract new memory candidates from the exchange
- Extracted candidates are written to Mem0 asynchronously — does not block the user

**Step 8 — Dashboard Data Sync:**
- After response completes, run the chat extraction pipeline (S-005-007) to identify and sync structured data to the appropriate life module store
- Sync is asynchronous — does not block the user or the response display

## Prerequisites

- S-002-001 (SOUL.md load — must exist before pipeline can assemble system prompt)
- S-002-002 (tone mode — must be retrievable per user)
- S-002-003 (module routing — classification logic)
- S-003-001 (Mem0 integration — memory retrieval)
- S-003-004 (decision policy — filtering logic)

## Acceptance Criteria

- [ ] Given a user sends any message, when the pipeline runs, then SOUL.md content is always present as the first layer of the system prompt — verifiable in server logs
- [ ] Given a user has stored memory items, when a relevant message arrives, then the filtered memory is injected into the system prompt and the response reflects that memory without the user having to re-state it
- [ ] Given a user sends an image with a classified category, when the pipeline assembles the prompt, then the correct module judgment lens is included in layer 3
- [ ] Given the AI provider call completes, when the pipeline finishes step 5, then the token count, model version, latency ms, and SOUL.md version hash are all present in the request log record
- [ ] Given the user's response is streaming, when steps 7 and 8 run, then they execute asynchronously and do not delay or interrupt the visible text stream
- [ ] Given the decision policy filter runs on 30 retrieved memory items, when the filter completes, then no more than 20 items are passed to the system prompt assembly
- [ ] Given SOUL.md and a user message are assembled into the prompt, when the AI provider call is made, then the system prompt ordering is: SOUL.md → tone → module context → memory → conversation history → current message

## Negative Scenarios

- Mem0 retrieval times out (>5 seconds) → pipeline proceeds with USER.md core profile only and zero retrieved memories; response is generated; the timeout is logged for monitoring
- AI provider call fails after 3 retries → return a structured error to the client: `{ "error": "response_unavailable" }`; client shows a brief inline error with "Try again" tap target; no raw error text is exposed to the user

## Edge Cases

- User message is very long (>2000 tokens) → conversation history is trimmed from the oldest end to maintain total prompt under the model's context limit; SOUL.md and current message are never trimmed
- Memory extraction step (step 7) fails silently → the user-facing response is already delivered; the failure is logged; memory is not stored for this exchange; no user-visible error

## Dependencies

- S-002-001 (SOUL.md)
- S-002-002 (tone mode)
- S-002-003 (module routing)
- S-003-001 (Mem0 integration)
- S-003-004 (decision policy)

## Technical Requirements

- **`OrchestrationPipeline.swift` (or equivalent backend module)** — a single `run(message: IncomingMessage, userId: String) async throws -> ResponseStream` function that executes all 8 steps in sequence; steps 7 and 8 are dispatched with `Task { }` after the stream completes
- **`SystemPromptBuilder.swift`** — assembles the layered prompt; accepts `soul: String, toneInstruction: String, moduleContext: String?, memories: [MemoryItem], history: [ChatMessage], userMessage: String`; returns `[APIMessage]`
- **`PipelineLogger.swift`** — records `{ pipeline_run_id, user_id, soul_version, model, tokens_in, tokens_out, latency_ms, image_category?, memory_count, step_durations }` per run
- **SSE streaming:** backend sends `data: { "token": "word" }\n\n` chunks; client `URLSessionDataTask` with delegate processes chunks in real time
- **AI provider abstraction:** `AIProvider` protocol with `func complete(messages: [APIMessage]) async throws -> AsyncStream<String>`; concrete implementations for OpenAI and Anthropic swappable via config flag
- **Context window management:** `PromptTruncator` utility that measures token count of assembled prompt using tiktoken-equivalent; trims `history` oldest-first until total is within the model's limit minus 1000 tokens (response buffer)
- **Timeout policy:** step 2 (memory retrieval): 5s; step 5 (AI call): 60s with streaming keepalive; classification sub-call: 10s

## Future Scope

- Pipeline versioning: A/B test different prompt assembly strategies
- Latency breakdown dashboard: per-step timing visible in admin panel
- Local on-device pipeline for sensitive messages (on-device LLM, iOS 18+)

## Do Not Do

- In this story, do not implement the streaming display UI — that is S-004-006
- Do not implement memory extraction logic — only dispatch the call; the extraction itself is S-003-003
- Do not implement dashboard sync logic — only dispatch the call; the sync itself is S-005-007
- Do not expose pipeline internals, system prompt content, or memory items to the user via any UI

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |
