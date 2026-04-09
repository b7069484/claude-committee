# Committee Review Output Format

## When This Is Used

This format is read by the meta agent during Phase 2 (Synthesis) and document assembly. Every committee review MUST follow this structure exactly.

**IMPORTANT: This is a SINGLE continuous output. Generate each section ONCE. Never repeat or restart any section.**

---

## Full Report Template

```
# Committee Review: [Deliverable Name]

**Committee:** [Collective Name or "Custom"] · **Members:** [N] ([N] reporting, [N] unavailable) · **Date:** [YYYY-MM-DD]
**Mode:** [Standard / Interactive] · **Report depth:** [default / quick / deep]

---

## 1. Executive Summary
[Meta agent's 200-word synthesis. Overall verdict, biggest opportunities, biggest risks.]

## 2. Committee Roster
Table with columns: #, Member, Company, Archetype, Tier, Status (✓ Complete / ⚠ Timed out)
Note if any unavailable: "Note: [N] members did not return reports ([names])."

## 3. Individual Reports

### 3.1 Full Reports
[For each full-report member:]
### [N]. [Member Name]
**Executive assessment:** [2-3 sentences]
**Key issues identified:** [2-4 issues with rationale/evidence, each boldface titled]
**Suggestions & opportunities:** [2-3 with benchmarks, each boldface titled]
**What I'd ship instead:** [1-2 paragraphs concrete alternative]

### 3.2 Focused Reports
[For each focused member:]
### [N]. [Member Name]
**Executive assessment:** [2-3 sentences]
**Top issues through this lens:** [2 issues with rationale]
**Key suggestion:** [One actionable suggestion with benchmark]

### 3.3 Flags
[For each flag-only member:]
### [N]. [Member Name]
FLAG: [issue] or "No [specialty] concerns identified."

### 3.4 Unavailable Members
[For each failed/timed out:]
### [N]. [Member Name]
**Status:** Report unavailable ([reason]). Missing from synthesis.

## 4. Debate Proceedings (Interactive Mode Only)
[Omit entirely if standard mode]
### Round 1: [Topic]
**Participants:** [members]
**Positions:** [member] [from report]: [summary]; [member] [extended]: [summary]
**User input:** [what user said or "None"]
**Resolution:** [Confirmed / Needs more / User decided / Deferred]

## 5. Synthesis
### Consensus Points
[Qualitative: "all agree", "strong majority", "split"]
- **[Point]** — [N]/[total] flagged. [Summary with agreeing members.]

### Key Tensions
**Tension [N]: [Topic]**
- *[Member A]* argues: [2-3 sentences with evidence]
- *[Member B]* counters: [2-3 sentences]
- *[Member A] responds:* [rebuttal]
- *[Member B] responds:* [rebuttal]
- **Committee note:** [resolved or genuine tradeoff?]
[--quick: skip rebuttals. --deep: 2 rebuttals + others weigh in. Interactive: incorporate debate findings.]

### Evidence & Benchmarks
[Consolidated, deduplicated list of real-world data cited across reports]

## 6. Blind Spots & Recommended Additions
[What committee can't evaluate, specific recommendations for who to add]

## 7. Next Steps
[Read next-steps-format.md. Produce both Path A and Path B.]
```

---

## Format Rules

1. **Rationale is mandatory.** Every issue and suggestion must be backed by real-world evidence, benchmarks, or case studies. Stating something without rationale makes the output malformed.

2. **Members speak independently.** During individual reports, members do NOT reference each other's analyses. They write as if they are the only reviewer. Cross-references happen only in the Synthesis section.

3. **Debate is genuine.** Tensions must reflect real disagreements, not manufactured drama. If all members agree on everything, there are no tensions — say so. Don't force tensions that don't exist.

4. **Blind spots must be honest.** Every committee has limits. Acknowledging them prevents false confidence and helps the user know where to seek additional input.

5. **Company anchoring shows in the voice.** Each member's report should feel like it comes from someone at that company — referencing that company's products, standards, and philosophy naturally, as a genuine analytical lens, not a gimmick.

6. **Missing members must be disclosed.** Any member who did not return a report must appear in the roster (with ⚠ status), in section 3.4, and must be noted as absent in the Synthesis section.

7. **Transparency tags are mandatory in debates.** Every position statement in Debate Proceedings must be tagged `[from report]` (position pulled directly from written report) or `[extended]` (position developed live during debate).
