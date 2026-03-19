# Architecture Task: Identity Layer File Structure (SOUL.md + Tones + Modules)

---

**ID:** AT-001-003
**Release:** R-001
**Stage:** Backlog
**Created:** 2026-03-18
**Updated:** 2026-03-18
**Extends:** None
**Depends On:** AT-001-002
**Version:** 1.0

---

## What

Populates the backend `identity/` directory with all fixed personality and behavioral files that the orchestration pipeline loads at runtime. These files are the source of Noxis's character — `SOUL.md` is loaded as the system prompt on every API call; tone files and module files are injected alongside it. Without this artifact, S-002-001 (load SOUL.md) and S-004-005 (orchestration pipeline) cannot be implemented, and no other story that produces an AI response will have a correct system prompt to test against. The source material for all files lives in `requirements/SOUL.md` and `requirements/AGENTS.md` — this AT translates those PM artifacts into the exact runtime file structure the backend expects.

## Artifact Location

```
backend/
  identity/
    SOUL.md                       ← fixed personality layer, full content
    tones/
      brother.md                  ← Sharp Older Brother register + vocabulary rules
      consultant.md               ← High-End Consultant register + vocabulary rules
      peer.md                     ← Confident Peer register + vocabulary rules
    modules/
      wardrobe.md                 ← judgment rules + memory read/write spec
      gym.md
      food.md
      spending.md
      routines.md
      social.md
      general.md
    boundaries/
      therapy.md                  ← in-character redirect per tone mode
      medical.md
      legal.md
      political.md
  services/
    prompt_builder.py             ← build_system_prompt() function
```

## Steps

1. `[Agent]` `backend/identity/SOUL.md` is created by copying the full content of `requirements/SOUL.md` verbatim — no edits, no paraphrasing; this file is version-controlled and must match exactly

2. `[Agent]` `backend/identity/tones/brother.md`, `consultant.md`, and `peer.md` are created from the tone sections of `requirements/SOUL.md` — each file contains: the tone's name, its register rules, vocabulary constraints, and 2–3 sample exchanges that illustrate the voice difference; content is derived entirely from the existing SOUL.md material

3. `[Agent]` `backend/identity/modules/*.md` — 7 module files created (wardrobe, gym, food, spending, routines, social, general); each file contains the module's judgment framework, memory read/write rules, and routing triggers as specified in `requirements/AGENTS.md`; content is derived entirely from existing PM docs

4. `[Agent]` `backend/identity/boundaries/*.md` — 4 boundary files created (therapy, medical, legal, political); each contains the in-character redirect instructions per tone mode as specified in `requirements/SOUL.md` hard boundaries section

5. `[Agent]` `services/prompt_builder.py` implements `build_system_prompt(tone_mode: str, module: str, memories: list) -> str` exactly as specified in `arch.md` System Prompt Assembly section; unit test confirms that calling with `tone_mode="brother"`, `module="wardrobe"`, `memories=[]` returns a string containing the contents of SOUL.md, brother.md, and wardrobe.md

## Acceptance Criteria

- [ ] `backend/identity/SOUL.md` content matches `requirements/SOUL.md` exactly (byte-for-byte or character-for-character — no reformatting)
- [ ] All 3 tone files exist: `brother.md`, `consultant.md`, `peer.md`; each file is non-empty and contains distinct content
- [ ] All 7 module files exist and are non-empty
- [ ] All 4 boundary files exist and are non-empty
- [ ] `build_system_prompt(tone_mode, module, memories)` returns a string that contains substrings from `SOUL.md`, the requested tone file, and the requested module file — verifiable with a unit test
- [ ] `build_system_prompt` raises `ValueError` if `tone_mode` is not one of `['brother', 'consultant', 'peer']`
- [ ] `build_system_prompt` raises `ValueError` if `module` is not one of `['wardrobe', 'gym', 'food', 'spending', 'routines', 'social', 'general']`
- [ ] No hardcoded personality text exists in any Python source file — all personality content comes from the identity files at runtime

## Depends On

AT-001-002 (backend scaffold must exist for `services/prompt_builder.py` to live in)

---

## Change Log

| Date | Version | Author | Change |
|------|---------|--------|--------|
| 2026-03-18 | 1.0 | — | Created |
