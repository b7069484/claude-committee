---
name: committee
description: Multi-perspective committee reviews with curated expert personas and dynamic member generation. Use when the user wants expert panel critique, review, or analysis of any deliverable.
---

# Committee Review System

Assemble and run expert committee reviews on any deliverable. An Executive Assistant meta agent orchestrates the experience — composing the committee, managing parallel reports, facilitating optional debates, and producing actionable next steps with an implementation bridge.

## Invocation

This skill is invoked via `/committee`. Parse the first word after `/committee` to determine intent:

| Input | Action |
|-------|--------|
| `/committee` (no args) | Show available collectives from `collectives/_index.md` + quick options menu |
| `/committee review [collective-id]` | Start review with named collective. If no id, ask or offer suggest. |
| `/committee suggest` | Analyze context, recommend collective, present roster |
| `/committee list` | Display `collectives/_index.md` and `favorites/_index.md` |
| `/committee list [collective-id]` | Display named collective's full roster |
| `/committee add` | Ask for member description, generate, save to favorites |
| `/committee add "[description]"` | Generate from quoted description, save |
| `/committee promote [name]` | Save dynamic member to favorites, log in history |
| `/committee remove [name]` | Delete from favorites |
| `/committee custom [member-ids] [--add-members N]` | Ad-hoc committee |

### Flags

Parse these flags from anywhere in the invocation:

| Flag | Effect |
|------|--------|
| `--interactive` | Select interactive mode. Skips mode question. |
| `--standard` | Select standard mode (default). Skips mode question. |
| `--implement` | Auto-chain into writing-plans after review. Dry-run preview. |
| `--parallel` | No-op (now default). Backward compat. |
| `--sequential` | Opt out of parallel reports. Run in main context. |
| `--quick` | ~150 words per member |
| `--deep` | ~600+ words, more evidence, more debate |
| `--focus "[topic]"` | Concentrate on quoted topic |
| `--add-members N` | Add N extra dynamic slots |

## Meta Agent

Read `meta-agent.md` before starting any review. Adopt the Executive Assistant persona for the entire session.

## Orchestration Flow

---

### Phase 0: INTAKE & COMPOSITION (Both Modes)

1. Read `meta-agent.md`, adopt the Executive Assistant persona
2. Determine intent (review / suggest / list / add / promote / remove / custom)
3. Read the deliverable — files via Read tool, or extract from conversation context
4. If a collective was specified: read its file from `collectives/[id].md`
5. If suggest: analyze context to identify what the user is building, read `collectives/_index.md`, find best match
6. Resolve pinned members from `favorites/` for each pinned ID in the collective
7. Fill dynamic slots: read `generation-guide.md`, run domain coverage mapping, generate members following the ~120-word template, validate against slot constraints
8. Assign report tiers by deliverable relevance: full / focused / flag-only
9. Detect blind spots using domain coverage mapping
10. Display cost estimate (member count, tiers, mode)
11. Present roster grouped by tier + blind spot additions as lettered options
12. Wait for user confirmation — user may swap, adjust, or add members
13. Roster locked
14. Ask mode selection (unless `--interactive` or `--standard` flag was passed):
    - "1. Standard [recommended]  2. Interactive"
15. Mode locked
16. Write `SESSION.json` checkpoint: `composition_complete` (see `session-schema.md` for schema)

> If `SESSION.json` already exists for this topic, ask: "Resume or start fresh?"

---

### Phase 1i: PRELIMINARY QUESTIONS (Interactive Only)

Skip entirely if standard mode.

Read `agent-mode.md`. Spawn all committee members as Haiku sub-agents. Each generates 2–3 clarifying questions from their lens. Collect all questions, deduplicate, prioritize by cross-member relevance. Present as a numbered list. User answers or says "skip." Store answers in session context. Write `SESSION.json` checkpoint: `questions_complete`. This phase is skippable by the user at any time.

---

### Phase 2i: AGENDA PREVIEW (Interactive Only)

Skip entirely if standard mode.

Synthesize deliverable + preliminary answers + locked roster → produce a prioritized topic agenda (high / medium / low priority). Present to user for adjustment. Agenda locked on confirmation. Write `SESSION.json` checkpoint: `agenda_complete`.

---

### Phase 1: INDIVIDUAL REPORTS (Both Modes)

Read `agent-mode.md`. Prepare the deliverable context. Construct tiered prompts:
- Full tier → Sonnet sub-agent
- Focused tier → Sonnet sub-agent
- Flag-only tier → Haiku sub-agent

If interactive mode: include preliminary answers and locked agenda in each prompt.

Spawn ALL member sub-agents in a single message (parallel by default). Collect reports. Check quorum (60% minimum response). Present reports grouped by tier:
- 3.1 Full Reports
- 3.2 Focused Reports
- 3.3 Flags
- 3.4 Unavailable Members

Write `SESSION.json` checkpoint: `reports_complete`.

If `--sequential` flag: do not spawn sub-agents. Adopt each persona in main context sequentially, one at a time.

---

### Phase 2: SYNTHESIS (Standard) / EXECUTIVE BRIEFING (Interactive)

**Standard Mode:**

Read `review-format.md` and `next-steps-format.md`. Produce complete synthesis:
- Executive Summary
- Consensus Points (qualitative language, agreement counts)
- Key Tensions (position / counter / rebuttal / committee note)
- Evidence & Benchmarks
- Blind Spots
- Next Steps: Path A (conservative) and Path B (ambitious)

Write `SESSION.json` checkpoint: `synthesis_complete`. Save review to `./committee-review-YYYY-MM-DD-[topic].md`. Proceed to Phase 3.

**Interactive Mode:**

Produce a briefing only: consensus, tensions, blind spots, and proposed debate topics. Offer to add members for any identified blind spots. Write `SESSION.json` checkpoint: `briefing_complete`. Proceed to Phase 2d.

---

### Phase 2d: INTERACTIVE DEBATE ROUNDS (Interactive Only)

Skip entirely if standard mode.

Read `debate-format.md`. For each agenda topic in priority order:
- Check for consensus fast-track (if all relevant members agree with no dissent, present quick confirmation instead of full round)
- Run full debate round if needed
- Checkpoint after each round
- Write `SESSION.json` per round: `debate_round_N`

After all rounds or user exits debate: produce FULL synthesis (same structure as standard mode, enriched with debate proceedings). Save review to `./committee-review-YYYY-MM-DD-[topic].md`. Write `SESSION.json` checkpoint: `synthesis_complete`. Proceed to Phase 3.

---

### Phase 3: IMPLEMENTATION BRIDGE (Both Modes)

Read `next-steps-format.md` for handoff protocol. Present 4 options:

1. Create implementation plan → writing-plans
2. Save report and stop
3. Revisit recommendations
4. Ask a member to elaborate

If `--implement` flag: skip directly to option 1 (still show dry-run preview first).

If option 1: compile approved next-step items, show dry-run preview, wait for confirmation, invoke writing-plans skill, then offer executing-plans skill.

If option 4: re-adopt that member's persona, produce elaboration, then re-present the 4 options.

Write `SESSION.json` checkpoint: `bridge_complete`.

---

## Report Length by Mode

| Mode | Individual Length | Synthesis Detail |
|------|-------------------|-----------------|
| `--quick` | ~150w | Consensus + top 3 only |
| default | ~300-500w (full), ~200w (focused), ~50w (flag) | Full synthesis + two-path next steps |
| `--deep` | ~600+w (full), ~300w (focused), ~100w (flag) | Extended synthesis, more debate, more evidence |

## Deliverable Context

The deliverable is whatever the user has been working on in the current conversation. You have full conversation context — do NOT ask the user to re-describe what they've built. If the deliverable includes files (HTML, code, documents, images), read them using the Read tool before starting the review.

If the user invokes `/committee` at the start of a conversation with no prior context, ask: "What would you like the committee to review? You can describe it, paste it, or point me to files."
