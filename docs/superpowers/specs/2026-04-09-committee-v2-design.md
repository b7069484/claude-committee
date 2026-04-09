# Committee v2 Design Spec — Interactive Executive-Facilitated Reviews

**Date:** 2026-04-09
**Author:** Israel Bitton
**Status:** Draft — awaiting approval

---

## Summary

Committee v2 transforms the review experience from a batch report into an executive-facilitated, interactive session. A meta agent (Executive Assistant) orchestrates the entire flow — composing the committee, managing preliminary research, facilitating topic-based debates, and producing an actionable final document with two-path next steps. The user interacts only with the meta agent; all committee work happens through model-tiered sub-agents.

## Goals

1. **Facilitated experience** — a meta agent manages the review end-to-end, presenting decisions as structured options (numbered lists, Y/N, A/B/C)
2. **Blind spot detection** — proactively identify missing perspectives and offer easy accept/reject additions
3. **Two review modes** — comprehensive (enhanced v1) and interactive (topic-based debate rounds with user checkpoints)
4. **Actionable output** — final document ends with two-path next steps splitting unanimous recommendations from contentious items requiring user sign-off
5. **Token efficiency** — model tiering (Haiku/Sonnet/Opus), tiered report depth, compressed debate context, agenda-as-budget
6. **Implementation bridge** — offer to hand approved recommendations directly to writing-plans → executing-plans
7. **Backward compatibility** — all v1 commands still work, enhanced behavior is additive

## Non-Goals

- Full agent swarm architecture (fights Claude Code's architecture, TOS risk)
- Real-time interjection during generation (not supported by Claude Code's turn model)
- Persistent committee state across sessions (each review is self-contained)
- Changes to the existing 26 favorite personas or 7 collectives

---

## Architecture

### Approach: Hybrid — Parallel Reports, Sequential Debates

- **Meta agent** runs in the main context window (Opus). It is the sole interface the user interacts with.
- **Individual reports** run as parallel sub-agents (Sonnet/Haiku). Truly independent — no bleed-through between members.
- **Debate rounds** run sequentially in the main context. The meta agent voices committee members based on their reports and persona definitions. No sub-agents spawned for debates.
- **Preliminary questions** run as parallel sub-agents (Haiku). Cheap probes to gather context before the expensive report phase.

### Model Tiering

| Model | Used For | Approx Cost Per Member |
|-------|----------|----------------------|
| **Haiku** | Preliminary questions, flag-only scans, blind spot probes | ~$0.001 |
| **Sonnet** | Full individual reports, focused individual reports | ~$0.01 |
| **Opus** | Meta agent (main context only): synthesis, briefing, debate facilitation, next steps | Main context |

The meta agent assigns model tiers during Phase 0 (Composition) based on domain relevance. The user can override any assignment.

### Token Optimization Strategies

1. **Tiered report depth** — full reports (400-500 words) for 3-4 core members, focused reports (200 words) for 3-4 secondary members, flag-only (50 words) for 2-3 specialists. Meta agent assigns tiers and presents to user for confirmation.
2. **Smart member selection per debate round** — only 3-4 relevant members speak per topic, not all 8-10. Reduces debate tokens by ~60%.
3. **Compressed debate context** — meta agent maintains a running summary (~200 words) instead of carrying full conversation history. Each round gets the summary + topic, not the full transcript.
4. **Agenda-as-budget** — high-priority contentious topics get full debate rounds. Low-priority consensus topics get quick confirmation ("All members agree on X. Confirm?").
5. **Model tiering** — Haiku for probes, Sonnet for reports, Opus only in main context. Estimated ~70% cost reduction vs. Opus-everywhere.

---

## Phase Flow

### Phase 0: Intake & Composition

**Context:** Main context, Opus (meta agent)

1. User invokes `/committee review [collective-id]` or `/committee suggest`
2. Meta agent reads the deliverable (files, docs, or description provided by user)
3. Meta agent analyzes: domain, product stage, target audience, complexity
4. Meta agent proposes committee roster with tier assignments:
   - **Full report** (Sonnet) — 3-4 members whose domain directly overlaps the deliverable
   - **Focused report** (Sonnet) — 3-4 members with relevant secondary lens
   - **Flag-only** (Haiku) — 2-3 specialist scanners (accessibility, security, compliance, etc.)
5. Meta agent detects blind spots — perspectives missing from the roster that the deliverable needs
6. Presents blind spot additions as easy accept/reject: numbered list, user types numbers to add or "all" to accept all
7. User confirms roster, swaps members, or adjusts tiers
8. **Roster locked**

**Interaction example:**
```
Meta Agent:
Here's the proposed committee for your AI code review bot:

FULL REPORTS (Sonnet):
  1. Head of Engineering, Stripe — API design, reliability
  2. Head of AI, Meta (FAIR) — model integration, inference
  3. Head of DX, Vercel — developer experience, onboarding
  4. Skeptic End User — will this actually get used?

FOCUSED REPORTS (Sonnet):
  5. Head of Product, OpenAI — competitive positioning
  6. Head of Security Engineering, Google — code scanning risks
  7. VC Partner — market viability, funding angle

FLAG-ONLY (Haiku):
  8. Regulatory & Compliance — data handling flags
  9. Accessibility Advocate — UI accessibility scan

BLIND SPOTS DETECTED:
  A. No enterprise buyer perspective — add Head of IT Procurement, Fortune 500?
  B. No pricing/monetization specialist — add Head of Growth, PLG SaaS?

Add blind spot members? (A, B, all, or none)
```

### Phase 1: Preliminary Questions

**Context:** Parallel sub-agents, Haiku

1. Each committee member is spawned as a Haiku sub-agent
2. Each receives: their persona definition + the deliverable
3. Each proposes 2-3 questions they want answered before their full review
4. Meta agent collects all questions, deduplicates, and prioritizes
5. Presents to user as a numbered list
6. User answers (can skip questions with "N/A")
7. Answers stored for distribution to members in Phase 3

**Purpose:** Members produce dramatically better reports when they have context about the user's constraints, decisions already made, and strategic priorities. This phase costs almost nothing (Haiku) but significantly improves Phase 3 output quality.

### Phase 2: Agenda Preview

**Context:** Main context, Opus (meta agent)

1. Meta agent synthesizes: deliverable analysis + user answers + roster composition
2. Generates a prioritized topic agenda:
   - Key decisions the deliverable faces
   - Critical risks identified
   - Open questions that need resolution
   - Opportunities spotted
3. Assigns priority per topic: **high** (full debate if interactive), **medium** (brief discussion), **low** (quick confirmation)
4. Presents agenda to user — user can reorder, add, remove topics
5. **Agenda locked**
6. Meta agent asks: "Do you want a comprehensive report at the end of the committee's deliberations, or do you want to interact with committee members in real-time through topic-based debate rounds?"
7. User chooses mode → flow forks at Phase 4

### Phase 3: Individual Reports

**Context:** Parallel sub-agents, Sonnet (full + focused) / Haiku (flag-only)

1. All members spawned as parallel sub-agents in a single message (critical for true parallelism)
2. Each agent receives:
   - Their persona definition (from favorites/ or dynamically generated)
   - The full deliverable
   - User's answers to preliminary questions
   - The agenda (so they can structure their analysis around it)
3. Report depth by tier:
   - **Full report** (Sonnet, 400-500 words): executive assessment, key issues with rationale, pros/cons, suggestions with benchmarks, "what I'd ship instead"
   - **Focused report** (Sonnet, 200 words): executive assessment, top 2 issues through their specific lens, key suggestion
   - **Flag-only** (Haiku, 50 words): scan for their specialty, report only if they find something ("No accessibility concerns" or "FLAG: color contrast fails WCAG AA on the pricing page mockup")
4. Reports collected back to main context
5. Meta agent reads all reports (full text enters main context)

### Phase 4: Executive Briefing

**Context:** Main context, Opus (meta agent)

1. Meta agent synthesizes all individual reports into an executive briefing:
   - **Consensus areas** — issues where 60%+ of members converge, with agreement counts
   - **Key tensions** — genuine disagreements, identifying who disagrees and why
   - **Blind spots detected** — perspectives missing even after the review (meta agent may suggest adding members for a follow-up review)
   - **Recommended debate topics** — ranked by impact and contentiousness
2. Presents briefing to user
3. If blind spots are detected, offers to add members for a follow-up mini-review (spawns additional Sonnet/Haiku sub-agents for just the new members, then updates the briefing)

**Mode fork:**
- **Comprehensive mode** → skip to Phase 6 (Final Document). The briefing IS the synthesis. If the user added blind spot members in step 3, their reports are incorporated.
- **Interactive mode** → proceed to Phase 5 (Debate Rounds). The briefing frames the debate agenda. New members participate in debates.

### Phase 5: Interactive Debate Rounds (Interactive Mode Only)

**Context:** Main context, Opus (meta agent voices members)

**Structure:** Topic-based rounds. Each round focuses on one agenda topic.

1. Meta agent introduces the topic and its priority
2. Selects 3-4 relevant members to speak (based on their reports and expertise)
3. Meta agent voices each selected member — faithfully representing their position from their report, extending it with the persona's analytical lens applied to the specific debate topic
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
```

**User can:**
- Add context or constraints the committee didn't know
- Challenge a specific position
- Confirm and move to the next topic
- Ask the meta agent to add a topic to the agenda
- Ask to hear more from a specific member

**Between rounds:** Meta agent updates the compressed running summary (~200 words). This summary — not the full transcript — carries forward into subsequent rounds, keeping context window growth linear, not exponential.

**Consensus fast-track:** If a topic has 80%+ agreement from reports, the meta agent presents it as a quick confirmation instead of a full round:
```
Topic: API-First Architecture
All 4 relevant members agree this is the correct approach. No dissent.
Confirm and move on? (Y/n)
```

### Phase 6: Final Document & Next Steps

**Context:** Main context, Opus (meta agent)

The meta agent assembles the complete review document and writes it to `./committee-review-YYYY-MM-DD-[topic].md`.

**Document structure:**

```markdown
# Committee Review: [Deliverable Name]

**Committee:** [collective name] · **Members:** [N] · **Date:** [YYYY-MM-DD]
**Mode:** [Comprehensive / Interactive] · **Report depth:** [default / quick / deep]

---

## 1. Executive Summary
Meta agent's high-level synthesis of the review (200 words).
Overall verdict, biggest opportunities, biggest risks.

## 2. Committee Roster
Table: member name, company, archetype, tier (full/focused/flag), model used.

## 3. Individual Reports

### 3.1 Full Reports
[Each full-report member's complete analysis]

### 3.2 Focused Reports
[Each focused-report member's analysis]

### 3.3 Flags
[Flag-only members' findings, if any]

## 4. Debate Proceedings (Interactive Mode Only)

### Round 1: [Topic]
Participants: [members]
Positions: [summary]
User input: [what user contributed]
Resolution: [outcome]

### Round 2: [Topic]
...

## 5. Synthesis

### Consensus Points
[Issues where 60%+ converged, with agreement counts and evidence]

### Key Tensions
[Genuine disagreements with both sides' positions and reasoning]

### Evidence & Benchmarks
[Real-world data, case studies, and benchmarks cited across reports]

## 6. Blind Spots & Recommended Additions
[What the committee couldn't evaluate]
[Specific roles/perspectives to add for follow-up review]

## 7. Next Steps

### Path A: Full Adoption
All recommendations implemented. Ordered by priority with dependencies mapped.

1. [Action item] — Owner: [suggested] — Depends on: [none / item N]
   Consensus: [unanimous / majority] — Risk: [low / medium / high]
2. ...

### Path B: Selective Adoption

**Section 1 — Ready to Act (Unanimous):**
1. [Action item] — all members agree, no review needed
2. ...

**Section 2 — Requires Your Decision (Contentious):**

**Item: [Topic]**
- Position A: [who + reasoning]
- Position B: [who + reasoning]
- Meta agent assessment: [analysis of trade-offs]
- If you choose A: [implications]
- If you choose B: [implications]
- Your decision: ___

**Item: [Topic]**
...
```

### Phase 7: Implementation Bridge

**Context:** Main context, Opus (meta agent)

After the final document is saved, the meta agent offers:

```
Review complete and saved to ./committee-review-2026-04-09-ai-code-review-bot.md

What would you like to do?
1. Create an implementation plan from your approved items
2. Save the report and stop here
3. Revisit specific recommendations before planning
```

**If user chooses 1:**
1. Meta agent compiles the approved items:
   - If Path A: all recommendations
   - If Path B: Section 1 items + any Section 2 items the user signed off on
2. Invokes `superpowers:writing-plans` with the approved items as the spec input
3. After plan is written and approved by user, offers: "Plan approved. Ready to begin implementation?" → invokes `superpowers:executing-plans`

**Handoff rules:**
- Only items explicitly approved by the user are passed to the plan
- Priority ordering from the committee synthesis is preserved
- Dependencies between items are included in the handoff
- The plan references the committee review document for context

---

## The Meta Agent

### Persona Definition

**Role:** Chief of Staff / Executive Facilitator
**Tone:** Crisp, structured, decisive — like a senior McKinsey engagement manager
**Not:** A chatbot. Not deferential. Not verbose.

### Core Behaviors

1. **Summarizes, never dumps** — raw data stays in sub-agents. The user sees synthesized insights.
2. **Presents decisions as structured options** — numbered lists, A/B/C, Y/N. Never open-ended "what do you think?"
3. **Proactively flags what needs attention** — doesn't wait for the user to ask. If something is contentious, surfaces it.
4. **Manages the user's time** — knows when to compress ("All agree, confirm?") and when to expand ("This is a real disagreement, here's why it matters").
5. **Tracks running state** — maintains awareness of what's been decided, what's open, what's changed throughout the session.
6. **Honest about limitations** — when the committee can't evaluate something, says so and suggests who could.

### Context It Maintains in Main Window

- Committee roster with tier assignments
- User's answers to preliminary questions (compressed)
- Agenda with priority/budget per topic
- Compressed report summaries (~50 words each, plus full reports available for reference)
- Running debate summary (~200 words, updated between rounds)
- Decision log: what user confirmed, what's still open

### What It Delegates to Sub-Agents

- Preliminary questions → Haiku sub-agents (parallel)
- Individual reports → Sonnet/Haiku sub-agents (parallel)
- Flag-only scans → Haiku sub-agents (parallel)

### What It Handles In Main Context

- Committee composition and blind spot detection
- Agenda generation and prioritization
- Executive briefing and synthesis
- Debate facilitation (voicing members from their reports + personas)
- User checkpoints and interaction
- Final document assembly
- Next steps generation
- Implementation bridge handoff

---

## File Structure

### New Files

| File | Purpose |
|------|---------|
| `skills/committee/meta-agent.md` | Executive assistant persona definition, behaviors, interaction patterns, context management rules |
| `skills/committee/debate-format.md` | Topic-based round structure, checkpoint format, compressed summary rules, consensus fast-track criteria |
| `skills/committee/next-steps-format.md` | Two-path action plan template, consensus vs. contentious classification rules, handoff format for writing-plans |

### Modified Files

| File | Changes |
|------|---------|
| `skills/committee/SKILL.md` | Major rewrite — 8-phase flow (Phases 0-7), meta agent orchestration, mode fork, implementation bridge. Replaces current 7-phase flow. |
| `skills/committee/agent-mode.md` | Add model tiering (Haiku/Sonnet parameter on Agent tool), preliminary questions phase, tiered report depth instructions |
| `skills/committee/review-format.md` | New final document structure — adds Executive Summary, Debate Proceedings section, two-path Next Steps. Individual report format stays the same. |
| `skills/committee/generation-guide.md` | Add blind spot detection logic, tier assignment rules for dynamic members, integration with meta agent's composition phase |

### Unchanged

- `skills/committee/collectives/` — all 7 collectives unchanged
- `skills/committee/favorites/` — all 26 personas unchanged
- `skills/committee/history/` — unchanged

---

## Backward Compatibility

| v1 Command | v2 Behavior | Breaking? |
|------------|-------------|-----------|
| `/committee` | Meta agent greets, shows collectives, suggests | No |
| `/committee review [id]` | Full v2 flow: composition → questions → reports → briefing → mode choice | No — enhanced, superset of v1 |
| `/committee suggest` | Meta agent analyzes context, recommends collective + tier assignments | No |
| `/committee list` | Shows all collectives and favorites | No change |
| `/committee custom [ids]` | Same + meta agent suggests tier assignments | No |
| `/committee add` | Same behavior | No change |
| `/committee promote` | Same behavior | No change |
| `/committee remove` | Same behavior | No change |
| `--parallel` | Now default. Reports always parallel. Flag kept but is a no-op. | **Behavioral change** — previously opt-in, now default |
| `--quick` / `--deep` | Still works — controls report depth per tier | No change |
| `--focus "[topic]"` | Still works — meta agent incorporates into agenda | No change |
| `--add-members N` | Still works — adds N extra dynamic slots | No change |

### New Flags

| Flag | Purpose |
|------|---------|
| `--express` | Skip preliminary questions (Phase 1), agenda preview (Phase 2), and debate (Phase 5). Composition → reports → briefing/synthesis → next steps. Fastest path for users who want v1-style output with v2 enhancements. |
| `--sequential` | Opt out of parallel reports. Run reports sequentially in main context (v1 default behavior). For users who prefer lower cost over independence. |

---

## Risk Assessment

| Risk | Mitigation |
|------|-----------|
| Context window fills with 8-10 full reports + debate | Tiered reports (not all full-length), compressed debate summaries, agenda-as-budget |
| Debate quality when meta agent voices members (not true independence) | Reports ARE independent (parallel sub-agents). Debates are refinement, not generation. The independence already happened. |
| User overwhelm from multi-phase flow | Express flag for fast path. Meta agent compresses consensus. Quick confirmations for settled topics. |
| Token cost higher than v1 | Model tiering offsets: Haiku probes + Sonnet reports vs. everything in Opus main context. Net cost similar or lower for most reviews. |
| TOS risk from automation patterns | All phases are user-initiated. Meta agent presents and waits. No autonomous loops. Normal conversational Claude Code usage. |

---

## Success Criteria

1. A user can run `/committee review tech-product-review` and experience the full v2 flow without confusion
2. Interactive mode produces measurably richer output than comprehensive mode (more user context incorporated, more refined recommendations)
3. Express mode (`--express`) produces output at least as good as v1 in fewer turns
4. Final document's Next Steps section is directly usable as input to writing-plans
5. Token cost for a default 10-member review is within 1.5x of v1's cost despite added phases
6. All v1 commands work without modification
