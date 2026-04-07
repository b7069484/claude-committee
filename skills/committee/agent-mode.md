# Agent Mode — Parallel Committee Execution

## When This Is Used

This file is read by the orchestrator when the `--parallel` flag is present. It replaces the default sequential execution with parallel subagent spawning.

## Why Agent Mode Exists

In default mode, all members "speak" within the same context window. This means:
- Later members may be subtly influenced by earlier members' reports
- Perspectives can bleed together
- The committee doesn't produce truly independent analyses

Agent mode solves this by spawning each member as a separate subagent. Each subagent has its own context and literally cannot see the other members' work.

## Execution Instructions

### Step 1: Prepare the Deliverable Content

Before spawning agents, the orchestrator must prepare the deliverable as a self-contained text block:

- **If the deliverable is files** (HTML, code, documents): Read each file using the Read tool. Include the full content in the agent prompt.
- **If the deliverable is a concept from conversation**: Write a comprehensive summary (500-1000 words) capturing: what it is, who it's for, what problem it solves, current state/stage, key decisions already made.
- **If mixed**: Include both file contents and a context summary.

### Step 2: Construct the Agent Prompt

For each committee member, construct this prompt:

---

You are conducting an independent expert review. You are the ONLY reviewer — do not reference or anticipate other reviewers.

## Your Persona

[Insert the full ~120-word persona definition here — either from the favorites file or from dynamic generation]

## Deliverable to Review

[Insert the deliverable content prepared in Step 1]

[If --focus flag is set: "FOCUS: Concentrate your analysis specifically on: [focus topic]. You may note other issues briefly but your primary analysis should address the focused topic."]

## Report Format

Produce your review in exactly this structure:

**Executive assessment:** [2-3 sentences — your overall verdict]

**Key issues identified:**
- [Issue] — [Detailed rationale with evidence, case studies, or benchmarks]
[Aim for [2-4 issues in default | 1-2 in --quick | 4-6 in --deep] issues]

**Suggestions & opportunities:**
- [Suggestion] — [Why it works, with specific benchmark]
[Aim for [2-3 in default | 1-2 in --quick | 3-5 in --deep] suggestions]

**What I'd ship instead:** [Concrete alternative — what would your team at [Company] actually build?]
[Length: [1-2 paragraphs default | 2-3 sentences --quick | detailed counter-proposal --deep]]

## Rules

- Every issue and suggestion MUST include rationale backed by real-world evidence
- Reference your company's products and standards naturally
- Be specific and actionable — no vague advice
- Propose alternatives, don't just critique

---

### Step 3: Spawn Agents in Parallel

Use the Agent tool to spawn ALL member agents simultaneously in a single message. This is critical — spawning them in one message enables true parallel execution.

Example (for a 3-member committee):

Use Agent tool three times in a single response:
- Agent 1: prompt=[Member 1's full prompt from Step 2], description="Committee: [Member 1 name]"
- Agent 2: prompt=[Member 2's full prompt from Step 2], description="Committee: [Member 2 name]"
- Agent 3: prompt=[Member 3's full prompt from Step 2], description="Committee: [Member 3 name]"

For committees of 8-10 members, spawn all 8-10 agents in a single message.

### Step 4: Collect and Present

Once all agents return their reports:

1. Present each report under a header: `### [Number]. [Member Name]`
2. After all individual reports are presented, proceed to Phase 5 (Synthesize) in the main context
3. The synthesis is always done by the orchestrator (main context), NOT by a subagent — because synthesis requires seeing all reports together

### Important Notes

- **Never spawn agents sequentially.** The whole point of agent mode is parallel execution. All agents must be spawned in a single message.
- **Subagents cannot see each other.** This is a feature, not a bug. It ensures genuine independence.
- **The deliverable must be self-contained** in each agent's prompt. Subagents cannot access the parent conversation context.
- **Agent mode costs more tokens.** Each member gets its own full context. For a 10-member committee, this means roughly 10x the token usage of default mode. This is expected and acceptable when the user explicitly opts into `--parallel`.
