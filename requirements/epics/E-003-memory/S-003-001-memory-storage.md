# Story: Memory Storage Layer (Mem0 Integration)

---

**ID:** S-003-001
**Epic:** E-003
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P0
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **Noxis to remember facts about me across every conversation**, so that **I never have to repeat myself and the advice I receive reflects my actual life, not a generic profile**.

## Goal

A persistent memory layer is integrated using Mem0 (or an equivalent vector + structured store). Memory entries are categorized, confidence-scored, and attributable to their source. Store, retrieve, and delete operations are exposed as a clean internal API used by the orchestration pipeline and memory extraction process.

## Why / Rationale

Every feature that makes Noxis feel personal — outfit advice that knows your style, food guidance that knows your diet preferences, gym recommendations that know your training split — depends on a reliable, queryable memory layer. Without this foundation, the product is a chat app. With it, it becomes an advisor that compounds knowledge over time.

## Functional Requirements

- Mem0 is integrated as the primary memory backend; all memory operations route through a `MemoryService` abstraction layer that can swap the underlying provider without changing call sites
- Each memory entry has the following schema:
  - `id: UUID`
  - `user_id: String`
  - `content: String` (natural language fact, e.g. "User hates slim-fit jeans")
  - `category: MemoryCategory` — enum: `wardrobe | gym | food | spending | routines | social | general`
  - `confidence: Float` (0.0–1.0; set by extraction model at time of creation)
  - `source: MemorySource` — enum: `chat | manual | onboarding`
  - `created_at: Date`
  - `updated_at: Date`
- **Store operation:** `addMemory(content:, category:, confidence:, source:, userId:) async throws -> Memory`
- **Retrieve operation:** `fetchMemories(userId:, query:, category:?, limit: Int = 30) async throws -> [Memory]` — uses Mem0's vector search to rank by relevance to the query string; optional category filter narrows the result set
- **Delete operation:** `deleteMemory(id:, userId:) async throws`
- **List operation:** `listMemories(userId:, category:?) async throws -> [Memory]` — returns all memories for a user optionally filtered by category; used by the Memory Review screen (S-003-007)
- Mem0 API calls are authenticated using an API key stored in the backend environment — never in the iOS client
- All memory operations are backend-mediated; the iOS client never calls Mem0 directly
- Memory operations are idempotent: adding a memory with identical content to an existing entry for the same user and category updates the confidence and `updated_at` rather than creating a duplicate

## Prerequisites

- S-002-003 (module routing — category enum must be consistent with module routing categories)

## Acceptance Criteria

- [ ] Given a new memory is added via `addMemory`, when the operation completes, then the memory is persisted and retrievable via `fetchMemories` with the correct category, confidence, and source fields
- [ ] Given `fetchMemories` is called with a query string, when results are returned, then they are ranked by relevance to the query and the most relevant item appears first
- [ ] Given `deleteMemory` is called with a valid memory ID, when the operation completes, then the memory is no longer returned by any retrieve or list operation
- [ ] Given a memory with identical content already exists for the user and category, when `addMemory` is called again, then a duplicate is not created — the existing entry's confidence and `updated_at` are updated instead
- [ ] Given the Mem0 API is unavailable, when `fetchMemories` is called, then the operation throws a `MemoryServiceError.unavailable` error and the orchestration pipeline falls back gracefully to USER.md context only
- [ ] Given `listMemories` is called with a category filter, when results are returned, then only memories matching that category are included in the response
- [ ] Given any memory operation, when it is executed, then the Mem0 API key is sourced from the backend environment variable — it is never present in the iOS client bundle or network traffic from the device

## Negative Scenarios

- Mem0 returns a malformed response → `MemoryService` throws `MemoryServiceError.decodingFailure`; the calling pipeline step catches it and logs the raw response for debugging; operation fails gracefully without crashing
- Memory store reaches Mem0's per-user record limit → `addMemory` receives a capacity error; the oldest lowest-confidence memory in the same category is evicted to make space; the new memory is then stored

## Edge Cases

- Memory content is an empty string → `addMemory` throws `MemoryServiceError.invalidContent` before making any API call; validated at the service layer
- Two concurrent `addMemory` calls for the same user and content arrive simultaneously → idempotency logic uses an upsert operation at the Mem0 level; one write wins; no duplicate is created

## Dependencies

- S-002-003 (module routing — category enum alignment)

## Technical Requirements

- **`MemoryService.swift` (backend)** — Swift or Python module (depending on backend language choice) exposing: `addMemory`, `fetchMemories`, `deleteMemory`, `listMemories`
- **Mem0 SDK:** use the official Mem0 Python SDK (`mem0ai`) or REST API; configure with `api_key` from env var `MEM0_API_KEY`
- **Category mapping:** `MemoryCategory` enum values map to Mem0 `metadata.category` field for filtering
- **Relevance search:** `mem0.search(query=query, user_id=user_id, limit=limit, filters={"category": category})` for retrieve; `mem0.get_all(user_id=user_id)` for list
- **Idempotency:** before `mem0.add()`, run `mem0.search(query=content, user_id=user_id, limit=1)` with exact-match threshold; if cosine similarity > 0.97, call `mem0.update(id, confidence)` instead
- **Backend API routes:** `POST /api/v1/memory` (add), `GET /api/v1/memory?query=&category=` (fetch), `DELETE /api/v1/memory/{id}` (delete), `GET /api/v1/memory/all?category=` (list)
- **No iOS direct calls:** iOS app communicates with `/api/v1/memory/*` only; never uses Mem0 API key

## Future Scope

- Memory importance weighting: high-importance memories (user goals, core preferences) decay more slowly than incidental facts
- Memory versioning: track full history of changes to a memory entry for audit and review
- Cross-user memory patterns (anonymized, opt-in): surface insights like "most users with your gym frequency prefer X"

## Do Not Do

- In this story, do not implement memory extraction from conversations — that is S-003-003
- Do not implement the memory review UI — that is S-003-007
- Do not implement contradiction resolution logic — that is S-003-005
- Do not give the iOS client direct access to the Mem0 API key

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |
