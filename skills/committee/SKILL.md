---
name: committee
description: Multi-perspective committee reviews with curated expert personas and dynamic member generation. Use when the user wants expert panel critique, review, or analysis of any deliverable.
---

# Committee Review System

Assemble and run expert committee reviews on any deliverable. Committees consist of curated favorite members and dynamically generated experts who each produce independent analytical reports, followed by a synthesized report with consensus, debate, and recommendations.

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
| `--parallel` | Execute in agent mode — read `agent-mode.md` for instructions |
| `--quick` | Set report length to ~150 words per member |
| `--deep` | Set report length to ~600+ words per member, more evidence, more debate rounds in synthesis |
| `--focus "[topic]"` | Instruct all members to concentrate their analysis on the quoted topic |
| `--add-members N` | Add N extra dynamic slots beyond the collective's default count |

## Orchestration Flow

### Phase 1: ROUTE

Parse the user's input to determine intent (review, suggest, list, add, promote, remove, custom). If ambiguous, default to `review` and ask which collective.

### Phase 2: ASSEMBLE

1. **If a collective was specified:** Read the collective file from `collectives/[id].md`
2. **If "suggest" was used:** Analyze the current conversation to identify what the user has been building/working on. Read `collectives/_index.md` to find the best-matching collective. Read its file.
3. **If "custom" was used:** Parse the listed member IDs. Set dynamic slot count from `--add-members` flag (default 0 if not specified).
4. **Resolve pinned members:** For each pinned member ID in the collective, read its file from `favorites/[id].md`
5. **Fill dynamic slots:** Read `generation-guide.md`. For each dynamic slot:
   - Analyze the deliverable (what is it, what domain, what stage, target audience)
   - Identify what perspectives are missing from the pinned roster
   - Generate a member following the ~120-word template in `generation-guide.md`
   - Validate against the collective's slot constraints (archetype balance, industry match, no overlap)
6. **Present the full roster** to the user showing: member name, company anchor, archetype, and a one-line description for each. Clearly label which are pinned favorites and which are dynamically generated.

### Phase 3: CONFIRM

Wait for the user to confirm the roster. They may:
- Say "run it" or equivalent → proceed to Execute
- Say "run it --parallel" → proceed to Execute in agent mode
- Request swaps → adjust the roster, re-present, wait for confirmation
- Request additions → add more dynamic slots, generate, re-present

### Phase 4 + 5: EXECUTE AND SYNTHESIZE (single continuous output)

Read `review-format.md` for the output template. **Phases 4 and 5 are ONE continuous output. Do NOT restart, repeat, or regenerate any section. Once an individual report is written, move to the next member. Once all members are done, move directly to the Synthesized Report. Never go back.**

**Default mode (no --parallel flag):**

For each member in the roster, in order:
1. Adopt the member's persona (use the full persona definition from their file or from dynamic generation)
2. Analyze the deliverable through that persona's specific lens
3. Produce the individual report following the format in `review-format.md`
4. Present the individual report under a header with the member's name

Important: Between members, consciously reset your perspective. Each member must analyze independently — do not reference or build on previous members' reports during individual report generation.

**Parallel mode (--parallel flag):**

Read `agent-mode.md` for detailed instructions on spawning subagents.

**After ALL individual reports are complete, proceed DIRECTLY to the Synthesized Report.** Do NOT re-read review-format.md or restart the output. Produce the synthesis as a continuation of the same output:

1. **Consensus Points:** Identify issues/observations where 60%+ of members converge. Count agreement and note it as [N]/[total].
2. **Key Tensions:** Find areas where members meaningfully disagree. For each tension:
   - State Member A's position with rationale
   - State Member B's counter-position with rationale
   - Give Member A one rebuttal turn
   - Give Member B one rebuttal turn
   - Add a Committee Note: is this resolved or a genuine tradeoff the user must decide?
3. **Top Recommendations:** Rank by (consensus strength × projected impact). Include agreement count, expected impact, and complexity rating (low/medium/high).
4. **Blind Spots Acknowledged:** What this committee is NOT qualified to evaluate. What perspectives are missing. What the user should seek input on elsewhere.

### Phase 6: SAVE REPORT

After the synthesis is complete, save the ENTIRE review (individual reports + synthesized report) as a markdown file in the current project directory:

1. **File path:** `./committee-review-YYYY-MM-DD-[short-topic].md` in the current working directory. Derive `[short-topic]` from the deliverable (e.g., `committee-review-2026-04-07-landing-page.md`).
2. **Use the Write tool** to save the full report content.
3. **Confirm to the user:** "Report saved to `[file path]`"

### Phase 7: FOLLOW-UP

After the review is complete, the user may:
- Ask a specific member to elaborate on a point → re-adopt that persona and expand
- Say "promote the [name] member" → save dynamic member to favorites
- Ask for a re-review after making changes → re-run with the same roster
- Invoke a different collective for a different angle on the same deliverable

## Report Length by Mode

| Mode | Individual Report Length | Synthesis Detail |
|------|------------------------|-----------------|
| `--quick` | ~150 words per member | Consensus + top 3 recommendations only |
| default | ~300-500 words per member | Full synthesis with tensions and debate |
| `--deep` | ~600+ words per member | Extended synthesis, more debate rounds, additional evidence |

## Deliverable Context

The deliverable is whatever the user has been working on in the current conversation. You have full conversation context — do NOT ask the user to re-describe what they've built. If the deliverable includes files (HTML, code, documents, images), read them using the Read tool before starting the review.

If the user invokes `/committee` at the start of a conversation with no prior context, ask: "What would you like the committee to review? You can describe it, paste it, or point me to files."
