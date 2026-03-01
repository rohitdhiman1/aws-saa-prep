You are wrapping up this study session. Perform the following steps in order, then exit.

---

## Step 1 — Review the session

Think through the full conversation: what was read, written, fixed, discussed, and decided. Identify:
- Files created or modified
- Concepts explained or clarified
- Knowledge gaps or corrections that surfaced
- Logical next task to pick up next session

---

## Step 2 — Update `docs/next-session.md`

Read the current file first. Then rewrite it with:
- **Target for:** the next logical session (e.g. "Phase X, Week Y — pick up from Z")
- **Up next:** 2–4 concrete, actionable bullet points (file + section + what to do)
- **Today's roadmap reference:** which roadmap topic this session covered
- **Open questions:** anything unresolved that needs answering next time (delete section if none)
- **Last session summary:** 3–6 bullet points summarising what was actually done this session (not what was planned)

Keep it tight — this file is read cold at the start of the next session.

---

## Step 3 — Update `docs/bugbase.md`

Read the current file. If any knowledge gaps, wrong assumptions, or corrections came up during this session, append entries using the existing format:

```
### [Service / Topic]
- **Gap:** what was wrong or unclear
- **Correct understanding:** the right answer
- **Source:** where clarified (AWS docs, this session, etc.)
- **Status:** open
```

If nothing new to log, leave the file unchanged.

---

## Step 4 — Update the project memory file

Read the current memory file at:
`/Users/rohitdhiman/.claude/projects/-Users-rohitdhiman-Documents-MyProjects-aws-saa-prep/memory/MEMORY.md`

Update it to reflect the current state of the project. Specifically:
- Update **Current State** to reflect newly created or modified files
- Update **What's Next** to match what you wrote in `docs/next-session.md`
- Add any new stable patterns, architectural decisions, or important file paths discovered
- Remove or correct anything that is now outdated or wrong
- Keep it under 200 lines — be ruthless about removing stale content

---

## Step 5 — Confirm and exit

Print a short summary of what was updated (one line per file). Then say:

> Session closed. Pick up from `docs/next-session.md` next time.
