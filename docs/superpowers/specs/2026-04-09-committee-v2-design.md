# Committee v2 Design Spec — Executive-Facilitated Reviews

**Date:** 2026-04-09 (revised after committee review)
**Author:** Israel Bitton
**Status:** Draft — awaiting final approval

---

## Summary

Committee v2 introduces an Executive Assistant meta agent that orchestrates the entire review experience. The default path is fast: compose committee → parallel reports → synthesis → actionable next steps → optional implementation handoff. Users who want deeper engagement can opt into Interactive mode, which adds preliminary questions, an agenda preview, and topic-based debate rounds with user checkpoints. All committee work runs through model-tiered sub-agents (Haiku/Sonnet), preserving the main context window for the meta agent's orchestration (Opus). A durable session checkpoint system ensures reviews are resumable, auditable, and resilient to failures.

## Goals

1. **Fast by default** — the standard review path delivers a complete synthesis and actionable next steps without requiring user participation in the process. Value first, ceremony opt-in.
2. **Facilitated experience** — a meta agent manages the review end-to-end, presenting decisions as structured options (numbered lists, Y/N, A/B/C)
3. **Blind spot detection** — proactively identify missing perspectives with coverage-mapped detection and easy accept/reject additions
4. **Two review modes** — Standard (fast, reports → synthesis → next steps) and Interactive (adds preliminary questions, agenda, topic-based debates with checkpoints). User chooses at the start.
5. **Actionable output** — final document ends with two-path next steps splitting unanimous recommendations from contentious items, with tensions surfaced prominently
6. **Implementation bridge** — the headline feature of v2. `--implement` flag chains approved recommendations directly into writing-plans → executing-plans with dry-run preview
7. **Token efficiency** — model tiering (Haiku/Sonnet/Opus), tiered report depth by deliverable relevance, compressed debate context, agenda-as-budget
8. **Resilience** — durable session checkpoints, sub-agent failure handling with quorum enforcement, resumable reviews
9. **Backward compatibility** — all v1 commands still work, enhanced behavior is additive

## Non-Goals

- Full agent swarm architecture (fights Claude Code's architecture, TOS risk)
- Real-time interjection during generation (not supported by Claude Code's turn model)
- Persistent committee state across sessions (each review is self-contained, though SESSION.json enables within-session resume)
- Changes to the existing 26 favorite personas or 7 collectives
- Exposing model names (Haiku/Sonnet/Opus) to end users — the spec uses them for implementation clarity, but the user-facing interface uses Standard/Deep/Express terminology

---

## Architecture

### Approach: Hybrid — Parallel Reports, Sequential Debates

- **Meta agent** runs in the main context window (Opus). It is the sole interface the user interacts with.
- **Individual reports** run as parallel sub-agents (Sonnet/Haiku). Truly independent — no bleed-through between members.
- **Debate rounds** (interactive mode only) run sequentially in the main context. The meta agent voices committee members based on their reports and persona definitions. No sub-agents spawned for debates.
- **Preliminary questions** (interactive mode only) run as parallel sub-agents (Haiku). Cheap probes to gather context before the report phase.

### Model Tiering

| Model | Used For | Approx Cost Per Member |
|-------|----------|----------------------|
| **Haiku** | Preliminary questions, flag-only scans, dynamic member generation | ~$0.001 |
| **Sonnet** | Full individual reports, focused individual reports | ~$0.01 |
| **Opus** | Meta agent (main context only): synthesis, briefing, debate facilitation, next steps | Main context — no per-member cost |

The meta agent assigns report depth tiers during Phase 0 (Composition) based on **deliverable relevance** — how directly the member's domain overlaps the deliverable — not roster balance. The user can override any assignment.

### Concrete Cost Model

For a typical 10-member committee (4 full, 4 focused, 2 flag-only) in Standard mode:

| Component | Tokens (est.) | Cost (est.) |
|-----------|--------------|-------------|
| Phase 0: Composition (Opus, main context) | ~2,000 | Included in main context |
| Phase 1: Reports — 4 full (Sonnet, ~1,500 tokens each) | ~6,000 | ~$0.02 |
| Phase 1: Reports — 4 focused (Sonnet, ~800 tokens each) | ~3,200 | ~$0.01 |
| Phase 1: Reports — 2 flag-only (Haiku, ~200 tokens each) | ~400 | ~$0.0004 |
| Phase 2: Synthesis + Next Steps (Opus, main context) | ~4,000 | Included in main context |
| **Sub-agent total** | **~9,600** | **~$0.03** |
| **Main context (Opus) total** | **~15,000-20,000** | **Subscription/usage** |

Interactive mode adds:
| Component | Tokens (est.) | Cost (est.) |
|-----------|--------------|-------------|
| Preliminary Questions (10 Haiku sub-agents) | ~3,000 | ~$0.003 |
| Debate rounds (3-4 rounds in Opus main context) | ~6,000-8,000 | Included in main context |

**Note:** The primary cost is the Opus main context, which is part of the user's Claude Code subscription or usage. Sub-agent costs are minimal. The model tiering primarily saves by keeping report generation in Sonnet rather than consuming Opus main context tokens.

### Session Checkpointing (Durable State)

Every phase transition writes a structured `SESSION.json` to disk alongside the review output. This is the backbone of v2's resilience.

**Schema:**

```json
{
  "version": "2.0",
  "session_id": "committee-2026-04-09-ai-code-review-bot",
  "created_at": "2026-04-09T14:30:00Z",
  "updated_at": "2026-04-09T14:45:00Z",
  "phase_cursor": "reports_complete",
  "mode": "standard",
  "deliverable_summary": "AI code review bot — product design spec",
  "roster": {
    "members": [
      {
        "id": "stripe-engineering-head",
        "name": "Head of Engineering at Stripe",
        "tier": "full",
        "model": "sonnet",
        "status": "report_complete"
      }
    ],
    "blind_spots_offered": ["enterprise-buyer", "pricing-specialist"],
    "blind_spots_accepted": ["pricing-specialist"]
  },
  "preliminary_answers": {},
  "agenda": [],
  "reports": {
    "stripe-engineering-head": {
      "status": "complete",
      "summary": "...(~50 words)...",
      "full_text_ref": "section_3_1"
    }
  },
  "debate_rounds": [],
  "decisions": {
    "confirmed": [],
    "open": [],
    "contentious": []
  },
  "cost_estimate": {
    "sub_agent_tokens": 9600,
    "main_context_tokens": 18000
  }
}
```

**Checkpoint rules:**
- Written to `./committee-session-YYYY-MM-DD-[topic].json` after each phase completes
- If the main context is interrupted, a new session can bootstrap from the last checkpoint
- The meta agent reads from SESSION.json to reconstruct state rather than relying solely on context memory
- Enables debugging: any session can be inspected after the fact

### Sub-Agent Failure Handling

Every parallel sub-agent spawn includes explicit failure semantics:

**Timeout:** Each sub-agent has a 120-second timeout. If a member's report doesn't return within that window, it's marked as `timed_out` in SESSION.json.

**Minimum quorum:** The meta agent proceeds to synthesis only if **at least 60% of members** returned reports. For a 10-member committee, that's 6 reports minimum. Below quorum, the meta agent informs the user and offers:
```
3 of 10 members failed to return reports (Stripe Engineering, VC Partner, Accessibility Advocate).

Options:
1. Retry the 3 failed members
2. Proceed with 7/10 reports (below recommended quorum)
3. Abort and save partial results
```

**Fallback stubs:** Failed members are included in the final document as:
```
### [N]. Head of Engineering at Stripe
**Status:** Report unavailable (timed out). This perspective is missing from the synthesis.
```

**Disclosure:** The synthesis always discloses missing members. If members are missing, the meta agent explicitly notes: "This synthesis is based on [N]/[total] reports. The following perspectives are absent: [list]."

---

## Phase Flow

### Two Modes — User Chooses at Start

After roster composition (Phase 0), the meta agent asks:

```
How would you like to run this review?
1. Standard review — reports → synthesis → next steps [recommended]
2. Interactive session — preliminary questions → agenda → reports → debates → next steps

You can also pass --interactive or --standard as a flag to skip this question.
```

**Standard mode** (default): Phase 0 → Phase 1 → Phase 2 → Phase 3 (Implementation Bridge)
**Interactive mode**: Phase 0 → Phase 1i → Phase 2i → Phase 1 → Phase 2 → Phase 2d → Phase 3

For clarity, phases are numbered by their execution order within each mode:

### Standard Mode Phases

| Phase | Name | Context |
|-------|------|---------|
| 0 | Intake & Composition | Main (Opus) |
| 1 | Individual Reports | Parallel sub-agents (Sonnet/Haiku) |
| 2 | Synthesis & Next Steps | Main (Opus) |
| 3 | Implementation Bridge | Main (Opus) |

### Interactive Mode Phases

| Phase | Name | Context |
|-------|------|---------|
| 0 | Intake & Composition | Main (Opus) |
| 1i | Preliminary Questions | Parallel sub-agents (Haiku) |
| 2i | Agenda Preview | Main (Opus) |
| 1 | Individual Reports | Parallel sub-agents (Sonnet/Haiku) |
| 2 | Executive Briefing | Main (Opus) |
| 2d | Interactive Debate Rounds | Main (Opus) |
| 3 | Synthesis, Next Steps & Implementation Bridge | Main (Opus) |

---

### Phase 0: Intake & Composition (Both Modes)

**Context:** Main context, Opus (meta agent)

1. User invokes `/committee review [collective-id]` or `/committee suggest`
2. Meta agent reads the deliverable (files, docs, or description provided by user)
3. Meta agent analyzes: domain, product stage, target audience, complexity
4. Meta agent proposes committee roster with **relevance-based tier assignments**:
   - **Full report** (Sonnet) — 3-4 members whose domain directly overlaps the deliverable
   - **Focused report** (Sonnet) — 3-4 members with relevant secondary lens
   - **Flag-only** (Haiku) — 2-3 specialist scanners (accessibility, security, compliance, etc.)
   - Tier assignment is based on how directly the member's expertise maps to the deliverable, not roster balance
5. Meta agent detects blind spots using **domain coverage mapping**:
   - Maps each roster member to a two-level taxonomy: **industry** (e.g., fintech, devtools, healthcare) × **function** (e.g., engineering, design, security, business)
   - Identifies empty cells that the deliverable touches
   - Presents gaps as specific, actionable suggestions — not vague "something might be missing"
6. Presents blind spot additions as easy accept/reject: lettered list, user types letters to add or "all"
7. **Cost estimate displayed:**
   ```
   Estimated review: 10 members (4 full, 4 focused, 2 flag-only)
   Sub-agent cost: ~$0.03 · Main context: ~15-20k tokens
   ```
8. User confirms roster, swaps members, or adjusts tiers
9. Meta agent asks: Standard review or Interactive session?
10. **Roster locked. Mode locked. SESSION.json checkpoint written.**

**Interaction example:**
```
Meta Agent:
Here's the proposed committee for your AI code review bot:

FULL REPORTS:
  1. Head of Engineering, Stripe — API design, reliability
  2. Head of AI, Meta (FAIR) — model integration, inference
  3. Head of DX, Vercel — developer experience, onboarding
  4. Skeptic End User — will this actually get used?

FOCUSED REPORTS:
  5. Head of Product, OpenAI — competitive positioning
  6. Head of Security Engineering, Google — code scanning risks
  7. VC Partner — market viability, funding angle

FLAG-ONLY:
  8. Regulatory & Compliance — data handling flags
  9. Accessibility Advocate — UI accessibility scan

COVERAGE GAPS DETECTED:
  A. No enterprise buyer perspective — add Head of IT Procurement, Fortune 500?
  B. No pricing/monetization specialist — add Head of Growth, PLG SaaS?

Estimated cost: ~$0.03 sub-agent · ~18k main context tokens

Add gap members? (A, B, all, or none)
Then: Standard review or Interactive session? (1/2)
```

---

### Phase 1i: Preliminary Questions (Interactive Mode Only)

**Context:** Parallel sub-agents, Haiku

1. Each committee member is spawned as a Haiku sub-agent
2. Each receives: their persona definition + the deliverable
3. Each proposes 2-3 questions they want answered before their full review
4. Meta agent collects all questions, deduplicates, and prioritizes
5. Presents to user as a numbered list (typically 6-10 unique questions after dedup)
6. User answers (can skip questions with "N/A" or answer in bulk)
7. Answers stored for distribution to members in Phase 1
8. **SESSION.json checkpoint written**

**Purpose:** Enriches the report phase with the user's real-world context — constraints, decisions already made, strategic priorities. This phase costs almost nothing (Haiku) but significantly improves report quality by eliminating assumptions.

**Skippable:** User can say "skip questions, go straight to reports" at any point.

---

### Phase 2i: Agenda Preview (Interactive Mode Only)

**Context:** Main context, Opus (meta agent)

1. Meta agent synthesizes: deliverable analysis + user answers + roster composition
2. Generates a prioritized topic agenda:
   - Key decisions the deliverable faces
   - Critical risks identified
   - Open questions that need resolution
   - Opportunities spotted
3. Assigns priority per topic: **high** (full debate round), **medium** (brief discussion), **low** (quick confirmation)
4. Presents agenda to user — user can reorder, add, remove topics
5. **Agenda locked. SESSION.json checkpoint written.**

**Note:** This agenda is a starting point, not the full scope. Committee members or the user can add topics during debates.

---

### Phase 1: Individual Reports (Both Modes)

**Context:** Parallel sub-agents, Sonnet (full + focused) / Haiku (flag-only)

1. All members spawned as parallel sub-agents **in a single message** (critical for true parallelism and independence)
2. Each agent receives:
   - Their persona definition (from favorites/ or dynamically generated)
   - The full deliverable
   - User's answers to preliminary questions (interactive mode) or empty (standard mode)
   - The agenda (interactive mode) or no agenda (standard mode — members structure their own analysis)
3. Report depth by tier:
   - **Full report** (Sonnet, 400-500 words): executive assessment, key issues with rationale and evidence, pros/cons, suggestions with benchmarks, "what I'd ship instead"
   - **Focused report** (Sonnet, 200 words): executive assessment, top 2 issues through their specific lens, key suggestion
   - **Flag-only** (Haiku, 50 words): scan for their specialty, report only if they find something. ("No accessibility concerns" is a valid one-line output. "FLAG: [specific issue]" when something is found.)
4. Reports collected back to main context
5. **Quorum check:** If fewer than 60% returned, meta agent pauses and offers retry/proceed/abort (see Failure Handling)
6. Meta agent reads all reports
7. **SESSION.json checkpoint written with per-member report status**

---

### Phase 2: Synthesis & Next Steps (Standard Mode) / Executive Briefing (Interactive Mode)

**Context:** Main context, Opus (meta agent)

#### Standard Mode: Full Synthesis

The meta agent produces the complete synthesis in one pass:

1. **Executive Summary** — 200-word high-level synthesis: overall verdict, biggest opportunities, biggest risks
2. **Consensus Points** — issues where a strong majority converged, with member counts and evidence. Uses qualitative language ("all members agree," "strong majority," "split") rather than percentage thresholds
3. **Key Tensions** — genuine disagreements with both sides' positions, reasoning, and what each side optimizes for
4. **Evidence & Benchmarks** — real-world data, case studies, and benchmarks cited across reports
5. **Blind Spots & Recommended Additions** — what the committee couldn't evaluate, who to add
6. **Next Steps** — two-path output (see Next Steps Engine below)

Then proceeds to Phase 3 (Implementation Bridge).

#### Interactive Mode: Executive Briefing

The meta agent produces a briefing to frame the debate:

1. Consensus areas, key tensions, blind spots, recommended debate topics
2. If blind spots are detected, offers to add members for a follow-up mini-review (spawns additional sub-agents, updates briefing)
3. Presents debate agenda (from Phase 2i, updated with report findings)
4. Proceeds to Phase 2d (Debates)

**SESSION.json checkpoint written.**

---

### Phase 2d: Interactive Debate Rounds (Interactive Mode Only)

**Context:** Main context, Opus (meta agent voices members)

**Structure:** Topic-based rounds. Each round focuses on one agenda topic.

1. Meta agent introduces the topic and its priority
2. Selects 3-4 relevant members to speak (based on their reports and expertise)
3. Meta agent voices each selected member, with these guardrails:
   - **Grounded in reports:** Every position must trace back to something the member said in their independent report. Meta agent does NOT invent new positions.
   - **Steelman instruction:** When voicing a dissenting member, the meta agent explicitly steelmans the strongest version of their opposition — not a strawman. This counteracts RLHF consensus-seeking.
   - **Transparency tags:** Each voiced position is tagged `[from report]` when directly from the member's Phase 1 analysis, or `[extended]` when the meta agent is extrapolating from their persona and analytical lens. This lets the user assess which debate points are grounded vs. generated.
4. Meta agent synthesizes the round: where members agree, where they diverge, what's at stake

**Checkpoint:** After each round, the meta agent pauses:
```
Round 2 complete: Pricing Model

Stripe Engineering and the VC disagree on freemium vs. usage-based.
McKinsey sides with usage-based for enterprise positioning.

Options:
1. Hear them debate this further
2. Add your constraints or context
3. You've decided — move on
4. Add a pricing specialist to weigh in
5. Switch to standard mode — wrap up with synthesis
```

**User can:**
- Add context or constraints the committee didn't know
- Challenge a specific position
- Confirm and move to the next topic
- Ask the meta agent to add a topic to the agenda
- Ask to hear more from a specific member
- **Switch to standard mode** — exit debates, proceed to synthesis with all debate context included

**Between rounds:** Meta agent updates the compressed running summary (~200 words). This summary carries forward, keeping context growth linear.

**Consensus fast-track:** If a topic has strong agreement from reports, the meta agent presents it as a quick confirmation:
```
Topic: API-First Architecture
All 4 relevant members agree this is the correct approach. No dissent.
Confirm and move on? (Y/n)
```

**Exit conditions:** Debates end when: (a) all agenda topics are covered, (b) user says "wrap it up," (c) user switches to standard mode, or (d) max rounds reached (configurable, default 8).

**SESSION.json checkpoint written after each round.**

After debates complete, the meta agent produces the full synthesis (same structure as Standard mode Phase 2, but enriched with debate proceedings) and proceeds to Phase 3.

---

### Phase 3: Implementation Bridge (Both Modes)

**Context:** Main context, Opus (meta agent)

This is the headline feature of v2 — the moment where review becomes action.

**After the final document is saved, the meta agent offers:**

```
Review complete and saved to ./committee-review-2026-04-09-ai-code-review-bot.md
Session saved to ./committee-session-2026-04-09-ai-code-review-bot.json

What would you like to do?
1. Create an implementation plan from your approved items → runs writing-plans
2. Save the report and stop here
3. Revisit specific recommendations before planning
4. Ask a committee member to elaborate on a point
```

**If user chooses 1 (or passes `--implement` flag):**

1. Meta agent compiles the approved items:
   - If Path A: all recommendations
   - If Path B: Section 1 items (unanimous) + any Section 2 items the user signed off on
2. **Dry-run preview** before handoff:
   ```
   Implementation plan will cover these 7 items:
   1. Redesign API authentication flow [unanimous, high priority]
   2. Add rate limiting middleware [unanimous, medium priority]
   3. Switch to usage-based pricing [contentious — you approved Position A]
   ...
   
   Proceed? (Y/n)
   ```
3. Invokes `superpowers:writing-plans` with the approved items as the spec input
4. After plan is written and approved by user, offers: "Plan approved. Ready to begin implementation?" → invokes `superpowers:executing-plans`

**Handoff contract** (the structured interface between committee output and writing-plans):

```json
{
  "source": "committee-review",
  "review_file": "./committee-review-2026-04-09-ai-code-review-bot.md",
  "session_file": "./committee-session-2026-04-09-ai-code-review-bot.json",
  "approved_items": [
    {
      "id": "rec-1",
      "action": "Redesign API authentication flow",
      "priority": 1,
      "consensus": "unanimous",
      "complexity": "high",
      "depends_on": [],
      "source_members": ["stripe-engineering-head", "google-security-head"],
      "implementation_hint": "Replace API key auth with OAuth 2.0 + JWT"
    }
  ]
}
```

**Handoff rules:**
- Only items explicitly approved by the user are passed to the plan
- Priority ordering from the committee synthesis is preserved
- Dependencies between items are included
- Each item traces back to specific committee members who recommended it
- The plan references the committee review document for full context

---

## Next Steps Engine

Every review (both modes) ends with a two-path next steps section. This is the primary actionable output.

### Path A: Full Adoption

All recommendations as ordered action items:

```markdown
### Path A: Full Adoption

1. **Redesign API authentication flow** — Stripe Engineering, Google Security
   Priority: High · Consensus: Unanimous · Complexity: High · Depends on: none
   
2. **Add rate limiting middleware** — Stripe Engineering, Netflix Platform
   Priority: High · Consensus: Unanimous · Complexity: Medium · Depends on: #1
   
3. **Switch to usage-based pricing** — McKinsey Strategy, VC Partner
   Priority: Medium · Consensus: Majority (6/10) · Complexity: High · Depends on: none
   ⚠️ Contentious: Stripe Engineering advocates freemium. See Tension #2.
```

### Path B: Selective Adoption

Surfaces tensions prominently — they're the most valuable signal, not something to bury:

```markdown
### Path B: Selective Adoption

**Ready to Act — Full Consensus:**
1. Redesign API authentication flow [all agree]
2. Add rate limiting middleware [all agree]
3. Implement structured logging [all agree]

**Requires Your Decision — Key Tensions:**

These are the items where the committee genuinely disagreed. Each represents a real strategic tradeoff.

**Tension 1: Pricing Model**
- Freemium (Stripe Engineering, Shopify Power User): Lower barrier to adoption. Evidence: Slack's freemium-to-enterprise pipeline converted 30% of free teams.
- Usage-based (McKinsey, VC Partner): Better unit economics, self-selecting for serious users. Evidence: Twilio's usage model drove 140% net revenue retention.
- Meta agent assessment: Freemium wins if your primary constraint is adoption speed. Usage-based wins if your primary constraint is unit economics. Your current stage (pre-launch) favors freemium.
- If you choose freemium: [specific implications and next steps]
- If you choose usage-based: [specific implications and next steps]
- Your decision: ___

**Tension 2: ...**
```

---

## The Meta Agent

### Persona Definition

**Role:** Chief of Staff / Executive Facilitator
**Tone:** Crisp, structured, decisive — like a senior McKinsey engagement manager
**Not:** A chatbot. Not deferential. Not verbose. Not a "yes-machine."

### Core Behaviors

1. **Summarizes, never dumps** — raw data stays in sub-agents. The user sees synthesized insights.
2. **Presents decisions as structured options** — numbered lists, A/B/C, Y/N. Never open-ended "what do you think?"
3. **Proactively flags what needs attention** — doesn't wait for the user to ask. If something is contentious, surfaces it.
4. **Manages the user's time** — knows when to compress ("All agree, confirm?") and when to expand ("This is a real disagreement, here's why it matters").
5. **Tracks running state via SESSION.json** — reads and writes the session file at each phase transition. Never relies solely on context memory.
6. **Honest about limitations** — when the committee can't evaluate something, says so and suggests who could. When members are missing from synthesis, discloses it.
7. **Steelmans opposition in debates** — when voicing dissenting members, presents the strongest version of their argument. Tags positions as `[from report]` or `[extended]` for transparency.

### Context It Maintains

The meta agent maintains a lean working state in the main context:

- Committee roster with tier assignments (~200 tokens)
- User's answers to preliminary questions, compressed (~100 tokens)
- Agenda with priorities (~150 tokens)
- Compressed report summaries (~50 words each = ~500 tokens for 10 members)
- Running debate summary (~200 words = ~300 tokens, updated between rounds)
- Decision log (~100 tokens)

**Total meta overhead: ~1,350 tokens.** Full report text is available in context from Phase 1 collection but is referenced via summaries during debates to keep working context lean.

The authoritative state is always SESSION.json on disk, not in-context memory.

### What It Delegates to Sub-Agents

- Dynamic member generation → Haiku sub-agents
- Preliminary questions → Haiku sub-agents (parallel, interactive mode)
- Individual reports → Sonnet/Haiku sub-agents (parallel, both modes)
- Flag-only scans → Haiku sub-agents (parallel, both modes)

### What It Handles In Main Context

- Committee composition, tier assignment, and blind spot detection
- Mode selection (standard vs. interactive)
- Agenda generation and prioritization (interactive mode)
- Executive briefing / full synthesis
- Debate facilitation with transparency tags (interactive mode)
- User checkpoints and interaction
- Final document assembly
- Next steps generation
- Implementation bridge handoff with dry-run preview
- Session checkpoint management

---

## File Structure

### New Files

| File | Purpose |
|------|---------|
| `skills/committee/meta-agent.md` | Executive assistant persona, behaviors, interaction patterns, context management rules, session checkpoint protocol |
| `skills/committee/debate-format.md` | Topic-based round structure, checkpoint format, steelman instructions, transparency tag rules, compressed summary rules, exit conditions |
| `skills/committee/next-steps-format.md` | Two-path action plan template, consensus classification (qualitative, not percentage), tension-prominent Path B format, implementation bridge handoff contract |
| `skills/committee/session-schema.md` | SESSION.json schema definition, checkpoint rules, failure recovery protocol, quorum enforcement |

### Modified Files

| File | Changes |
|------|---------|
| `skills/committee/SKILL.md` | Major rewrite — two-mode flow (standard/interactive), meta agent orchestration, mode selection at start, implementation bridge as headline, session checkpointing at every phase transition |
| `skills/committee/agent-mode.md` | Model tiering (Haiku/Sonnet parameter on Agent tool), tiered report depth by deliverable relevance, failure handling (timeout, quorum, fallback stubs), retry protocol |
| `skills/committee/review-format.md` | New document structure — adds Executive Summary, transparency-tagged Debate Proceedings, tension-prominent two-path Next Steps, member status disclosure for incomplete committees |
| `skills/committee/generation-guide.md` | Domain coverage mapping for blind spot detection (industry × function taxonomy), tier assignment by deliverable relevance, integration with meta agent's composition phase |

### Unchanged

- `skills/committee/collectives/` — all 7 collectives unchanged
- `skills/committee/favorites/` — all 26 personas unchanged
- `skills/committee/history/` — unchanged

---

## Backward Compatibility

| v1 Command | v2 Behavior | Breaking? |
|------------|-------------|-----------|
| `/committee` | Meta agent greets, shows collectives, suggests | No |
| `/committee review [id]` | v2 flow: composition → mode choice → reports → synthesis → next steps → implementation bridge | No — enhanced superset |
| `/committee suggest` | Meta agent analyzes context, recommends collective + tier assignments | No |
| `/committee list` | Shows all collectives and favorites | No change |
| `/committee custom [ids]` | Same + meta agent suggests tier assignments | No |
| `/committee add` | Same behavior | No change |
| `/committee promote` | Same behavior | No change |
| `/committee remove` | Same behavior | No change |
| `--parallel` | Now default. Reports always parallel. Flag kept but is a no-op. | **Behavioral change** — previously opt-in |
| `--quick` / `--deep` | Still works — controls report depth across all tiers | No change |
| `--focus "[topic]"` | Still works — meta agent incorporates into agenda/analysis | No change |
| `--add-members N` | Still works — adds N extra dynamic slots | No change |

### New Flags

| Flag | Purpose |
|------|---------|
| `--interactive` | Select interactive mode (preliminary questions + agenda + debates). Skips the mode selection question. |
| `--standard` | Select standard mode (the default). Skips the mode selection question. Equivalent to v1 with v2 enhancements. |
| `--implement` | Auto-chain into writing-plans → executing-plans after review completes. Shows dry-run preview before handoff. |
| `--sequential` | Opt out of parallel reports. Run sequentially in main context (v1 default behavior). |

### Removed Flags

| Flag | Replacement |
|------|-------------|
| `--express` | No longer needed. Standard mode IS the fast path. `--standard` is the explicit equivalent. |

---

## Risk Assessment

| Risk | Mitigation |
|------|-----------|
| Context window fills with reports + debate | Tiered reports, compressed summaries, SESSION.json as authoritative state (not context memory), lean meta overhead (~1,350 tokens) |
| Debate quality — meta agent voicing may smooth edges (RLHF consensus-seeking) | Steelman instructions, transparency tags (`[from report]` vs `[extended]`), positions grounded in independent reports, user can always request the raw report for any member |
| Sub-agent failure | Quorum enforcement (60% minimum), timeout thresholds (120s), fallback stubs, explicit disclosure in synthesis, retry option |
| Session interrupted mid-review | SESSION.json enables resume from last checkpoint. New session bootstraps from file. |
| User overwhelm from mode choice | Standard is the default and recommended. Interactive is opt-in. Mode can be switched mid-review (interactive → standard). |
| Implementation bridge takes irreversible actions | Dry-run preview before any handoff. Explicit user confirmation. Only approved items passed. |
| Token cost higher than v1 | Sub-agent costs are minimal (~$0.03). Main context is the primary cost and is similar to v1. Cost estimate shown before execution. |
| TOS risk from automation | All phases are user-initiated. Meta agent presents and waits. No autonomous loops. Normal conversational usage. |
| Brand/IP risk from real-company persona anchoring | Personas are analytical lenses, not impersonations. They reference company philosophies and public standards. No trademark claims. Consider adding a disclaimer to output. Flag for legal review before any public promotion. |

---

## Success Criteria

1. A user can run `/committee review tech-product-review` and get a complete, actionable review in standard mode without ever being asked to participate in the process beyond roster confirmation and mode selection
2. Standard mode delivers value (synthesis + next steps) in roughly the same token budget as v1
3. Interactive mode produces measurably richer output — more user context incorporated, more refined recommendations, debate proceedings that surface genuine tensions
4. `--implement` flag successfully chains committee output into writing-plans with no manual translation
5. SESSION.json enables a review to be resumed after interruption with no loss of completed work
6. Sub-agent failures are handled gracefully — user is informed, synthesis discloses missing perspectives, retry is available
7. All v1 commands work without modification
8. No user-facing reference to model names — the interface uses Standard/Interactive/Deep terminology

---

## Appendix: Committee Review Feedback Incorporated

This spec was revised based on a 10-member committee review of the original draft. Key changes:

1. **Inverted default** — Standard (fast) path is now the default. Interactive is opt-in. (9/10 consensus)
2. **Session checkpointing** — SESSION.json written at every phase transition. (7/10 consensus)
3. **Sub-agent failure handling** — Quorum, timeouts, fallback stubs, disclosure. (7/10 consensus)
4. **Implementation bridge promoted** — Headline feature with `--implement` flag, dry-run preview, structured handoff contract. (6/10 consensus)
5. **Cost model made concrete** — Token budgets per scenario replace vague "70% reduction" claim. Cost estimate shown to user. (6/10 consensus)
6. **Debate transparency** — Positions tagged `[from report]` / `[extended]`. Steelman instruction for dissent. (6/10 consensus)
7. **Qualitative consensus language** — "All agree" / "strong majority" / "split" replaces percentage thresholds. (Stripe Engineering)
8. **Domain coverage mapping** — Industry × function taxonomy for blind spot detection. (Meta AI)
9. **Tensions surfaced prominently in Path B** — Not buried under flags. (Shopify Power User)
10. **Mode switching** — User can exit interactive mode mid-debate and switch to standard synthesis. (Netflix Platform)
11. **Legal/IP flag** — Real-company persona anchoring noted for legal review. (OpenAI Product)
