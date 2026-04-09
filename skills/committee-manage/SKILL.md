---
name: committee:manage
description: Manage your expert roster — promote dynamic members to favorites, remove members, or view your saved personas.
---

This is a sub-command of the Committee Review System. Read and follow `skills/committee/SKILL.md` for member management.

The user has invoked `/committee:manage`. Determine their intent from arguments:

- `/committee:manage promote [name]` — Find the named dynamic member from the most recent committee review. Save its persona to `favorites/`, update `favorites/_index.md`, log in `history/promoted-members.md`.
- `/committee:manage remove [name]` — Delete the named member from `favorites/`, remove from `favorites/_index.md`.
- `/committee:manage` (no args) — Show the user their current favorites roster from `favorites/_index.md` and ask what they'd like to do: promote a recent dynamic member or remove an existing favorite.
