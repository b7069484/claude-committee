# Debate Format — Interactive Mode

## When This Is Used

Read during Phase 2d (interactive mode only). This format governs how the meta agent
facilitates topic-based debate rounds between committee members after their independent
reports have been submitted. It does not apply in standard (non-interactive) mode.

---

## Round Structure

Each debate topic follows five steps.

### Step 1: Topic Introduction

The meta agent opens each round by naming the topic, its priority level from the agenda,
and where it appeared in the Phase 1 reports. This anchors the debate in already-completed
analysis rather than speculation.

Example: "Round [N] — Topic: [X] (Priority: High | Agenda item 3). Three members addressed
this directly. One dissenting view was filed."

### Step 2: Member Selection

Select 3–4 members per round based on these criteria:
- Their Phase 1 report directly addressed this topic
- Their analytical lens maps to the domain in question
- At least one dissenter is included if one exists

Do not include members whose reports are tangential. Fewer focused voices beat broad coverage.

### Step 3: Voicing Members

The meta agent voices each selected member in sequence. Three guardrails apply:

**Grounding rule:** Every position must trace to that member's Phase 1 report. The meta
agent does not invent new positions. If the report is silent on a subtopic, the member
is silent.

**Steelman rule:** When voicing dissent, present the strongest version. Do not soften,
hedge, or qualify to avoid friction. State the position directly and confidently with
evidence from the report. Let it stand before the next speaker responds. Actively
resist any pull toward RLHF consensus-seeking — the value of debate is in the tension.

**Transparency tags:** Each voiced position carries a tag:
- `[from report]` — directly from the member's independent Phase 1 report
- `[extended]` — extrapolated from their analytical lens, not explicit in report

Format for each voice:

```
**[Member Name]** [from report]: "[Their position, 2–4 sentences.]"

**[Member Name]** [extended]: "[Their position, 2–4 sentences.]"
```

### Step 4: Round Synthesis

After all selected members have spoken, the meta agent delivers a synthesis paragraph:
- Where members agree and on what basis
- Where they diverge and what drives the divergence
- What is materially at stake in the disagreement
- What the user would need to decide or believe to side with each camp

This is analytical summary, not resolution. The meta agent does not pick a winner.

### Step 5: Checkpoint

The meta agent MUST pause here and present numbered options:

```
Where do you want to take this?
1. Hear them debate further
2. Add your constraints or context
3. You've decided — move on
4. Add a [relevant specialist] to weigh in
5. Switch to standard mode — wrap up with synthesis
```

**Option 1 — Debate further:** Re-enter Step 3 with the same members responding to each
other's positions. Maximum 2 additional rounds per topic before the meta agent forces a
checkpoint regardless.

**Option 2 — Add constraints or context:** User input is recorded in SESSION.json under
`user_context` for this topic. The meta agent acknowledges it explicitly, then either
continues the round or closes the topic based on whether the new context resolves the
tension.

**Option 3 — Move on:** Topic is marked resolved with the user's decision logged.
Proceed to the next topic on the agenda.

**Option 4 — Add a specialist:** The meta agent generates a dynamic member appropriate
to the topic gap, then spawns a Sonnet sub-agent to produce a focused report on this
specific topic. The new member is introduced, voiced once with their report findings,
and included in a final synthesis before checkpoint repeats.

**Option 5 — Switch to standard mode:** Exit all remaining debates. The meta agent
produces a full synthesis document covering all completed and incomplete topics, using
the compressed running summary as the base. No further debate rounds are run.

---

## Consensus Fast-Track

Before beginning a full round, check for strong agreement across relevant members.

If all relevant members agree:

```
Topic: [X]. All [N] relevant members agree: [one-sentence summary of consensus].
No dissent on record.
Confirm and move on? (Y/n)
```

If the user confirms, log the consensus in SESSION.json and move to the next topic.
Skip Steps 2–5. Only run a full round if (a) the user declines the fast-track, or
(b) meaningful dissent exists.

---

## Compressed Running Summary

Between rounds, the meta agent maintains a ~200-word running summary. This summary
captures:
- Topics covered and how they resolved (user decided, consensus reached, tension held)
- User decisions and any constraints or context added mid-debate
- Outstanding tensions not yet resolved
- Any dynamic members added and what they contributed

This summary — not the full debate transcript — carries forward as context between
rounds and into the final synthesis phase. It is stored in SESSION.json under
`running_summary` and updated after each completed round.

---

## Exit Conditions

Debates end when any of the following occur:

- (a) All agenda topics have been covered
- (b) User says "wrap it up" or equivalent
- (c) User selects Option 5 at any checkpoint
- (d) Maximum of 8 rounds has been reached

On exit, the meta agent proceeds directly to final synthesis using the compressed
running summary and any user context logged during the session.

---

## What Goes in the Final Document

The debate proceedings section of the output document follows this format per round:

```
### Round [N]: [Topic Name]

**Participants:** [Member A], [Member B], [Member C]

**Positions:**
- **[Member A]** [from report]: "[Position]"
- **[Member B]** [from report]: "[Position]"
- **[Member C]** [extended]: "[Position]"

**User Input:** [Any constraints or context added, or "None"]

**Resolution:** [How this topic closed — user decision, consensus, or held tension]
```

Rounds that were fast-tracked via consensus appear as:

```
### Round [N]: [Topic Name] (Consensus)

**Summary:** [One-sentence consensus statement]
**Resolution:** Confirmed by user.
```
