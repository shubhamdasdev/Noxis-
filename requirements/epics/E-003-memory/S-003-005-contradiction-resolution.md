# Story: Memory Contradiction Resolution

---

**ID:** S-003-005
**Epic:** E-003
**Project:** noxis
**Status:** Backlog
**Stage:** Draft
**Priority:** P2
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Version:** 1.0

---

## Story

As a **user**, I want **Noxis to notice when something about me has changed and update its understanding accordingly**, so that **I'm not getting advice based on who I was three months ago**.

## Goal

When two memories for the same fact conflict, the more recent one wins and the older one is replaced. When Noxis detects a contradiction mid-response, it surfaces it explicitly in natural language — acknowledging the change, declaring which version it is using, and inviting a correction if needed. The stored memory is updated to the newer value.

## Why / Rationale

People change. Training frequency shifts. Dietary preferences evolve. Style preferences rotate. A memory system that doesn't handle contradiction becomes stale and starts giving outdated advice. Explicit contradiction surfacing is also a trust-building mechanism — it shows the user that Noxis pays attention and is honest about uncertainty.

## Functional Requirements

- Contradiction detection is triggered in two places:
  1. During the decision policy filter (S-003-004) — silently resolves conflicts in the retrieved set before injection (older item removed)
  2. During memory extraction (S-003-003) — when a new extraction candidate conflicts with an existing Mem0 entry, the contradiction resolution flow is invoked
- **Resolution rule:** the more recently `updated_at` memory wins; the older memory is replaced (not just flagged) in Mem0
- **User-facing surfacing:** when a contradiction is resolved during extraction (a new fact overrides an existing one), the orchestration pipeline includes a natural-language surface in the next relevant response:
  - Format: "Last time we spoke about this, [old value]. But today you mentioned [new value] — I'm updating what I know. Let me know if I've got it wrong."
  - The surfacing is appended to the response naturally, not as a separate system message
  - Surfacing only fires once per resolved contradiction; after that, the updated memory is treated as the new ground truth
- **Surfacing gate:** surfacing only happens when:
  - The old memory is at least 14 days old (very recent flip-flops are resolved silently)
  - The new memory has confidence >= 0.70
  - The contradiction is in a meaningful category (wardrobe, gym, food, routines) — not `general` or `social`
- When a contradiction is resolved, the old memory is deleted from Mem0 and replaced by the new memory entry
- A `contradiction_resolutions` log table records: `user_id`, `old_memory_id`, `new_memory_id`, `old_content`, `new_content`, `resolved_at`, `surfaced: Bool`

## Prerequisites

- S-003-003 (memory extraction — contradictions are detected when new extractions conflict with existing entries)
- S-003-004 (decision policy — silent contradiction resolution in the filter layer)

## Acceptance Criteria

- [ ] Given an existing memory "User trains 4x per week" and a new extraction "User now trains 6x per week", when the contradiction is detected, then the older memory is deleted from Mem0 and replaced by the newer one
- [ ] Given the resolved contradiction meets the surfacing gate conditions (old memory > 14 days, new confidence >= 0.70, meaningful category), when the next relevant conversation occurs, then Noxis surfaces the change in natural language within the response
- [ ] Given the surfacing fires once for a resolved contradiction, when subsequent conversations reference the same topic, then the surfacing language does not appear again — only the updated memory is used silently
- [ ] Given a contradiction between two memories where the old memory is less than 14 days old, when the resolution runs, then the contradiction is resolved silently with no user-facing message
- [ ] Given a contradiction is resolved, when the `contradiction_resolutions` table is inspected, then a record exists with both old and new memory IDs, content, timestamp, and the correct `surfaced` boolean
- [ ] Given two contradicting memories with identical `updated_at` timestamps, when resolution runs, then the higher-confidence item wins; if confidence is also equal, the one with the longer content string is retained as the more specific fact

## Negative Scenarios

- The contradiction detection produces a false positive (two memories that are actually compatible) → the more recent item is retained, which is the safer outcome; a surfacing message acknowledging the "change" may fire if the gates are met; the user can correct Noxis mid-conversation which triggers a new extraction
- Contradiction resolution fails to delete the old Mem0 entry (API error) → the new entry is still stored; a dead-letter record is written to `contradiction_resolutions` with `resolved: false`; a background retry job attempts cleanup

## Edge Cases

- Three-way contradiction (three memories all describing the same fact with different values) → each pair is resolved sequentially by recency; the oldest is eliminated first, then the second; the most recent survives
- Contradictions in the `social` category (e.g., who the user is dating) → resolved silently without surfacing; personal relationship facts are too sensitive for Noxis to narrate back

## Dependencies

- S-003-003 (memory extraction)
- S-003-004 (decision policy)

## Technical Requirements

- **`ContradictionResolver.swift` (backend)** — `resolve(existing: Memory, incoming: MemoryCandidate) async throws -> Resolution` where `Resolution = .replaceExisting | .keepExisting | .silent`; `replaceExisting` triggers `MemoryService.deleteMemory(existing.id)` then `MemoryService.addMemory(incoming)`
- **Conflict detection in extraction:** `MemoryService.addMemory` runs a pre-write search for semantically similar entries (cosine similarity > 0.85); if found and the content differs directionally, it routes to `ContradictionResolver` instead of writing directly
- **Surfacing injection:** `ContradictionResolver` returns a `SurfacingNote?` — a short string appended to the pipeline's `moduleContext` layer for the current response only; it is consumed once and not stored
- **Surfacing gate check:** `shouldSurface(old: Memory, new: MemoryCandidate) -> Bool` — checks `daysSinceUpdate(old) >= 14 && new.confidence >= 0.70 && surfaceableCategories.contains(new.category)`
- **`contradiction_resolutions` table:** `(id UUID, user_id STRING, old_memory_id UUID, new_memory_id UUID, old_content TEXT, new_content TEXT, resolved_at TIMESTAMP, surfaced BOOL, surfacing_response_id UUID?)`
- **Background retry:** a `ProcessingJob` runs every 6 hours to retry unresolved deletions from the dead-letter list

## Future Scope

- User-facing contradiction acknowledgement: show a brief toast "Noxis updated something it knows about you" with a link to the Memory Review screen
- Conflict visualization in the Memory Review screen — show resolved contradictions as a log
- Configurable sensitivity: user can set "memory update sensitivity" to low/medium/high, affecting whether surfacing fires or stays silent

## Do Not Do

- In this story, do not implement the Memory Review UI — that is S-003-007
- Do not surface contradictions in the `social` category — keep personal relationship details private
- Do not implement the silent filter-layer resolution separately — the decision policy (S-003-004) handles that; this story owns the extraction-layer resolution and user-facing surfacing only

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |
