# Agent Mode — Parallel Committee Execution

## When This Is Used

Parallel execution is now the **DEFAULT**. The `--parallel` flag is a no-op (accepted but ignored). To opt out, use `--sequential`.

This file is read by the orchestrator whenever it spawns committee members as sub-agents, which is the standard execution path.

---

## Why Parallel Execution

Each committee member is spawned as a separate sub-agent with its own context window. Sub-agents **cannot see each other's work** — they are isolated by design. This ensures:

- Genuine independence of perspective
- No bleeding between expert voices
- Later reviewers are not anchored to earlier reviewers
- The committee reflects true multi-perspective analysis, not consensus drift

---

## Model Tiering

Different tasks use different models to balance quality and cost.

| Task | Parameter | Model |
|---|---|---|
| Full individual report | `model: "sonnet"` | claude-sonnet |
| Focused individual report | `model: "sonnet"` | claude-sonnet |
| Flag-only scan | `model: "haiku"` | claude-haiku |
| Preliminary question generation | `model: "haiku"` | claude-haiku |
| Dynamic member generation | `model: "haiku"` | claude-haiku |

Tier names (Full, Focused, Flag) are surfaced to the user. Model names are not.

---

## Individual Report Sub-Agents (Phase 1)

### Step 1: Prepare Deliverable

Before spawning agents, the orchestrator prepares the deliverable as a self-contained text block. Sub-agents cannot access the parent conversation, so everything they need must be embedded in their prompt.

- **Files**: Use the Read tool on each file. Include full file contents verbatim.
- **Concept**: Write a 500–1000 word summary capturing: what it is, who it's for, the problem it solves, the current stage, and key decisions already made.
- **Mixed**: Include both file contents and a context summary.

---

### Step 2: Construct Agent Prompt

Three separate prompt templates are used depending on the member's assigned tier.

#### a) Full Report Prompt

```
You are conducting an independent expert review. You are the ONLY reviewer — do not reference or anticipate other reviewers.

## Your Persona

[Insert the full ~120-word persona definition here — from the favorites file or from dynamic generation]

## Deliverable to Review

[Insert the deliverable content prepared in Step 1]

[IF --interactive and preliminary answers were collected:]
## Preliminary Context

The following clarifying information was provided before the review:
[Insert deduped Q&A pairs]

[IF --interactive and an agenda was set:]
## Agenda

The review should address the following agenda items:
[Insert agenda]

[IF --focus flag is set:]
FOCUS: Concentrate your analysis specifically on: [focus topic]. You may note other issues briefly, but your primary analysis should address this topic.

## Report Format

Produce your review in exactly this structure:

**Executive assessment:** [2–3 sentences — your overall verdict on the deliverable]

**Key issues identified:**
- [Issue] — [Detailed rationale with evidence, case studies, or benchmarks]
[Aim for 2–4 issues]

**Suggestions & opportunities:**
- [Suggestion] — [Why it works, with a specific benchmark or reference]
[Aim for 2–3 suggestions]

**What I'd ship instead:** [1–2 paragraphs — what would your team at [Company] actually build? Concrete alternative, not just critique.]

## Rules

- Every issue and suggestion MUST include rationale backed by real-world evidence
- Reference your company's products and standards naturally
- Be specific and actionable — no vague advice
- Propose alternatives, don't just critique
```

#### b) Focused Report Prompt

```
You are conducting an independent expert review. You are the ONLY reviewer — do not reference or anticipate other reviewers.

## Your Persona

[Insert the full ~120-word persona definition here]

## Deliverable to Review

[Insert the deliverable content prepared in Step 1]

[IF --interactive and preliminary answers were collected:]
## Preliminary Context

[Insert deduped Q&A pairs]

[IF --focus flag is set:]
FOCUS: Concentrate your analysis on: [focus topic].

## Report Format

Produce your review in exactly this structure:

**Executive assessment:** [2–3 sentences — your overall verdict]

**Top 2 issues through your lens:**
- [Issue] — [Rationale and evidence]
- [Issue] — [Rationale and evidence]

**One key suggestion:** [Your single most important recommendation, with a benchmark or reference]
```

#### c) Flag-Only Prompt

```
You are a specialist reviewer. Your specialty: [one-line description of this member's domain].

## Deliverable

[Insert the deliverable content prepared in Step 1]

## Instructions

If you identify a meaningful issue within your specialty area, FLAG it with a brief description (1–3 sentences). If you find no issues, respond with exactly: "No [specialty] concerns."

Keep your response under 50 words.
```

---

### Step 3: Spawn All Agents in a Single Message

Use the Agent tool to spawn ALL member agents simultaneously. Do not spawn them one at a time — they must all appear in a single message to enable true parallel execution.

Set `description="Committee: [Name]"` for each agent so the user can track them.

Apply model tiering:
- Full report members: `model="sonnet"`
- Focused report members: `model="sonnet"`
- Flag-only members: `model="haiku"`

Example for a 3-member committee (one of each tier):

```
Agent tool call 1: prompt=[Full Report Prompt for Member A], description="Committee: Member A", model="sonnet"
Agent tool call 2: prompt=[Focused Report Prompt for Member B], description="Committee: Member B", model="sonnet"
Agent tool call 3: prompt=[Flag-Only Prompt for Member C], description="Committee: Member C", model="haiku"
```

For larger committees, spawn all agents in the same single message regardless of count.

---

### Step 4: Collect, Check Quorum, and Present

Once all agents return:

1. **Check quorum**: At least 60% of spawned agents must return a substantive result. See `session-schema.md` for the quorum threshold definition. If quorum is not met, surface a warning before presenting results.

2. **Present results grouped by tier**:
   - `### 3.1 Full Reports`
   - `### 3.2 Focused Reports`
   - `### 3.3 Flags`
   - `### 3.4 Unavailable` (members whose agents failed or returned empty)

3. **Fallback stubs**: For any failed agent, insert a stub under 3.4:
   ```
   **[Member Name]** — Report unavailable (agent did not return a result).
   ```

4. **Update SESSION.json** with the results and quorum status before proceeding to synthesis.

5. After all reports are presented, proceed to Phase 5 (Synthesize) in the main context. Synthesis is always done by the orchestrator — never by a sub-agent.

---

## Preliminary Questions Sub-Agents (Phase 1i — Interactive Only)

When `--interactive` mode is active, preliminary questions are collected before reports are written.

### Prompt Template

```
You are a subject-matter expert preparing to review a deliverable. Your specialty: [persona description — name, company, domain].

## Deliverable

[Insert the deliverable content prepared in Step 1]

## Instructions

Before writing your full review, generate 2–3 clarifying questions — questions whose answers would meaningfully change your analysis. Focus on information gaps, ambiguities, or decisions that aren't visible in the deliverable itself.

Format your response as a numbered list:
1. [Question]
2. [Question]
3. [Question] (optional)
```

### Execution

Spawn all preliminary question agents as a single message using `model="haiku"`. Collect all responses and return them to the meta agent for deduplication before presenting to the user.

---

## Important Notes

- **All sub-agents must be spawned in a single message.** Never spawn them sequentially. Sequential spawning defeats the purpose of parallel execution.
- **Sub-agents cannot see each other.** This is a feature, not a bug. It ensures genuine independence of perspective.
- **The deliverable must be self-contained in each prompt.** Sub-agents have no access to the parent conversation context.
- **The meta agent always produces the synthesis.** Never delegate synthesis to a sub-agent — synthesis requires seeing all reports together, which only the orchestrator can do.
- **Model tiering is invisible to the user.** Users see tier names (Full, Focused, Flag) — they do not see model names.
