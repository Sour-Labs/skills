---
name: handoff
description: Save the current session's working state to persistent memory so the user can start a fresh session and continue seamlessly. Use when the user wants to wrap up before hitting the context limit or restart in a clean session. Triggers on "save this to memory before I restart", "we're near the context limit", "save state so I can start fresh", "prepare a handoff", "save your progress to memory".
---

# Session Handoff

Write everything a fresh session would need to continue the current work into the
persistent memory directory (the path is in your system prompt). The next session only
gets MEMORY.md plus whatever files it recalls — assume zero conversation context survives.

## What to capture

Review the whole session and distill:

1. **Task and goal** — what we're working on and what "done" looks like.
2. **Current state** — completed and verified vs. in progress vs. not started. Be honest:
   if something was attempted and failed, say so.
3. **Next steps** — concrete and ordered. The first one should be executable immediately
   (exact command, file, or URL to start from).
4. **Decisions and constraints** — choices made and why, approaches explicitly ruled out,
   preferences the user expressed during this session.
5. **Gotchas** — non-obvious discoveries: quirks, env issues, workarounds, misleading
   errors that cost time this session.
6. **Pointers** — absolute file paths, branch names, PR/issue links, commands. Convert
   relative dates ("yesterday", "last week") to absolute dates.

Leave out anything the next session can re-derive from the code, git history, or
CLAUDE.md — record only the delta that lives in this conversation.

## How to save

1. Write ONE memory file per task — `handoff-<task-slug>.md`, `metadata.type: project`,
   following the standard memory format. If a handoff for this task already exists,
   update it in place rather than creating a second one.
2. Start the body with a line noting this is a session handoff and the date, so a future
   session knows it can delete the file once the task is finished.
3. If the session surfaced durable feedback or user preferences unrelated to this task,
   save those as separate `feedback`/`user` memories (don't bury them in the handoff).
4. Anything important that lives only in the session scratchpad will be lost — copy those
   files to a durable location and record the new path in the handoff.
5. Add/update the one-line pointer in MEMORY.md.

## Confirm

End your reply with:
- which files you wrote or updated,
- anything relevant you deliberately did not save and why,
- confirmation that the user can start a fresh session and simply ask to continue the
  task — the memory index will load automatically.
