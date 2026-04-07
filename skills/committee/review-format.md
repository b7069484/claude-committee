# Committee Review Output Format

## When This Is Used

This format is read by the orchestrator during Phase 4 (Execute) and Phase 5 (Synthesize). Every committee review MUST follow this structure exactly.

## IMPORTANT: This is a SINGLE continuous output. Generate each section ONCE. Never repeat or restart any section. After completing all individual reports, proceed directly to the Synthesized Report.

## Full Report Template

# Committee Review: [Deliverable Name]
[N] members · [Collective Name or "Custom"] · [Date]

---

## Individual Reports

### [Number]. [Member Name]

**Executive assessment:** [2-3 sentences — their overall verdict on the deliverable, framed through their specific expertise lens. This should immediately tell the reader where this expert stands.]

**Key issues identified:**
- [Issue title] — [Detailed rationale: WHY this is a problem. Must include at least one of: real-world evidence, case study, data point, or industry precedent. Never just "this is bad" — always "this is bad because X happened when Y company did the same thing" or "this violates Z principle which is critical because..."]
- [Additional issues following the same format. Aim for 2-4 issues per member in default mode, 1-2 in --quick mode, 4-6 in --deep mode.]

**Suggestions & opportunities:**
- [Suggestion] — [Why it works, with specific benchmark. E.g., "Stripe does X, which increased Y by Z%" or "Apple's approach to this in [product] demonstrates that..."]
- [Additional suggestions. Aim for 2-3 in default mode, 1-2 in --quick mode, 3-5 in --deep mode.]

**What I'd ship instead:** [1-2 paragraphs in default mode. This is NOT abstract advice. This is a concrete alternative — "Here's what this would look like if my team at [Company] built it." Describe the specific changes, structure, or approach they would take. In --quick mode, reduce to 2-3 sentences. In --deep mode, expand to a detailed counter-proposal.]

---

[Repeat for all N members]

---

## Synthesized Report

### Consensus Points
[Identify issues or observations where 60%+ of members converge on the same concern or opportunity.]

- **[Consensus point]** — [N]/[total] members flagged this. [One-sentence summary of the shared concern, noting which members agree.]
- [Additional consensus points, ordered by agreement strength (highest first).]

### Key Tensions

[Identify areas where members meaningfully disagree — not minor wording differences, but genuine strategic or philosophical disagreements.]

**Tension [N]: [Topic Name]**
- *[Member A name]* argues: [Their position in 2-3 sentences, including their rationale and evidence]
- *[Member B name]* counters: [Their opposing position in 2-3 sentences, including their rationale and evidence]
- *[Member A name] responds:* [One rebuttal turn — they can defend their position, concede partially, or refine their argument. 2-3 sentences.]
- *[Member B name] responds:* [One rebuttal turn — same rules. 2-3 sentences.]
- **Committee note:** [Is this tension resolved by the debate? Or is it a genuine tradeoff where the user must make a judgment call? If a tradeoff, briefly describe what each side optimizes for.]

[In --quick mode, skip the rebuttal turns — just present the two positions and the committee note.]
[In --deep mode, allow up to 2 rebuttal turns per member and invite other members to weigh in.]

### Top Recommendations
[Ranked by: consensus strength × projected impact. These are the committee's prioritized action items.]

1. **[Specific action]** — [N]/[total] agreement · [Expected impact in concrete terms] · Complexity: [low/medium/high]
2. [Continue ranking. Aim for 3-5 in default mode, top 3 in --quick mode, 5-8 in --deep mode.]

### Blind Spots Acknowledged
[What this committee is NOT qualified to evaluate. Be specific and honest.]

- [Perspective or domain not represented on this panel]
- [Type of analysis that would require different expertise]
- [Recommendation for what the user should seek input on elsewhere and from whom]

## Format Rules

1. **Rationale is mandatory.** No member may identify an issue, make a suggestion, or state an assessment without explaining WHY and backing it up with real-world evidence, benchmarks, or case studies. If a member states something without rationale, the output is malformed.

2. **Each member speaks independently.** During individual reports, members do NOT reference each other's analyses. They write as if they are the only reviewer. Cross-references happen only in the Synthesized Report.

3. **The debate is genuine.** Tensions must reflect real disagreements, not manufactured drama. If all members agree on everything, there are no tensions — say so. Don't force tensions that don't exist.

4. **Blind spots must be honest.** Every committee has limits. Acknowledging them prevents false confidence and helps the user know where to seek additional input.

5. **Company anchoring shows in the voice.** Each member's report should feel like it comes from someone at that company — referencing that company's products, standards, and philosophy naturally. Not as a gimmick, but as a genuine analytical lens.
