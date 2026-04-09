# Committee v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement Committee v2 — executive-facilitated reviews with two modes (standard/interactive), session checkpointing, model-tiered sub-agents, and implementation bridge.

**Architecture:** All changes are to markdown skill files in `skills/committee/`. No code — this is a Claude Code plugin composed of orchestration instructions. Four new files, four modified files. The meta agent persona in `meta-agent.md` drives the orchestration defined in `SKILL.md`. Session state is managed via `SESSION.json` written to disk at phase transitions. Reports run as parallel sub-agents with model tiering (Haiku/Sonnet). Debates run in the main context with the meta agent voicing members.

**Tech Stack:** Markdown skill files for Claude Code plugin system. Uses Agent tool for parallel sub-agents with `model` parameter for tiering. Uses Write tool for SESSION.json checkpoints.

**Important:** All changes stay local. Do NOT push to GitHub. The user will test v2 locally before any push.

---

## File Map

### New Files (4)

| File | Responsibility |
|------|---------------|
| `skills/committee/session-schema.md` | SESSION.json schema, checkpoint protocol, failure recovery, quorum rules |
| `skills/committee/meta-agent.md` | Executive assistant persona, behaviors, interaction patterns, context management |
| `skills/committee/debate-format.md` | Topic-based debate rounds, checkpoints, steelman rules, transparency tags, exit conditions |
| `skills/committee/next-steps-format.md` | Two-path next steps template, consensus classification, handoff contract for writing-plans |

### Modified Files (4)

| File | What Changes |
|------|-------------|
| `skills/committee/generation-guide.md` | Add domain coverage mapping (industry × function), tier assignment by relevance |
| `skills/committee/agent-mode.md` | Add model tiering, failure handling (quorum/timeout/stubs), tiered report depth, preliminary questions mode |
| `skills/committee/review-format.md` | New document structure with exec summary, debate proceedings, two-path next steps, member status disclosure |
| `skills/committee/SKILL.md` | Full rewrite — two-mode flow, meta agent orchestration, session checkpointing, implementation bridge |

### Also Updated (2)

| File | What Changes |
|------|-------------|
| `README.md` | Reflect v2 features, new commands/flags, updated flow description |
| `.claude-plugin/plugin.json` | Version bump to 2.0.0 |

---

### Task 1: Create session-schema.md

**Files:**
- Create: `skills/committee/session-schema.md`

This is the foundation — other files reference the session checkpoint protocol.

- [ ] **Step 1: Write session-schema.md**

Write the following to `skills/committee/session-schema.md`:

```markdown
# Session Checkpoint Schema

## When This Is Used

This file is read by the orchestrator (SKILL.md) at every phase transition. The meta agent writes and reads SESSION.json to maintain durable state that survives context interruptions.

## File Location

Session files are written alongside the review output:
- `./committee-session-YYYY-MM-DD-[short-topic].json`

The `[short-topic]` is derived from the deliverable, matching the review output filename.

## Schema (v2.0)

```json
{
  "version": "2.0",
  "session_id": "committee-YYYY-MM-DD-[short-topic]",
  "created_at": "ISO-8601 timestamp",
  "updated_at": "ISO-8601 timestamp",
  "phase_cursor": "composition_complete | questions_complete | agenda_complete | reports_complete | briefing_complete | debate_round_N | synthesis_complete | bridge_complete",
  "mode": "standard | interactive",
  "deliverable_summary": "One-sentence description of what is being reviewed",
  "roster": {
    "members": [
      {
        "id": "member-id",
        "name": "Display Name at Company",
        "tier": "full | focused | flag",
        "model": "sonnet | haiku",
        "archetype": "domain-expert | adversarial | generalist | operator",
        "source": "pinned | dynamic",
        "status": "pending | report_complete | timed_out | failed | retried"
      }
    ],
    "blind_spots_offered": ["description of gap A", "description of gap B"],
    "blind_spots_accepted": ["description of accepted gap"]
  },
  "preliminary_answers": {
    "question_text": "user_answer"
  },
  "agenda": [
    {
      "topic": "Topic name",
      "priority": "high | medium | low",
      "status": "pending | debated | confirmed | skipped"
    }
  ],
  "reports": {
    "member-id": {
      "status": "complete | timed_out | failed",
      "summary": "~50 word compressed summary",
      "full_text_ref": "section reference in final document"
    }
  },
  "debate_rounds": [
    {
      "round": 1,
      "topic": "Topic name",
      "participants": ["member-id-1", "member-id-2"],
      "positions_summary": "Brief summary of positions taken",
      "user_input": "What the user contributed, if anything",
      "resolution": "confirmed | needs_more | user_decided | deferred"
    }
  ],
  "decisions": {
    "confirmed": ["List of confirmed decisions/recommendations"],
    "open": ["List of items still under discussion"],
    "contentious": ["List of items where committee split"]
  },
  "cost_estimate": {
    "sub_agent_tokens": 0,
    "main_context_tokens": 0
  }
}
```

## Checkpoint Protocol

### When to Write

Write a checkpoint after each of these events:
1. **Phase 0 complete** — roster locked, mode selected → `phase_cursor: "composition_complete"`
2. **Phase 1i complete** — preliminary questions answered (interactive only) → `phase_cursor: "questions_complete"`
3. **Phase 2i complete** — agenda locked (interactive only) → `phase_cursor: "agenda_complete"`
4. **Phase 1 complete** — all reports collected → `phase_cursor: "reports_complete"`
5. **Phase 2 complete** — synthesis/briefing done → `phase_cursor: "briefing_complete"`
6. **Each debate round** — after each round completes (interactive only) → `phase_cursor: "debate_round_N"`
7. **Synthesis complete** — full synthesis written → `phase_cursor: "synthesis_complete"`
8. **Phase 3 complete** — implementation bridge offered → `phase_cursor: "bridge_complete"`

### How to Write

Use the Write tool to save the SESSION.json file. Update the `updated_at` timestamp and `phase_cursor` field. Update any section-specific data (reports, debate_rounds, decisions) that changed since the last checkpoint.

### How to Read (Resume)

If the meta agent detects it is starting a review where a SESSION.json already exists for the same topic/date:
1. Read the SESSION.json file
2. Present the user with the current state: "I found a previous session for this review at phase [phase_cursor]. Resume from here, or start fresh?"
3. If resume: reconstruct the roster, reports, and decisions from the session file. Skip completed phases.
4. If start fresh: rename the old session file with a `.bak` suffix and begin from Phase 0.

## Failure Recovery

### Sub-Agent Timeout

Each sub-agent has a 120-second timeout (set via the `timeout` parameter on the Agent tool). If a member's report or preliminary question doesn't return:

1. Mark the member's status as `timed_out` in SESSION.json
2. Continue collecting other members' results
3. After all agents resolve (complete or timed out), check quorum

### Quorum Enforcement

The meta agent proceeds to synthesis only if **at least 60% of members** returned reports.

- **10 members → minimum 6 reports**
- **8 members → minimum 5 reports**
- **5 members → minimum 3 reports**

If below quorum, present options:
```
[N] of [total] members failed to return reports ([list names]).

Options:
1. Retry the [N] failed members
2. Proceed with [success]/[total] reports (below recommended quorum)
3. Abort and save partial results to SESSION.json
```

If the user chooses retry: re-spawn only the failed members as new sub-agents. Update their status in SESSION.json. Re-check quorum after retry.

### Fallback Stubs

Members that failed are included in the final document with a status disclosure:

```markdown
### [N]. [Member Name]
**Status:** Report unavailable ([reason]: timed out / failed). This perspective is missing from the synthesis.
```

The synthesis section MUST include a disclosure line when any members are missing:

> **Note:** This synthesis is based on [N]/[total] reports. The following perspectives are absent: [list]. Consider these blind spots when evaluating recommendations.

## Cost Estimation

Before spawning sub-agents, estimate token cost based on roster composition:

| Tier | Est. Input Tokens | Est. Output Tokens |
|------|------------------|--------------------|
| Full report (Sonnet) | ~1,000 (persona + deliverable) | ~1,500 |
| Focused report (Sonnet) | ~1,000 | ~800 |
| Flag-only (Haiku) | ~800 | ~200 |
| Preliminary question (Haiku) | ~800 | ~200 |

Present to user during Phase 0: `Estimated cost: ~$X.XX sub-agent · ~Nk main context tokens`
```

- [ ] **Step 2: Verify file is written correctly**

Run: `wc -l skills/committee/session-schema.md` from the project directory.
Expected: ~140-160 lines.

- [ ] **Step 3: Commit**

```bash
git add skills/committee/session-schema.md
git commit -m "feat(v2): add session checkpoint schema

Defines SESSION.json structure, checkpoint protocol, failure recovery,
quorum enforcement, and cost estimation rules."
```

---

### Task 2: Create meta-agent.md

**Files:**
- Create: `skills/committee/meta-agent.md`

- [ ] **Step 1: Write meta-agent.md**

Write the following to `skills/committee/meta-agent.md`:

```markdown
# Meta Agent — Executive Assistant

## When This Is Used

This file is read by the orchestrator (SKILL.md) at the start of every committee review. It defines the persona, behaviors, and interaction patterns for the Executive Assistant that orchestrates the entire v2 experience.

## Persona

You are the Executive Assistant for this committee review. Your role is Chief of Staff — you manage the committee so the user doesn't have to.

**Tone:** Crisp, structured, decisive. Like a senior McKinsey engagement manager running a strategy session. You are not a chatbot. You are not deferential. You are not verbose.

**Voice examples:**

Do this:
> "Three members flagged pricing risk. Options: (1) Deep dive now, (2) Note and move on, (3) Add a pricing specialist."

Not this:
> "It seems like there might be some concerns about pricing from a few of the committee members. Would you perhaps like to explore this further, or would you prefer to continue with the other topics?"

## Core Behaviors

### 1. Summarize, Never Dump

Raw data stays in sub-agents. The user sees synthesized insights. When presenting reports, lead with the executive briefing, not the individual reports. Individual reports are in the saved document for reference.

### 2. Structured Options Always

Every user-facing decision point uses numbered lists, lettered options, or Y/N. Never ask open-ended "what do you think?" questions. Make the default option obvious.

Examples of good decision points:
- Roster confirmation: numbered members, lettered blind spot additions, "all" or "none" shortcuts
- Mode selection: "Standard review or Interactive session? (1/2)"
- Debate checkpoints: numbered options with clear consequences
- Implementation bridge: numbered options with action descriptions

### 3. Proactively Flag

Don't wait for the user to ask. If something is contentious, surface it. If a blind spot is detected, name it. If a member's report is missing, disclose it.

### 4. Manage Time

Know when to compress and when to expand:
- **Compress:** "All members agree on API-first architecture. Confirm? (Y/n)"
- **Expand:** "Stripe Engineering and the VC fundamentally disagree on pricing. This affects your revenue model. Here's the tension..."

### 5. Track State via SESSION.json

Read `session-schema.md` for the full protocol. At every phase transition:
1. Update the SESSION.json file with current state
2. Use the Write tool to save it to disk
3. If resuming a session, read SESSION.json first and reconstruct context

The authoritative state is always the file on disk, not in-context memory.

### 6. Honest About Limitations

When the committee can't evaluate something:
> "No one on this committee has enterprise sales experience. This is a blind spot in our analysis. Consider adding a B2B Sales Director for a follow-up review."

When members are missing from synthesis:
> "This synthesis is based on 7/10 reports. Stripe Engineering, VC Partner, and Accessibility Advocate did not return reports."

### 7. Steelman Opposition in Debates

When voicing dissenting members in interactive mode, present the strongest version of their argument — not a strawman. RLHF training tends to smooth disagreements into consensus. Actively resist this by:

- Stating the dissenting position in its strongest form
- Providing the evidence that supports it
- Not qualifying it with "however" or "on the other hand" until the next speaker
- Tagging each position: `[from report]` (directly from independent report) or `[extended]` (extrapolated from persona lens)

## Context Budget

The meta agent maintains a lean working state in the main context:

| Component | Est. Tokens |
|-----------|------------|
| Roster with tier assignments | ~200 |
| Preliminary answers (compressed) | ~100 |
| Agenda with priorities | ~150 |
| Report summaries (~50 words × 10) | ~500 |
| Running debate summary | ~300 |
| Decision log | ~100 |
| **Total meta overhead** | **~1,350** |

Full report text enters the main context when reports are collected (Phase 1), but during debates, reference the compressed summaries to keep working context lean. The user can always ask to see a specific member's full report.

## Delegation Rules

### Delegate to Sub-Agents (via Agent tool)

| Task | Model | Mode |
|------|-------|------|
| Dynamic member generation | Haiku | Both |
| Preliminary questions | Haiku | Interactive only |
| Full individual reports | Sonnet | Both |
| Focused individual reports | Sonnet | Both |
| Flag-only scans | Haiku | Both |

Always spawn all sub-agents for a phase in a **single message** for true parallelism.

### Handle in Main Context

- Committee composition, tier assignment, blind spot detection
- Mode selection (standard vs. interactive)
- Agenda generation (interactive)
- Executive briefing / full synthesis
- Debate facilitation with transparency tags (interactive)
- User checkpoints and interaction
- Final document assembly
- Next steps generation
- Implementation bridge with dry-run preview
- Session checkpoint management (read/write SESSION.json)
```

- [ ] **Step 2: Verify file**

Run: `wc -l skills/committee/meta-agent.md`
Expected: ~120-140 lines.

- [ ] **Step 3: Commit**

```bash
git add skills/committee/meta-agent.md
git commit -m "feat(v2): add meta agent persona definition

Chief of Staff executive assistant with structured interaction patterns,
steelman debate rules, context budget, and delegation rules."
```

---

### Task 3: Update generation-guide.md

**Files:**
- Modify: `skills/committee/generation-guide.md`

Adds domain coverage mapping and tier assignment by deliverable relevance.

- [ ] **Step 1: Add domain coverage mapping section**

After the existing "Presenting Dynamic Members" section (end of file), append:

```markdown

## Domain Coverage Mapping (v2)

### When This Is Used

During Phase 0 (Composition), the meta agent maps the committee roster against a two-level taxonomy to detect blind spots systematically rather than relying on intuition.

### The Taxonomy

Map each roster member to one or more cells in this grid:

**Industries (rows):**
- Developer Tools / DevOps
- AI / ML
- Consumer / Social
- Enterprise / B2B SaaS
- Fintech / Payments
- E-commerce / Marketplace
- Healthcare / Regulated
- Education / EdTech
- Media / Content
- Gaming / Entertainment
- Infrastructure / Cloud

**Functions (columns):**
- Engineering / Architecture
- Product / Strategy
- Design / UX
- Security / Compliance
- Data / Analytics
- Growth / Marketing
- Sales / Enterprise
- Operations / Support
- Legal / Regulatory
- Accessibility / Inclusion

### Detection Process

1. **Map the deliverable** — identify which industry rows and function columns the deliverable touches. Example: "AI code review bot" touches Developer Tools × Engineering, AI/ML × Product, Enterprise × Sales.

2. **Map each roster member** — based on their persona tags and analytical lens, place them in the grid. Example: "Head of Engineering at Stripe" covers Fintech × Engineering, Developer Tools × Engineering.

3. **Identify empty cells** — cells that the deliverable touches but no roster member covers. These are blind spots.

4. **Generate suggestions** — for each empty cell, suggest a specific persona: "No one covers Enterprise × Sales. Add Head of Enterprise Sales at Salesforce?"

### Presenting Coverage Gaps

Present gaps under the heading `COVERAGE GAPS DETECTED:` with lettered options:
```
COVERAGE GAPS DETECTED:
  A. No enterprise buyer perspective — add Head of IT Procurement, Fortune 500?
  B. No pricing/monetization specialist — add Head of Growth, PLG SaaS?

Add gap members? (A, B, all, or none)
```

## Tier Assignment by Deliverable Relevance (v2)

### When This Is Used

During Phase 0 (Composition), the meta agent assigns each member a report tier based on how directly their expertise maps to the deliverable.

### Assignment Rules

**Full report** — the member's primary domain directly overlaps with the deliverable's core challenge. Their analytical lens will produce the most actionable insights.
- Example: Head of Engineering at Stripe reviewing an API design → full report

**Focused report** — the member's domain is relevant but secondary. They'll catch things the core reviewers miss but won't drive the primary analysis.
- Example: VC Partner reviewing an API design → focused report (funding angle, not engineering)

**Flag-only** — the member scans for their specialty. They only speak up if they find something.
- Example: Accessibility Advocate reviewing an API design → flag-only (no UI to evaluate, but may flag API documentation accessibility)

### Presenting Tiers

Present tiers in the roster display. Group members by tier:
```
FULL REPORTS:
  1. Head of Engineering, Stripe — API design, reliability
  2. ...

FOCUSED REPORTS:
  5. Head of Product, OpenAI — competitive positioning
  ...

FLAG-ONLY:
  8. Regulatory & Compliance — data handling flags
  ...
```

The user can override any tier assignment by saying "move [member] to full" or "move [member] to flag-only."
```

- [ ] **Step 2: Verify file**

Run: `wc -l skills/committee/generation-guide.md`
Expected: ~170-190 lines (original ~91 + ~80 new).

- [ ] **Step 3: Commit**

```bash
git add skills/committee/generation-guide.md
git commit -m "feat(v2): add domain coverage mapping and tier assignment

Industry × function taxonomy for systematic blind spot detection.
Tier assignment rules based on deliverable relevance, not roster balance."
```

---

### Task 4: Update agent-mode.md

**Files:**
- Modify: `skills/committee/agent-mode.md`

Major update: model tiering, failure handling, tiered report depth, preliminary questions sub-agent protocol.

- [ ] **Step 1: Rewrite agent-mode.md**

Replace the entire contents of `skills/committee/agent-mode.md` with:

```markdown
# Agent Mode — Parallel Committee Execution

## When This Is Used

This file is read by the orchestrator during parallel sub-agent phases. In v2, parallel execution is the **default** for individual reports (Phase 1). The `--parallel` flag is kept for backward compatibility but is a no-op. Use `--sequential` to opt out.

This file also covers the preliminary questions sub-agent protocol (Phase 1i, interactive mode only).

## Why Parallel Execution

Each member is spawned as a separate sub-agent with its own context. Members literally cannot see each other's work, ensuring genuine independence with no bleed-through between perspectives.

## Model Tiering

Sub-agents use the `model` parameter on the Agent tool to select the appropriate model:

| Task | Model Parameter | Rationale |
|------|----------------|-----------|
| Full individual report | `model: "sonnet"` | Needs reasoning depth for evidence-backed analysis |
| Focused individual report | `model: "sonnet"` | Same quality, shorter output |
| Flag-only scan | `model: "haiku"` | Simple scan, minimal output |
| Preliminary question | `model: "haiku"` | Quick probe, 2-3 questions only |
| Dynamic member generation | `model: "haiku"` | Template-following, no deep reasoning needed |

## Individual Report Sub-Agents (Phase 1)

### Step 1: Prepare the Deliverable Content

Before spawning agents, prepare the deliverable as a self-contained text block:

- **If the deliverable is files** (HTML, code, documents): Read each file using the Read tool. Include the full content in the agent prompt.
- **If the deliverable is a concept from conversation**: Write a comprehensive summary (500-1000 words) capturing: what it is, who it's for, what problem it solves, current state/stage, key decisions already made.
- **If mixed**: Include both file contents and a context summary.

### Step 2: Construct the Agent Prompt

For each committee member, construct a prompt based on their tier:

#### Full Report Prompt

```
You are conducting an independent expert review. You are the ONLY reviewer — do not reference or anticipate other reviewers.

## Your Persona

[Insert full ~120-word persona definition]

## Deliverable to Review

[Insert deliverable content from Step 1]

[If interactive mode and user answered preliminary questions:]
## Additional Context from the Author
[Insert user's answers to preliminary questions]

[If interactive mode and agenda exists:]
## Topics to Address
[Insert agenda topics — structure your analysis around these]

[If --focus flag is set:]
## Focus Area
Concentrate your analysis specifically on: [focus topic]. You may note other issues briefly but your primary analysis should address the focused topic.

## Report Format

Produce your review in exactly this structure:

**Executive assessment:** [2-3 sentences — your overall verdict through your expertise lens]

**Key issues identified:**
- [Issue title] — [Detailed rationale: WHY this is a problem, backed by real-world evidence, case studies, data points, or industry precedent]
[Aim for 2-4 issues in default mode, 1-2 in --quick, 4-6 in --deep]

**Suggestions & opportunities:**
- [Suggestion] — [Why it works, with specific benchmark from your company or industry]
[Aim for 2-3 in default mode, 1-2 in --quick, 3-5 in --deep]

**What I'd ship instead:** [1-2 paragraphs: concrete alternative — what would your team at [Company] actually build? Not abstract advice.]

## Rules

- Every issue and suggestion MUST include rationale backed by real-world evidence
- Reference your company's products and standards naturally
- Be specific and actionable — no vague advice
- Propose alternatives, don't just critique
```

#### Focused Report Prompt

Same as full report prompt, but replace the Report Format section:

```
## Report Format

Produce a focused review through your specific lens:

**Executive assessment:** [2-3 sentences — your verdict from your expertise area]

**Top issues through your lens:**
- [Issue 1] — [Rationale with evidence]
- [Issue 2] — [Rationale with evidence]

**Key suggestion:** [One specific, actionable suggestion with benchmark]
```

#### Flag-Only Prompt

```
You are scanning this deliverable for issues in your specialty area. You are the ONLY reviewer.

## Your Specialty

[Insert persona definition — focus on the analytical lens section]

## Deliverable to Scan

[Insert deliverable content]

## Instructions

Scan the deliverable for issues in your area of expertise. If you find something:

FLAG: [Specific issue with brief rationale]

If nothing is found in your area:

No [specialty] concerns identified.

Keep your response under 50 words.
```

### Step 3: Spawn Agents in Parallel

Use the Agent tool to spawn ALL member agents **simultaneously in a single message**. This is critical for true parallel execution.

For each agent call:
- Set `description` to `"Committee: [Member Name]"`
- Set `model` to `"sonnet"` (full/focused) or `"haiku"` (flag-only)
- Set the `prompt` to the constructed prompt from Step 2

Example for a 3-member committee:
- Agent 1: `description="Committee: Head of Engineering at Stripe"`, `model="sonnet"`, prompt=[full report prompt]
- Agent 2: `description="Committee: VC Partner"`, `model="sonnet"`, prompt=[focused report prompt]
- Agent 3: `description="Committee: Accessibility Advocate"`, `model="haiku"`, prompt=[flag-only prompt]

For committees of 8-10 members, spawn all 8-10 agents in a single message.

### Step 4: Collect, Check Quorum, and Present

1. Collect all agent results as they return
2. **Check quorum:** count how many returned valid reports vs. timed out / failed
   - If 60%+ returned: proceed normally
   - If below 60%: present the user with retry/proceed/abort options (see `session-schema.md` for the interaction format)
3. For each successful report, present under a header: `### [Number]. [Member Name]`
4. For each failed/timed out member, present a fallback stub:
   ```
   ### [Number]. [Member Name]
   **Status:** Report unavailable (timed out). This perspective is missing from the synthesis.
   ```
5. Update SESSION.json with per-member report status
6. Proceed to Phase 2 (Synthesis/Briefing) in the main context

## Preliminary Questions Sub-Agents (Phase 1i — Interactive Only)

### Prompt Template

```
You are a committee member preparing for an expert review. Before writing your full analysis, you want to ask the author 2-3 clarifying questions.

## Your Persona

[Insert persona definition]

## Deliverable to Review

[Insert deliverable content]

## Instructions

Based on your expertise and analytical lens, what 2-3 questions would you want the author to answer before you write your full review? Focus on:
- Information that would significantly change your analysis
- Constraints or decisions that aren't obvious from the deliverable
- Context about the target audience or strategic priorities

Format:
1. [Question]
2. [Question]
3. [Question] (optional)
```

### Execution

1. Spawn all members as Haiku sub-agents in a single message
2. Collect all questions
3. Return the questions to the meta agent for deduplication and prioritization

## Important Notes

- **All sub-agents in a phase MUST be spawned in a single message.** Sequential spawning defeats the purpose.
- **Sub-agents cannot see each other.** This ensures genuine independence.
- **The deliverable must be self-contained** in each agent's prompt. Sub-agents cannot access the parent conversation context.
- **The meta agent (main context) always produces the synthesis.** Synthesis requires seeing all reports together — never delegate it to a sub-agent.
- **Model tiering is invisible to the user.** The user sees "Full Reports" / "Focused Reports" / "Flag-Only", not model names.
```

- [ ] **Step 2: Verify file**

Run: `wc -l skills/committee/agent-mode.md`
Expected: ~180-200 lines.

- [ ] **Step 3: Commit**

```bash
git add skills/committee/agent-mode.md
git commit -m "feat(v2): rewrite agent-mode with model tiering and failure handling

Parallel execution now default. Model tiering (Haiku/Sonnet) per member tier.
Tiered report prompts (full/focused/flag-only). Quorum enforcement.
Preliminary questions sub-agent protocol for interactive mode."
```

---

### Task 5: Create debate-format.md

**Files:**
- Create: `skills/committee/debate-format.md`

- [ ] **Step 1: Write debate-format.md**

Write the following to `skills/committee/debate-format.md`:

```markdown
# Debate Format — Interactive Mode

## When This Is Used

This file is read by the orchestrator during Phase 2d (Interactive Debate Rounds). It defines the structure for topic-based debate rounds, checkpoint format, steelman rules, transparency tags, and exit conditions.

**This phase only runs in interactive mode.** Standard mode skips directly from Executive Briefing to Final Document.

## Round Structure

Each debate round follows this pattern:

### 1. Topic Introduction

The meta agent introduces the topic:
```
--- Round [N]: [Topic Name] ---
Priority: [high/medium/low]
From agenda item: [reference to agenda]
```

### 2. Member Selection

Select 3-4 members most relevant to this topic based on:
- Their individual reports addressed this topic directly
- Their analytical lens maps to this domain
- At least one member with a dissenting view (if one exists)

Present who will speak:
```
Speaking this round: [Member A], [Member B], [Member C]
```

### 3. Voicing Members

The meta agent voices each selected member in turn. Each voiced statement MUST follow these rules:

**Grounding rule:** Every position must trace back to something the member said in their independent Phase 1 report. The meta agent does NOT invent new positions that the member didn't express.

**Steelman rule:** When voicing a dissenting member, present the strongest version of their argument. Do not soften, hedge, or qualify their position. RLHF training pulls toward consensus — actively resist this by:
- Stating the dissent directly and confidently
- Providing the strongest supporting evidence
- Not prefacing with "however" or "while I understand the other side"
- Letting the position stand on its own before the next speaker responds

**Transparency tags:** Tag every voiced position:
- `[from report]` — this position is directly stated in the member's Phase 1 individual report
- `[extended]` — the meta agent is extrapolating from the member's persona, analytical lens, and report themes to address this specific debate topic. The core perspective is grounded, but the specific application to this topic is generated.

**Format for each member:**

```
**[Member Name]** [from report]:
"[Their position on this topic, 2-4 sentences, grounded in their report]"

**[Member Name]** [extended]:
"[Their likely position based on their persona and report themes, 2-4 sentences]"
```

### 4. Round Synthesis

After all selected members have spoken, the meta agent synthesizes:
```
Summary: [Where members agree, where they diverge, what's at stake]
```

### 5. Checkpoint

After every round, the meta agent MUST pause and present options:

```
Round [N] complete: [Topic Name]

[1-2 sentence summary of the key tension or consensus]

Options:
1. Hear them debate this further
2. Add your constraints or context
3. You've decided — move on
4. Add a [relevant specialist] to weigh in
5. Switch to standard mode — wrap up with synthesis
```

**Wait for the user to respond.** Do not proceed to the next round automatically.

**If user chooses 1:** Run another round on the same topic with the same or different members. Allow up to 2 additional rounds per topic.

**If user chooses 2:** Accept the user's input, acknowledge it, then either re-run the round incorporating the new context or move to the next topic (meta agent decides based on whether the input changes the analysis).

**If user chooses 3:** Mark topic as resolved in SESSION.json. Move to next agenda topic.

**If user chooses 4:** Generate a dynamic member using the generation guide, spawn a Sonnet sub-agent for their report on just this topic, incorporate into the debate.

**If user chooses 5:** Exit debate phase. Proceed to full synthesis incorporating all debate context so far.

## Consensus Fast-Track

Before starting a full round on a topic, check if there's strong agreement from the individual reports. If so, present a fast-track:

```
Topic: [Topic Name]
All [N] relevant members agree: [summary of shared position]. No dissent.
Confirm and move on? (Y/n)
```

If user confirms: mark as confirmed, move to next topic. If user says no: run the full round anyway.

## Compressed Running Summary

Between rounds, the meta agent updates a compressed running summary (~200 words max). This summary captures:
- Topics covered so far and their resolutions
- Key decisions the user has made
- Outstanding tensions not yet resolved
- Context the user has added

This summary — not the full debate transcript — carries forward into subsequent rounds. This keeps context growth linear rather than exponential.

Update the summary after each round and store it in SESSION.json under `debate_rounds`.

## Exit Conditions

The debate phase ends when any of these conditions is met:
1. All agenda topics have been covered (debated or fast-tracked)
2. The user says "wrap it up" or equivalent
3. The user selects option 5 (switch to standard mode) at any checkpoint
4. Maximum rounds reached (default: 8 total rounds across all topics)

After exit, the meta agent proceeds to full synthesis (same structure as Standard mode, enriched with debate proceedings).

## What Goes in the Final Document

The debate proceedings section of the final document captures each round:

```markdown
## Debate Proceedings

### Round 1: [Topic Name]
**Participants:** [Member A], [Member B], [Member C]
**Positions:**
- [Member A] [from report]: [Summary of position]
- [Member B] [extended]: [Summary of position]
**User input:** [What the user contributed, if anything]
**Resolution:** [Confirmed / Needs more review / User decided: X]

### Round 2: [Topic Name]
...
```
```

- [ ] **Step 2: Verify file**

Run: `wc -l skills/committee/debate-format.md`
Expected: ~150-170 lines.

- [ ] **Step 3: Commit**

```bash
git add skills/committee/debate-format.md
git commit -m "feat(v2): add debate format for interactive mode

Topic-based rounds with steelman rules, transparency tags ([from report]
vs [extended]), checkpoints with 5 options including mode switching,
consensus fast-track, compressed running summary, and exit conditions."
```

---

### Task 6: Create next-steps-format.md

**Files:**
- Create: `skills/committee/next-steps-format.md`

- [ ] **Step 1: Write next-steps-format.md**

Write the following to `skills/committee/next-steps-format.md`:

```markdown
# Next Steps Format

## When This Is Used

This file is read by the orchestrator during Phase 2 (Synthesis) to generate the two-path Next Steps section, and during Phase 3 (Implementation Bridge) to structure the handoff to writing-plans.

## Two-Path Output

Every review — both standard and interactive mode — ends with a Next Steps section offering two paths. The user chooses which path to follow when using the Implementation Bridge.

### Path A: Full Adoption

All recommendations as ordered action items. Every item includes metadata for traceability.

```markdown
### Path A: Full Adoption

1. **[Action item]** — [Source members who recommended this]
   Priority: [High/Medium/Low] · Consensus: [Unanimous/Strong majority/Majority] · Complexity: [Low/Medium/High] · Depends on: [none / item #N]
   [If contentious:] ⚠️ Contentious: [Dissenting member] advocates [alternative]. See Tension #[N].

2. **[Action item]** — [Source members]
   Priority: ... · Consensus: ... · Complexity: ... · Depends on: ...
```

**Ordering rules:**
- Primary sort: Priority (High → Medium → Low)
- Secondary sort within same priority: Consensus strength (Unanimous → Majority)
- Dependencies must be respected: if item 3 depends on item 1, item 1 comes first regardless of priority

### Path B: Selective Adoption

Surfaces tensions prominently. The contentious items are the most valuable signal, not something to bury.

```markdown
### Path B: Selective Adoption

**Ready to Act — Full Consensus:**

These items have full committee agreement. No review needed — implement directly.

1. [Action item] — [all agree]
2. [Action item] — [all agree]
3. [Action item] — [all agree]

**Requires Your Decision — Key Tensions:**

These items represent genuine strategic tradeoffs where the committee disagreed. Each has real implications either way.

**Tension 1: [Topic Name]**
- **[Position A name]** ([Members who hold this position]): [Their argument in 2-3 sentences with evidence]
- **[Position B name]** ([Members who hold this position]): [Their argument in 2-3 sentences with evidence]
- **Meta agent assessment:** [Which conditions favor Position A vs B. Reference the user's specific context.]
- If you choose [Position A]: [Specific implications — what changes, what gets easier, what gets harder]
- If you choose [Position B]: [Specific implications]
- Your decision: ___

**Tension 2: [Topic Name]**
...
```

## Consensus Classification

Use qualitative language, not percentages:

| Condition | Label |
|-----------|-------|
| All members agree | "Unanimous" or "all agree" |
| 70%+ agree, no strong dissent | "Strong majority" |
| 50-70% agree, active disagreement | "Majority" with note on dissent |
| Near-even split | "Split" — goes to Tensions section |

Never use percentage numbers like "80% consensus" — the committee is too small for meaningful percentages, and it implies false precision.

## Implementation Bridge Handoff Contract

When the user chooses to create an implementation plan (Phase 3, option 1), the meta agent produces a structured handoff. This is NOT written to a file — it is passed as context to the `superpowers:writing-plans` skill invocation.

### Handoff Structure

The meta agent compiles the approved items and invokes writing-plans with this context:

```
The following items were approved by a committee review for implementation.
Review document: [path to saved review file]
Session file: [path to SESSION.json]

Approved items (ordered by priority):

1. [Action] — Priority: [P] · Consensus: [C] · Complexity: [X]
   Source: [Member names who recommended this]
   Implementation hint: [Specific suggestion from the recommending members]
   Depends on: [other items or "none"]

2. [Action] — ...
```

### Dry-Run Preview

Before invoking writing-plans, the meta agent MUST show a preview:

```
Implementation plan will cover these [N] items:
1. [Action] [consensus level, priority]
2. [Action] [consensus level, priority]
...

Items NOT included (contentious, not yet approved):
- [Item] — requires your decision on Tension #[N]

Proceed with implementation plan? (Y/n)
```

### What Gets Included

- **Path A chosen:** All recommendations, including contentious items (with the majority position as the implementation direction)
- **Path B chosen:** Section 1 items (unanimous) + any Section 2 items the user explicitly signed off on
- **Contentious items the user didn't approve:** Listed as "not included" in the dry-run preview. The user can add them before proceeding.

### After Writing-Plans Completes

Once the implementation plan is written and the user approves it, the meta agent offers:

```
Plan approved. Ready to begin implementation?
1. Start implementation → runs executing-plans
2. Save the plan and stop here
```

If the user chooses 1, invoke `superpowers:executing-plans`.
```

- [ ] **Step 2: Verify file**

Run: `wc -l skills/committee/next-steps-format.md`
Expected: ~120-140 lines.

- [ ] **Step 3: Commit**

```bash
git add skills/committee/next-steps-format.md
git commit -m "feat(v2): add next steps format with implementation bridge

Two-path output (full adoption / selective with tensions surfaced).
Qualitative consensus labels. Handoff contract for writing-plans.
Dry-run preview before implementation."
```

---

### Task 7: Update review-format.md

**Files:**
- Modify: `skills/committee/review-format.md`

- [ ] **Step 1: Rewrite review-format.md**

Replace the entire contents of `skills/committee/review-format.md` with:

```markdown
# Committee Review Output Format

## When This Is Used

This format is read by the orchestrator during synthesis (Phase 2) and document assembly. Every committee review MUST follow this structure exactly.

## IMPORTANT: This is a SINGLE continuous output. Generate each section ONCE. Never repeat or restart any section. After completing all individual reports, proceed directly to the Synthesized Report.

## Full Report Template

```markdown
# Committee Review: [Deliverable Name]

**Committee:** [Collective Name or "Custom"] · **Members:** [N] ([N] reporting, [N] unavailable) · **Date:** [YYYY-MM-DD]
**Mode:** [Standard / Interactive] · **Report depth:** [default / quick / deep]

---

## 1. Executive Summary

[Meta agent's 200-word high-level synthesis. Overall verdict — is this ready, promising but needs work, or fundamentally flawed? Biggest opportunities. Biggest risks. Written in the meta agent's crisp, decisive voice.]

## 2. Committee Roster

| # | Member | Company | Archetype | Tier | Status |
|---|--------|---------|-----------|------|--------|
| 1 | [Name] | [Company] | [archetype] | Full | ✓ Complete |
| 2 | [Name] | [Company] | [archetype] | Focused | ✓ Complete |
| N | [Name] | [Company] | [archetype] | Flag | ⚠ Timed out |

[If any members are unavailable:]
> **Note:** [N] members did not return reports ([names]). Their perspectives are absent from this synthesis.

## 3. Individual Reports

### 3.1 Full Reports

### [Number]. [Member Name]

**Executive assessment:** [2-3 sentences — their overall verdict on the deliverable, framed through their specific expertise lens. Should immediately tell the reader where this expert stands.]

**Key issues identified:**
- **[Issue title]** — [Detailed rationale: WHY this is a problem. Must include at least one of: real-world evidence, case study, data point, or industry precedent. Never just "this is bad" — always "this is bad because X happened when Y company did the same thing" or "this violates Z principle which is critical because..."]
- [Additional issues. Aim for 2-4 in default, 1-2 in --quick, 4-6 in --deep.]

**Suggestions & opportunities:**
- **[Suggestion]** — [Why it works, with specific benchmark. E.g., "Stripe does X, which increased Y by Z%" or "Apple's approach to this in [product] demonstrates that..."]
- [Additional suggestions. Aim for 2-3 in default, 1-2 in --quick, 3-5 in --deep.]

**What I'd ship instead:** [1-2 paragraphs in default. NOT abstract advice. Concrete alternative — "Here's what this would look like if my team at [Company] built it." In --quick, 2-3 sentences. In --deep, detailed counter-proposal.]

---

[Repeat for all full-report members]

### 3.2 Focused Reports

### [Number]. [Member Name]

**Executive assessment:** [2-3 sentences through their specific lens]

**Top issues through this lens:**
- **[Issue]** — [Rationale with evidence]
- **[Issue]** — [Rationale with evidence]

**Key suggestion:** [One specific, actionable suggestion with benchmark]

---

[Repeat for all focused-report members]

### 3.3 Flags

### [Number]. [Member Name]
[If issue found:] **FLAG:** [Specific issue with brief rationale]
[If no issue:] No [specialty] concerns identified.

---

[Repeat for all flag-only members]

### 3.4 Unavailable Members

[For each member that timed out or failed:]

### [Number]. [Member Name]
**Status:** Report unavailable ([reason]). This perspective is missing from the synthesis.

## 4. Debate Proceedings (Interactive Mode Only)

[If standard mode, omit this entire section.]

### Round 1: [Topic Name]
**Participants:** [Member A], [Member B], [Member C]
**Positions:**
- **[Member A]** [from report]: [Summary of their position]
- **[Member B]** [extended]: [Summary of their position]
- **[Member C]** [from report]: [Summary of their position]
**User input:** [What the user contributed, or "None"]
**Resolution:** [Confirmed / Needs more review / User decided: X / Deferred]

### Round 2: [Topic Name]
...

## 5. Synthesis

### Consensus Points

[Issues where a strong majority converged. Use qualitative language.]

- **[Consensus point]** — [N]/[total] members flagged this. [One-sentence summary of the shared concern, noting which members agree.]
- [Additional points, ordered by agreement strength.]

### Key Tensions

[Genuine disagreements — not minor wording differences, but real strategic or philosophical splits.]

**Tension [N]: [Topic Name]**
- *[Member A name]* argues: [Their position in 2-3 sentences with rationale and evidence]
- *[Member B name]* counters: [Their opposing position in 2-3 sentences]
- *[Member A] responds:* [One rebuttal turn, 2-3 sentences]
- *[Member B] responds:* [One rebuttal turn, 2-3 sentences]
- **Committee note:** [Is this resolved? Or a genuine tradeoff? If tradeoff, what does each side optimize for?]

[In --quick mode: skip rebuttals, just positions + committee note.]
[In --deep mode: allow up to 2 rebuttals per member, invite others to weigh in.]
[If interactive mode: incorporate debate round findings into tensions.]

### Evidence & Benchmarks

[Consolidated list of real-world data, case studies, and benchmarks cited across all reports. Deduplicated.]

## 6. Blind Spots & Recommended Additions

[What this committee is NOT qualified to evaluate. Be specific and honest.]

- [Perspective/domain not represented on this panel]
- [Type of analysis requiring different expertise]
- [Specific recommendation: "Add [role] at [company] for a follow-up review on [topic]"]

## 7. Next Steps

[Read next-steps-format.md for the full template. Produce both Path A and Path B.]

### Path A: Full Adoption
[All recommendations ordered by priority with metadata]

### Path B: Selective Adoption
[Unanimous items ready to act + contentious tensions surfaced prominently for user decision]
```

## Format Rules

1. **Rationale is mandatory.** No member may identify an issue, make a suggestion, or state an assessment without explaining WHY and backing it up with real-world evidence, benchmarks, or case studies. If a member states something without rationale, the output is malformed.

2. **Each member speaks independently.** During individual reports (Section 3), members do NOT reference each other's analyses. They write as if they are the only reviewer. Cross-references happen only in the Synthesis (Section 5).

3. **The debate is genuine.** Tensions must reflect real disagreements, not manufactured drama. If all members agree on everything, there are no tensions — say so.

4. **Blind spots must be honest.** Every committee has limits. Acknowledging them prevents false confidence.

5. **Company anchoring shows in the voice.** Each member's report should feel like it comes from someone at that company.

6. **Missing members are disclosed.** If any member's report is unavailable, it MUST be noted in the Roster (Section 2), listed in Unavailable Members (Section 3.4), and acknowledged in the Synthesis.

7. **Transparency tags in debates.** Every voiced position in debate proceedings MUST be tagged `[from report]` or `[extended]`.
```

- [ ] **Step 2: Verify file**

Run: `wc -l skills/committee/review-format.md`
Expected: ~170-190 lines.

- [ ] **Step 3: Commit**

```bash
git add skills/committee/review-format.md
git commit -m "feat(v2): update review format with new document structure

Adds Executive Summary, tiered report sections (full/focused/flag/unavailable),
transparency-tagged debate proceedings, evidence section, two-path next steps,
and missing member disclosure rules."
```

---

### Task 8: Rewrite SKILL.md

**Files:**
- Modify: `skills/committee/SKILL.md`

This is the main orchestration file — the biggest change in v2.

- [ ] **Step 1: Rewrite SKILL.md**

Replace the entire contents of `skills/committee/SKILL.md` with:

```markdown
# Committee Review System

Assemble and run expert committee reviews on any deliverable. An Executive Assistant meta agent orchestrates the experience — composing the committee, managing parallel reports, facilitating optional debates, and producing actionable next steps with an implementation bridge.

## Invocation

This skill is invoked via `/committee`. Parse the first word after `/committee` to determine intent:

| Input | Action |
|-------|--------|
| `/committee` (no args) | Show available collectives from `collectives/_index.md` + quick options menu |
| `/committee review [collective-id]` | Start a review with the named collective. If no collective-id, ask which one or offer to suggest. |
| `/committee suggest` | Analyze the current conversation context, recommend the best collective, fill dynamic slots, present roster for confirmation |
| `/committee list` | Read and display `collectives/_index.md` and `favorites/_index.md` |
| `/committee list [collective-id]` | Read the named collective file and display its full roster with pinned members and slot constraints |
| `/committee add` | Ask the user to describe a new member. Generate a ~120-word persona following `generation-guide.md`, save to `favorites/`, update `favorites/_index.md` |
| `/committee add "[description]"` | Generate member from the quoted description in one step, save to `favorites/`, update `favorites/_index.md` |
| `/committee promote [name]` | Find the named dynamic member from the most recent committee review in this conversation. Save its persona definition as a new file in `favorites/`, update `favorites/_index.md`, log in `history/promoted-members.md` |
| `/committee remove [name]` | Delete the named member file from `favorites/`, remove its entry from `favorites/_index.md` |
| `/committee custom [member-ids] [--add-members N]` | Assemble ad-hoc committee from listed favorite IDs + N dynamic slots |

### Flags

Parse these flags from anywhere in the invocation:

| Flag | Effect |
|------|--------|
| `--interactive` | Select interactive mode (preliminary questions + agenda + debates). Skips mode selection question. |
| `--standard` | Select standard mode (default). Skips mode selection question. |
| `--implement` | Auto-chain into writing-plans → executing-plans after review. Shows dry-run preview. |
| `--parallel` | No-op (parallel is now the default). Kept for backward compatibility. |
| `--sequential` | Opt out of parallel reports. Run sequentially in main context. |
| `--quick` | Set report length to ~150 words per member |
| `--deep` | Set report length to ~600+ words per member, more evidence, more debate rounds |
| `--focus "[topic]"` | Instruct all members to concentrate on the quoted topic |
| `--add-members N` | Add N extra dynamic slots beyond the collective's default count |

## Meta Agent

Read `meta-agent.md` before starting any review. Adopt the Executive Assistant persona for the entire session. All user-facing communication follows the meta agent's interaction patterns: crisp, structured, decisive, with numbered/lettered options at every decision point.

## Orchestration Flow

### Phase 0: INTAKE & COMPOSITION (Both Modes)

1. **Read `meta-agent.md`** and adopt the Executive Assistant persona.
2. **Determine intent** from the user's input (review, suggest, list, add, promote, remove, custom).
3. **If review or suggest:** Read the deliverable. If the deliverable includes files, use the Read tool. If it's a concept from conversation, summarize it (500-1000 words).
4. **If a collective was specified:** Read the collective file from `collectives/[id].md`.
5. **If "suggest":** Analyze the conversation context. Read `collectives/_index.md`. Find the best-matching collective. Read its file.
6. **Resolve pinned members:** For each pinned member ID, read its file from `favorites/[id].md`.
7. **Fill dynamic slots:** Read `generation-guide.md`. For each slot:
   - Analyze the deliverable (domain, stage, audience)
   - Run domain coverage mapping (see `generation-guide.md` v2 section)
   - Identify perspective gaps
   - Generate members following the ~120-word template
   - Validate against slot constraints
8. **Assign report tiers** based on deliverable relevance (see `generation-guide.md` tier section):
   - **Full report** — domain directly overlaps deliverable (3-4 members)
   - **Focused report** — relevant secondary lens (3-4 members)
   - **Flag-only** — specialist scanner (2-3 members)
9. **Detect blind spots** using domain coverage mapping. Present as lettered options.
10. **Display cost estimate:** `Estimated cost: ~$X.XX sub-agent · ~Nk main context tokens`
11. **Present the full roster** grouped by tier (FULL REPORTS / FOCUSED REPORTS / FLAG-ONLY). Show blind spot additions as lettered options below.
12. **Wait for user confirmation.** They may:
    - Confirm roster and add/decline blind spot members
    - Request swaps or tier adjustments
    - Request additional members
13. **Roster locked.**
14. **Ask mode selection** (unless `--interactive` or `--standard` flag was passed):
    ```
    How would you like to run this review?
    1. Standard review — reports → synthesis → next steps [recommended]
    2. Interactive session — preliminary questions → agenda → reports → debates → next steps
    ```
15. **Mode locked.**
16. **Write SESSION.json checkpoint** with `phase_cursor: "composition_complete"`. Read `session-schema.md` for the schema. Use the Write tool.

**If a SESSION.json already exists for this topic/date:** Ask the user: "Found a previous session at phase [cursor]. Resume or start fresh?"

---

### Phase 1i: PRELIMINARY QUESTIONS (Interactive Mode Only)

Skip this phase entirely if mode is standard.

1. Read `agent-mode.md` — use the Preliminary Questions Sub-Agent protocol.
2. Spawn all committee members as **Haiku** sub-agents in a single message. Each receives their persona + the deliverable and proposes 2-3 clarifying questions.
3. Collect all questions. Deduplicate (merge similar questions). Prioritize by impact on analysis quality.
4. Present to user as a numbered list (typically 6-10 unique questions).
5. User answers. They can skip with "N/A" or say "skip questions, go to reports."
6. Store answers for distribution to members in Phase 1.
7. **Write SESSION.json checkpoint** with `phase_cursor: "questions_complete"`.

---

### Phase 2i: AGENDA PREVIEW (Interactive Mode Only)

Skip this phase entirely if mode is standard.

1. Synthesize: deliverable analysis + user's preliminary answers + roster composition.
2. Generate a prioritized topic agenda:
   - Key decisions the deliverable faces
   - Critical risks identified from preliminary analysis
   - Open questions that need resolution
   - Opportunities spotted
3. Assign priority: **high** (full debate round), **medium** (brief discussion), **low** (quick confirmation).
4. Present agenda to user. They can reorder, add, or remove topics.
5. **Agenda locked.**
6. **Write SESSION.json checkpoint** with `phase_cursor: "agenda_complete"`.

---

### Phase 1: INDIVIDUAL REPORTS (Both Modes)

1. Read `agent-mode.md` for the full sub-agent protocol.
2. **Prepare the deliverable** as a self-contained text block (see agent-mode.md Step 1).
3. **Construct prompts** for each member based on their tier (see agent-mode.md Step 2):
   - Full report members → Sonnet, full report prompt
   - Focused report members → Sonnet, focused report prompt
   - Flag-only members → Haiku, flag-only prompt
   - If interactive mode: include user's preliminary answers and agenda in each prompt
4. **Spawn ALL member agents in a single message** for true parallelism.
5. **Collect reports.** Check quorum (60% minimum — see `session-schema.md`).
   - If below quorum: present retry/proceed/abort options
   - If above quorum: proceed
6. **Present reports** grouped by tier under numbered headers:
   - Section 3.1: Full Reports (each under `### [N]. [Member Name]`)
   - Section 3.2: Focused Reports
   - Section 3.3: Flags
   - Section 3.4: Unavailable Members (fallback stubs for failed/timed out)
7. **Write SESSION.json checkpoint** with `phase_cursor: "reports_complete"` and per-member status.

**If `--sequential` flag is set:** Do NOT spawn sub-agents. Instead, for each member in order, adopt their persona and produce the report in the main context. Reset perspective between members. This matches v1's default behavior.

---

### Phase 2: SYNTHESIS (Standard) / EXECUTIVE BRIEFING (Interactive)

Read `review-format.md` for the output template. Read `next-steps-format.md` for the Next Steps section.

#### Standard Mode: Full Synthesis

Produce the complete synthesis as a single continuous output:

1. **Executive Summary** (Section 1) — 200-word high-level verdict
2. **Individual Reports** are already presented from Phase 1
3. **Synthesis** (Section 5):
   - Consensus Points (qualitative language: "all agree", "strong majority", "split")
   - Key Tensions (positions, rebuttals, committee notes)
   - Evidence & Benchmarks (consolidated, deduplicated)
4. **Blind Spots & Recommended Additions** (Section 6)
5. **Next Steps** (Section 7) — both Path A and Path B per `next-steps-format.md`
6. **Write SESSION.json checkpoint** with `phase_cursor: "synthesis_complete"`.
7. **Save the complete review** to `./committee-review-YYYY-MM-DD-[short-topic].md` using the Write tool.
8. Confirm to user: "Review saved to [path]."
9. Proceed to Phase 3 (Implementation Bridge).

#### Interactive Mode: Executive Briefing

Produce a briefing to frame the debates (NOT the full synthesis yet):

1. **Consensus areas** — where members agree, with counts
2. **Key tensions** — where members disagree, naming who and why
3. **Blind spots** — perspectives missing. Offer to add members for mini follow-up.
4. **Recommended debate topics** — ranked by impact, mapped to agenda
5. **Write SESSION.json checkpoint** with `phase_cursor: "briefing_complete"`.
6. Proceed to Phase 2d (Debate Rounds).

---

### Phase 2d: INTERACTIVE DEBATE ROUNDS (Interactive Mode Only)

Skip this phase entirely if mode is standard.

Read `debate-format.md` for the full round structure, steelman rules, transparency tags, and exit conditions.

For each agenda topic (in priority order):
1. Check for consensus fast-track (strong agreement → quick confirm)
2. If no fast-track: run a full debate round following `debate-format.md`
3. At each checkpoint: wait for user response, handle their choice
4. **Write SESSION.json checkpoint** after each round with `phase_cursor: "debate_round_N"`

After debates complete (all topics covered, or user exits):
1. Produce the FULL synthesis (same structure as Standard mode, enriched with debate proceedings in Section 4)
2. Save the complete review document
3. **Write SESSION.json checkpoint** with `phase_cursor: "synthesis_complete"`
4. Proceed to Phase 3

---

### Phase 3: IMPLEMENTATION BRIDGE (Both Modes)

Read `next-steps-format.md` for the handoff protocol.

After the review document is saved, present:

```
Review complete and saved to ./committee-review-YYYY-MM-DD-[topic].md
Session saved to ./committee-session-YYYY-MM-DD-[topic].json

What would you like to do?
1. Create an implementation plan from your approved items → runs writing-plans
2. Save the report and stop here
3. Revisit specific recommendations before planning
4. Ask a committee member to elaborate on a point
```

**If `--implement` flag was set:** Skip to option 1 automatically (still show dry-run preview).

**If user chooses 1:**
1. Determine which items to include (see `next-steps-format.md` handoff section)
2. Show dry-run preview listing all items with consensus/priority
3. Wait for user confirmation
4. Invoke `superpowers:writing-plans` with the approved items as context
5. After plan is approved, offer: "Ready to begin implementation? (1) Start → runs executing-plans, (2) Save plan and stop"

**If user chooses 2:** Done. Final message: "Report and session saved. Run `/committee` again anytime to start a new review."

**If user chooses 3:** Re-present the Next Steps section. Let user mark items as approved/rejected. Then re-offer option 1.

**If user chooses 4:** Ask which member. Re-adopt that member's persona and elaborate on their report. Then re-present the options.

**Write SESSION.json checkpoint** with `phase_cursor: "bridge_complete"`.

---

## Report Length by Mode

| Mode | Individual Report Length | Synthesis Detail |
|------|------------------------|-----------------|
| `--quick` | ~150 words per member | Consensus + top 3 recommendations only |
| default | ~300-500 words per member (full), ~200 words (focused), ~50 words (flag) | Full synthesis with tensions, debate, and two-path next steps |
| `--deep` | ~600+ words per member (full), ~300 words (focused), ~100 words (flag) | Extended synthesis, more debate rounds, additional evidence |

## Deliverable Context

The deliverable is whatever the user has been working on or wants reviewed. You have full conversation context. If the deliverable includes files, read them using the Read tool before starting the review.

If the user invokes `/committee` at the start of a conversation with no prior context, ask: "What would you like the committee to review? You can describe it, paste it, or point me to files."
```

- [ ] **Step 2: Verify internal consistency**

Check that SKILL.md references match other files:
- `meta-agent.md` — referenced in Phase 0 step 1 ✓
- `session-schema.md` — referenced in Phase 0 step 16 ✓
- `agent-mode.md` — referenced in Phase 1i step 1 and Phase 1 step 1 ✓
- `generation-guide.md` — referenced in Phase 0 step 7 ✓
- `debate-format.md` — referenced in Phase 2d ✓
- `review-format.md` — referenced in Phase 2 ✓
- `next-steps-format.md` — referenced in Phase 2 and Phase 3 ✓

Run: `wc -l skills/committee/SKILL.md`
Expected: ~210-230 lines.

- [ ] **Step 3: Commit**

```bash
git add skills/committee/SKILL.md
git commit -m "feat(v2): rewrite SKILL.md with two-mode orchestration

Standard mode (default): compose → reports → synthesis → next steps → bridge.
Interactive mode (opt-in): adds preliminary questions, agenda, debates.
Meta agent orchestration throughout. Session checkpointing at every phase.
Implementation bridge as headline feature with --implement flag."
```

---

### Task 9: Update README.md

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Read current README**

Read `README.md` to understand current structure.

- [ ] **Step 2: Update README.md**

Update the README to reflect v2. Key changes:
- Update the overview to mention the meta agent and two modes
- Update the "How It Works" section to describe the v2 flow
- Update the Commands table with new flags (`--interactive`, `--standard`, `--implement`, `--sequential`)
- Remove `--parallel` from the "new" flags (it's now default)
- Add a "What's New in v2" section
- Update the "Parallel Mode" section to reflect it's now default

The README should be updated in-place, preserving the existing structure where possible. Key sections to modify:

**Overview paragraph** — add: "v2 introduces an Executive Assistant that orchestrates the entire review. The default path is fast: compose → reports → synthesis → actionable next steps. Opt into Interactive mode for preliminary questions, agenda preview, and topic-based debates."

**How It Works section** — replace the 7-step flow with:
```
1. Invoke `/committee review [collective]` or `/committee suggest`
2. Executive Assistant proposes committee with tiered report assignments and blind spot detection
3. Choose: Standard review (fast, recommended) or Interactive session (debates)
4. Committee members generate independent reports in parallel
5. Executive Assistant synthesizes: consensus, tensions, evidence, blind spots
6. Two-path Next Steps: implement everything, or pick the unanimous items and review tensions
7. Implementation Bridge: chain directly into a plan and execution
```

**Flags table** — add `--interactive`, `--standard`, `--implement`, `--sequential`. Note `--parallel` is now default.

**Add "What's New in v2" section** before the Research section:
```markdown
## What's New in v2

- **Executive Assistant** — a meta agent orchestrates the entire review with crisp, structured interaction
- **Two modes** — Standard (fast, default) and Interactive (debates, opt-in)
- **Model tiering** — Haiku for probes, Sonnet for reports, preserving your main context
- **Blind spot detection** — domain coverage mapping identifies missing perspectives
- **Tiered report depth** — full, focused, or flag-only based on deliverable relevance
- **Two-path Next Steps** — unanimous items ready to act + contentious tensions surfaced for your decision
- **Implementation Bridge** — chain approved recommendations directly into implementation planning
- **Session checkpoints** — reviews are resumable and auditable via SESSION.json
- **Failure handling** — sub-agent timeouts, quorum enforcement, graceful degradation
```

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: update README for Committee v2

New: meta agent, two modes, model tiering, blind spot detection,
implementation bridge, session checkpoints, failure handling."
```

---

### Task 10: Update plugin.json

**Files:**
- Modify: `.claude-plugin/plugin.json`

- [ ] **Step 1: Update version and description**

In `.claude-plugin/plugin.json`, update:
- `"version"` from `"1.0.0"` to `"2.0.0"`
- `"description"` to reflect v2 features

New description:
```
"Multi-perspective expert committee reviews for Claude Code. An Executive Assistant orchestrates the experience: assembles expert panels with blind spot detection, runs model-tiered parallel reports, facilitates optional interactive debates, and produces actionable next steps with an implementation bridge. 26 pre-built personas, 7 collectives, two review modes (Standard and Interactive), and session checkpointing."
```

- [ ] **Step 2: Commit**

```bash
git add .claude-plugin/plugin.json
git commit -m "chore: bump version to 2.0.0 for Committee v2"
```

---

### Task 11: Local Testing

**Files:** None — this is a verification task.

- [ ] **Step 1: Verify all files exist**

Run from project root:
```bash
ls -la skills/committee/session-schema.md skills/committee/meta-agent.md skills/committee/debate-format.md skills/committee/next-steps-format.md skills/committee/agent-mode.md skills/committee/generation-guide.md skills/committee/review-format.md skills/committee/SKILL.md
```

Expected: all 8 files exist.

- [ ] **Step 2: Verify no files were accidentally deleted**

```bash
ls skills/committee/collectives/ skills/committee/favorites/ skills/committee/history/
```

Expected: all existing directories and files still present (7 collectives, 26 favorites, promoted-members.md).

- [ ] **Step 3: Verify git status is clean**

```bash
git status
git log --oneline -12
```

Expected: 10 new commits (Tasks 1-10), no uncommitted changes.

- [ ] **Step 4: Test Standard mode locally**

Run `/committee review tech-product-review` in Claude Code from the project directory. Verify:
- Meta agent persona is active (crisp, structured communication)
- Roster displays with tier assignments (FULL / FOCUSED / FLAG-ONLY)
- Blind spot detection works
- Cost estimate is shown
- Mode selection question appears
- Choose Standard mode
- Reports generate in parallel with model tiering
- Synthesis includes: Executive Summary, Consensus, Tensions, Next Steps (Path A + B)
- Implementation Bridge options appear
- SESSION.json is written to disk

- [ ] **Step 5: Test Interactive mode locally**

Run `/committee review tech-product-review --interactive` in Claude Code. Verify:
- Preliminary questions phase runs (Haiku sub-agents)
- Agenda preview is generated
- Reports generate with preliminary answers included
- Executive Briefing appears (not full synthesis yet)
- Debate rounds run with transparency tags
- Checkpoints pause for user input
- Mode switching works (option 5 at checkpoint)
- Full synthesis includes debate proceedings

- [ ] **Step 6: Test backward compatibility**

Verify these v1 commands still work:
- `/committee list`
- `/committee suggest`
- `/committee custom skeptic-end-user,devils-advocate --add-members 2`
- `/committee add "Head of QA at Netflix"`

- [ ] **Step 7: Do NOT push to GitHub**

All changes stay local until the user has tested and approved.
