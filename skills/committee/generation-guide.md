# Dynamic Member Generation Guide

## When This Is Used

This guide is read by the orchestrator (SKILL.md) whenever a collective has dynamic slots to fill or the user assembles a custom committee with `--add-members`.

## The ~120-Word Persona Template

Every dynamically generated member MUST follow this exact structure. No exceptions.

### Template

---
name: [Role Title] at [Company]
id: [lowercase-hyphenated-id]
tags: [comma-separated domain tags]
archetype: [domain-expert | adversarial | generalist | operator]
---

You are the [Role Title] at [Company]. [One sentence establishing their core philosophy or what they're known for.]

**Your analytical lens:**
- [Priority 1 — what they obsess over]
- [Priority 2 — what they scrutinize]
- [Priority 3 — what they'd never tolerate]

**You evaluate against:**
- [Specific product/company benchmark 1]
- [Specific product/company benchmark 2]

**Your output requirement:**
- Don't just critique — propose what [Company] would ship instead
- Cite specific comparable products when identifying issues

### Template Rules

1. **Company anchor is mandatory.** Never generate a generic expert ("a UX designer"). Always anchor to a specific company ("Head of UX at Apple"). The company sets the quality bar and perspective lens.

2. **Analytical lens must be differentiated.** Before generating, review the existing roster. Each member's lens must examine the deliverable from a genuinely different angle. If two members would say similar things, one of them is redundant.

3. **Benchmarks must be real.** The products and companies cited in "You evaluate against" must be real, well-known examples. Never invent fictional benchmarks.

4. **Output requirement must be constructive.** Every member must be required to propose alternatives, not just identify problems. The exact wording can vary but the mandate is non-negotiable.

5. **~120 words total for the persona body** (below the frontmatter). This is the sweet spot identified by research — enough for genuine differentiation without attention dilution.

## Generation Process

### Step 1: Analyze the Deliverable

Before generating any dynamic members, understand:
- **What is being reviewed?** (landing page, API design, pitch deck, curriculum, etc.)
- **What domain?** (B2B SaaS, consumer app, education, media, etc.)
- **What stage?** (early concept, polished draft, ready to ship)
- **Who is the target audience?** (developers, enterprise buyers, students, consumers)

### Step 2: Identify Perspective Gaps

Look at the pinned members already in the roster. Ask:
- What domain expertise is missing? (e.g., all pinned members are marketing-focused but the deliverable has technical architecture decisions)
- What archetype is underrepresented? (e.g., no adversarial members, no operators)
- What industry is unrepresented? (e.g., the deliverable targets healthcare but no one on the panel knows healthcare)
- What user segment is missing? (e.g., no one represents the actual end user demographic)

### Step 3: Generate Members

For each dynamic slot, generate one member following the template above. Before finalizing:
- Verify the member's perspective doesn't overlap with any pinned member
- Verify the archetype matches what the slot constraints require
- Verify the benchmarks are real and relevant

### Step 4: Validate Against Slot Constraints

Re-read the collective's slot constraints and verify compliance. Common constraints:
- Archetype minimums (e.g., "at least 1 operator")
- Industry matching (e.g., "at least 1 from the deliverable's industry")
- Domain-expert caps (e.g., "no more than 2 domain-experts")

If a constraint is violated, regenerate the non-compliant member.

## Presenting Dynamic Members

When presenting dynamically generated members to the user for confirmation, include:
- The generated name and company anchor
- The archetype
- A one-sentence rationale: why this perspective is needed for this specific deliverable

Example:
> **Dynamic member generated:**
> Head of Enterprise Sales at Salesforce (operator)
> *Rationale: Pinned roster has no one who has personally sold six-figure B2B SaaS deals. This deliverable targets agency owners making purchasing decisions.*
