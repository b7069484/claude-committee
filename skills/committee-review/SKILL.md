---
name: committee:review
description: Start an expert committee review of any deliverable. Choose Standard (fast, automated) or Interactive (step-by-step with debates). Pass a collective name or let the EA suggest one.
---

This is a sub-command of the Committee Review System. Read and follow `skills/committee/SKILL.md` with intent: **review**.

The user has invoked `/committee:review`. Parse any arguments after the command as the collective ID and/or deliverable reference. Then execute the orchestration flow starting at Phase 0 (Intake & Mode Selection).

If arguments include a file path or description, that is the deliverable to review.
If arguments include a collective name (e.g., `tech-product-review`), use that collective.
If no collective specified, use the `suggest` logic to auto-recommend one.
