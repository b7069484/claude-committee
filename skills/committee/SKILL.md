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

### Phase 0: INTAKE & MODE SELECTION

This is the FIRST interaction with the user after reading the deliverable. The mode question comes FIRST — before any roster, composition, or configuration.

1. Read `meta-agent.md`, adopt the Executive Assistant persona.
2. Determine intent (review / suggest / list / add / promote / remove / custom).
3. Read the deliverable — files via Read tool, or extract from conversation context.
4. **FIRST QUESTION — mode selection** (unless `--interactive` or `--standard` flag was passed):

```
How would you like to run this review?
1. Standard review — I handle everything and deliver a complete report [recommended]
2. Interactive session — we configure together, then debate key topics step by step
```

5. **Wait for the user's answer.** Do not present any roster, composition, or analysis yet.
6. Mode locked. Write `SESSION.json` checkpoint: `mode_selected` (see `session-schema.md`).

> If `SESSION.json` already exists for this topic, ask: "Resume or start fresh?" before the mode question.

---

### STANDARD MODE FLOW

After the user selects Standard mode, the meta agent composes the committee, presents it for ONE confirmation, then runs the entire review automatically from there.

#### Phase S1: COMPOSE & CONFIRM ROSTER

1. If a collective was specified: read from `collectives/[id].md`. If suggest: analyze context, read `collectives/_index.md`, find best match.
2. Resolve pinned members from `favorites/`.
3. Fill dynamic slots: read `generation-guide.md`, run domain coverage mapping, generate members, validate constraints.
4. Assign report tiers by deliverable relevance (full / focused / flag-only).
5. Detect blind spots — include the most critical gap member(s) in the proposal.
6. **Present the full proposed roster** grouped by tier (FULL REPORTS / FOCUSED REPORTS / FLAG-ONLY), with blind spot additions marked. Include cost estimate.
7. **Wait for user confirmation.** User can approve, swap members, adjust tiers, or add/remove members. This is the ONLY question in standard mode.
8. Roster locked. Write `SESSION.json` checkpoint: `roster_locked`.

#### Phase S2: AUTO-REPORT (No Further User Input)

Everything from here is fully automated — no more questions.

1. Read `agent-mode.md`. Prepare deliverable context. Construct tiered prompts.
2. Spawn ALL member sub-agents in a single message (parallel).
3. Collect reports. Check quorum (60%). If below quorum, retry failed members once automatically, then proceed with what's available.
4. Write `SESSION.json` checkpoint: `reports_complete`.

If `--sequential` flag: adopt each persona in main context sequentially instead of spawning sub-agents.

#### Phase S2: SYNTHESIS & NEXT STEPS

Read `review-format.md` and `next-steps-format.md`. Produce the complete output as a single continuous document:
- Executive Summary
- Committee Roster (with tiers and any auto-added blind spot members noted)
- Individual Reports (grouped by tier: Full → Focused → Flags → Unavailable)
- Synthesis: Consensus Points, Key Tensions, Evidence & Benchmarks
- Blind Spots & Recommended Additions
- Next Steps: Path A (full adoption) and Path B (selective adoption with tensions surfaced)

Write `SESSION.json` checkpoint: `synthesis_complete`. Save review to `./committee-review-YYYY-MM-DD-[topic].md`. Proceed to Phase 3 (Implementation Bridge).

---

### INTERACTIVE MODE FLOW (Step-by-Step Configuration)

After the user selects Interactive mode, the meta agent walks through each configuration step ONE AT A TIME. Each step is its own message. Lock each answer before moving to the next.

#### Phase I1: ROSTER COMPOSITION

1. If a collective was specified: read from `collectives/[id].md`. If suggest: analyze context, find best match.
2. Resolve pinned members. Fill dynamic slots (read `generation-guide.md`).
3. Assign report tiers by deliverable relevance.
4. **Present the roster** grouped by tier (FULL REPORTS / FOCUSED REPORTS / FLAG-ONLY) with rationale per member.
5. **Wait for user confirmation.** User may swap members, adjust tiers, or approve.
6. Roster locked. Write `SESSION.json` checkpoint: `roster_locked`.

#### Phase I2: BLIND SPOT DETECTION

1. Run domain coverage mapping against the locked roster.
2. **Present gaps** as lettered options under "COVERAGE GAPS DETECTED:".
3. **Wait for user response.** User types letters to add, "all", or "none".
4. If members added: generate them, assign tiers, confirm.
5. Blind spots resolved. Write `SESSION.json` checkpoint: `blindspots_resolved`.

#### Phase I3: COST ESTIMATE & CONFIRMATION

1. Display cost estimate: member count, tier breakdown, estimated sub-agent cost, estimated main context tokens.
2. **Wait for user to confirm** they're ready to proceed.
3. Write `SESSION.json` checkpoint: `composition_complete`.

#### Phase I4: PRELIMINARY QUESTIONS

1. Read `agent-mode.md`. Spawn all committee members as Haiku sub-agents. Each generates 2-3 clarifying questions.
2. Collect, deduplicate, prioritize by cross-member relevance.
3. **Present as a numbered list.** User answers or says "skip."
4. Store answers. Write `SESSION.json` checkpoint: `questions_complete`.

This phase is skippable — user can say "skip questions, go to reports."

#### Phase I5: AGENDA PREVIEW

1. Synthesize deliverable + preliminary answers + roster → prioritized topic agenda (high / medium / low priority).
2. **Present agenda to user.** They can reorder, add, or remove topics.
3. Agenda locked on confirmation. Write `SESSION.json` checkpoint: `agenda_complete`.

#### Phase I6: INDIVIDUAL REPORTS

1. Read `agent-mode.md`. Prepare deliverable context. Construct tiered prompts.
2. Include preliminary answers and locked agenda in each prompt.
3. Spawn ALL member sub-agents in a single message (parallel).
4. Collect reports. Check quorum (60%). If below quorum, present retry/proceed/abort options.
5. Present reports grouped by tier: Full → Focused → Flags → Unavailable.
6. Write `SESSION.json` checkpoint: `reports_complete`.

If `--sequential` flag: adopt each persona in main context sequentially.

#### Phase I7: EXECUTIVE BRIEFING

1. Read all reports. Produce a briefing: consensus areas, key tensions, blind spots, recommended debate topics.
2. **Present briefing to user.** Offer to add members for any new blind spots.
3. Present debate agenda (updated with report findings).
4. Write `SESSION.json` checkpoint: `briefing_complete`.

#### Phase I8: INTERACTIVE DEBATE ROUNDS

Read `debate-format.md`. For each agenda topic in priority order:
- Check for consensus fast-track (if all relevant members agree, present quick confirmation)
- Run full debate round if needed
- **Checkpoint after each round** — wait for user response
- Write `SESSION.json` per round: `debate_round_N`

After all rounds or user exits: produce FULL synthesis (same structure as standard, enriched with debate proceedings). Save review to `./committee-review-YYYY-MM-DD-[topic].md`. Write `SESSION.json` checkpoint: `synthesis_complete`. Proceed to Phase 3.

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
