# Session Checkpoint Schema

## When This Is Used

This file is read by the orchestrator at every phase transition. Before advancing to the next phase, the orchestrator writes the current SESSION.json to disk. On startup, the orchestrator checks for an existing SESSION.json and offers to resume or start fresh. This enables interrupted sessions to recover without re-running completed phases.

---

## File Location

SESSION.json lives alongside the review output file:

```
./committee-session-YYYY-MM-DD-[short-topic].json
```

Example: `./committee-session-2026-04-09-ai-regulation.json`

The short topic is derived from the deliverable title (lowercase, hyphens, max 30 chars).

---

## Schema (v2.0)

```json
{
  "version": "2.0",
  "session_id": "<uuid-or-timestamp-string>",
  "created_at": "<ISO-8601>",
  "updated_at": "<ISO-8601>",
  "phase_cursor": "<enum>",
  "mode": "standard | interactive",
  "deliverable_summary": "<one-sentence description of what is being reviewed>",

  "roster": {
    "members": [
      {
        "id": "<member-id>",
        "name": "<display name>",
        "tier": "full | focused | flag",
        "model": "sonnet | haiku",
        "archetype": "<e.g. Devil's Advocate, Domain Expert, Ethicist>",
        "source": "pinned | dynamic",
        "status": "pending | report_complete | timed_out | failed | retried"
      }
    ],
    "blind_spots_offered": ["<archetype or angle suggested but not selected>"],
    "blind_spots_accepted": ["<archetype or angle accepted and added to roster>"]
  },

  "preliminary_answers": {
    "<question text>": "<answer text>"
  },

  "agenda": [
    {
      "topic": "<topic string>",
      "priority": "high | medium | low",
      "status": "pending | debated | confirmed | skipped"
    }
  ],

  "reports": {
    "<member-id>": {
      "status": "pending | complete | timed_out | failed | stub",
      "summary": "<~50 word summary of this member's position>",
      "full_text_ref": "<inline or path reference to full report text>"
    }
  },

  "debate_rounds": [
    {
      "round": 1,
      "topic": "<topic debated>",
      "participants": ["<member-id>", "<member-id>"],
      "positions_summary": "<brief summary of positions taken>",
      "user_input": "<user's steering input, if any>",
      "resolution": "<agreed resolution or note of ongoing contention>"
    }
  ],

  "decisions": {
    "confirmed": ["<decision string>"],
    "open": ["<unresolved question string>"],
    "contentious": ["<point of ongoing disagreement>"]
  },

  "cost_estimate": {
    "sub_agent_tokens": 0,
    "main_context_tokens": 0
  }
}
```

### `phase_cursor` Enum Values

| Value | Meaning |
|---|---|
| `composition_complete` | Roster finalized, blind spot check done |
| `questions_complete` | Preliminary questions answered |
| `agenda_complete` | Agenda topics confirmed |
| `reports_complete` | All member reports received (or timed out) |
| `briefing_complete` | Cross-briefing delivered to members |
| `debate_round_N` | Debate round N finished (replace N with integer) |
| `synthesis_complete` | Synthesis document written |
| `bridge_complete` | Bridge section and final output written |

---

## Checkpoint Protocol

### When to Write

Write SESSION.json after each of these 8 phase events:

1. Roster finalized (blind spot check complete) → set cursor to `composition_complete`
2. Preliminary questions answered → set cursor to `questions_complete`
3. Agenda confirmed → set cursor to `agenda_complete`
4. All member reports received or timed out → set cursor to `reports_complete`
5. Cross-briefing delivered → set cursor to `briefing_complete`
6. Each debate round completes → set cursor to `debate_round_N`
7. Synthesis written → set cursor to `synthesis_complete`
8. Bridge and final output written → set cursor to `bridge_complete`

### How to Write

Use the Write tool to overwrite the SESSION.json file at each checkpoint:

- Update `updated_at` to current ISO-8601 timestamp
- Update `phase_cursor` to the completed phase value
- Update all relevant sub-fields (roster statuses, report entries, debate rounds, decisions)
- Do not change `created_at` or `session_id` after initial write

### How to Read / Resume

On startup, check for an existing SESSION.json matching the pattern `./committee-session-*.json` in the working directory.

If found, present the user with:

```
Found existing session: [session_id]
Topic: [deliverable_summary]
Last checkpoint: [phase_cursor]
Created: [created_at]

Resume from last checkpoint, or start a fresh session?
  R = Resume
  F = Fresh start
```

If the user chooses Resume, advance to the phase immediately following `phase_cursor` and skip all completed phases. Roster, reports, and prior decisions carry over.

If the user chooses Fresh start, archive the old file by appending `.archived` to its name and begin a new session.

---

## Failure Recovery

### Sub-Agent Timeout

Sub-agents that do not return within **120 seconds** are marked `timed_out`. After timeout:

1. Log the timeout in the member's `reports` entry with `status: "timed_out"`
2. Attempt one automatic retry — mark `status: "retried"` during retry
3. If retry also times out, mark `status: "failed"` and generate a fallback stub (see below)

### Quorum Enforcement

Minimum quorum is **60%** of the full roster (round up).

| Roster size | Quorum required |
|---|---|
| 10 members | 6 |
| 8 members | 5 |
| 5 members | 3 |
| 4 members | 3 |
| 3 members | 2 |

Check quorum after all reports are received (or timed out). If quorum is not met, pause and present the user with:

```
Quorum not met: [N] of [total] members responded (need [quorum]).
Failed members: [list names]

How would you like to proceed?
  1 = Retry failed members (up to 2 more attempts)
  2 = Proceed with available reports (below quorum — noted in synthesis)
  3 = Abort session
```

If the user selects option 2, record the below-quorum status in `decisions.open` and add a mandatory disclosure note to the synthesis.

### Fallback Stubs

For each failed member, generate a stub entry in `reports`:

```json
{
  "status": "stub",
  "summary": "[Member name] did not respond. Perspective not available for this review.",
  "full_text_ref": null
}
```

The stub is included in the cross-briefing so other members know the perspective is missing. The synthesis must explicitly note which members failed and that their views are absent.

### Mandatory Disclosure in Synthesis

If any member has `status: "failed"` or `status: "stub"`, the synthesis output must open with:

> **Note:** This review is based on [N] of [total] committee members. The following perspectives were unavailable due to timeout or failure: [names]. Conclusions may be incomplete in areas covered by those archetypes.

---

## Cost Estimation

Present a cost estimate to the user during Phase 0 (composition), before any sub-agents run.

### Estimated Tokens Per Member Tier

| Tier | Input tokens (est.) | Output tokens (est.) |
|---|---|---|
| Full member | ~1,000 | ~1,500 |
| Focused member | ~1,000 | ~800 |
| Flag member | ~800 | ~200 |
| Preliminary Q&A | ~800 | ~200 |

These are per-member estimates. Multiply by roster size and sum across tiers for a session total.

Store running totals in `cost_estimate.sub_agent_tokens` and `cost_estimate.main_context_tokens`. Update after each phase completes.

Present the pre-run estimate in Phase 0 as:

```
Estimated token usage for this session:
  Sub-agent calls: ~[N] tokens
  Main context: ~[N] tokens
  Total: ~[N] tokens

Proceed?
```
