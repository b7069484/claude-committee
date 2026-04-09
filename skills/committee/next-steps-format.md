# Next Steps Format

## When This Is Used

Read during **Phase 2 synthesis** (when the meta agent consolidates member positions) and **Phase 3 implementation bridge** (when handing off approved items to writing-plans).

---

## Two-Path Output

### Path A: Full Adoption

All recommendations rendered as an ordered action list. The user wants to implement everything; the output is a ready-to-execute plan.

**Each item includes:**
- Action description with source members
- Priority: High / Medium / Low
- Consensus: Unanimous / Strong majority / Majority (see classification table)
- Complexity: Low / Medium / High
- Depends on: none, or item #N
- If contentious: `⚠ WARNING` flag naming the dissenting member and referencing the relevant Tension #N

**Ordering rules:** Primary sort by priority (High first). Secondary sort by consensus strength (Unanimous > Strong majority > Majority). Dependencies always respected — a blocked item appears after its dependency regardless of priority.

**Template:**

```markdown
## Next Steps — Full Adoption

1. **[Action description]**
   - Source: [Member A], [Member B]
   - Priority: High
   - Consensus: Unanimous
   - Complexity: Low
   - Depends on: none

2. **[Action description]**
   - Source: [Member C]
   - Priority: High
   - Consensus: Strong majority
   - Complexity: Medium
   - Depends on: none
   - ⚠ WARNING: [Member D] dissents — see Tension #1

3. **[Action description]**
   - Source: [Member A], [Member C]
   - Priority: Medium
   - Consensus: Unanimous
   - Complexity: High
   - Depends on: item #2
```

---

### Path B: Selective Adoption

Tensions are surfaced prominently — not buried. The user makes explicit decisions before implementation begins.

**Section 1 — Ready to Act (Full Consensus):**

Numbered list of unanimous items, formatted identically to Path A items (no warnings needed).

**Section 2 — Requires Your Decision (Key Tensions):**

Each tension is fully unpacked:
- Position A: statement — members who hold it — supporting evidence
- Position B: statement — members who hold it — supporting evidence
- Meta agent assessment: which conditions favor A, which favor B
- Specific implications of choosing A vs B
- `Your decision: ___`

**Template:**

```markdown
## Next Steps — Selective Adoption

### Ready to Act — Full Consensus:

1. **[Action description]**
   - Source: [Member A], [Member B]
   - Priority: High | Consensus: Unanimous | Complexity: Low
   - Depends on: none

### Requires Your Decision — Key Tensions:

#### Tension #1: [Short title]

**Position A:** [Recommendation statement]
- Members: [Member A], [Member B]
- Evidence: [Key supporting point]

**Position B:** [Recommendation statement]
- Members: [Member C]
- Evidence: [Key supporting point]

**Meta agent assessment:**
- Favor A when: [conditions — e.g., speed is the priority, team is small]
- Favor B when: [conditions — e.g., long-term maintainability matters, scale is expected]

**Implications:**
- Choose A → [specific downstream consequence]
- Choose B → [specific downstream consequence]

Your decision: ___
```

---

## Consensus Classification Table

| Condition | Label |
|---|---|
| All members agree | **Unanimous** (or "all agree") |
| 70%+ agree, no strong dissent | **Strong majority** |
| 50–70% agree, active disagreement present | **Majority** — note dissent |
| Near-even split, strong arguments on both sides | **Split** — escalate to Tensions |

**Never use percentage numbers in output.** The committee is too small for meaningful percentages. Use the qualitative labels only.

---

## Implementation Bridge Handoff Contract

The handoff is **not written to a file**. It is passed as inline context when invoking the writing-plans skill.

### Handoff Structure

A plain text block passed to writing-plans containing:

1. **Context header** — paths to the session file and review output file
2. **Approved items** — numbered list, each with:
   - Action
   - Priority
   - Consensus
   - Complexity
   - Source members
   - Implementation hint (one sentence)
   - Dependencies

### Dry-Run Preview

Before invoking writing-plans, display a preview:

```
Implementation Bridge — Dry Run Preview

INCLUDED:
  1. [Action] — Unanimous, High priority
  2. [Action] — Strong majority, High priority
  3. [Action] — Majority, Medium priority (majority position adopted)

NOT INCLUDED (contentious, not yet approved):
  - [Action] — Split (Tension #1, awaiting your decision)
  - [Action] — Split (Tension #2, awaiting your decision)

Proceed? (Y/n)
```

### What Gets Included

- **Path A:** All recommendations. For contentious items, the majority position is used; the dissent is noted in the implementation hint.
- **Path B:** Unanimous items plus any contentious items the user explicitly approved in the Tensions section. Not-approved contentious items appear in the "NOT INCLUDED" list.

### After writing-plans Completes

Offer to run executing-plans:

> "Plan written. Run executing-plans to begin implementation? (Y/n)"
