---
name: "consolidate-memory"
description: "Reflective pass over your memory files — merge duplicates, fix stale facts, prune the index. Use when the user says \"consolidate memory\", \"clean up memory\", or \"tidy your notes\", or when the memory index exceeds 25KB / 200 lines, or after a long session with many new facts learned."
---

# Memory Consolidation

You're doing a reflective pass over what you've learned about this user and their work. The goal: a future session should be able to orient quickly — who they work with, what they're focused on, how they like things done — without re-asking.

Your system prompt's auto-memory section defines the directory, file format, and memory types. Follow it.

## When to run

- User explicitly asks: "consolidate memory", "clean up your notes", "tidy memory"
- The memory index (`MEMORY.md`) exceeds 200 lines or ~25KB
- After a long session where many new facts, people, or projects were introduced
- Do **not** run if fewer than 5 memory files exist — there's nothing to consolidate yet

## Phase 1 — Take stock

- List the memory directory and read the index (`MEMORY.md`)
- Skim each topic file. Note which ones overlap, which look stale, which are thin.

## Phase 2 — Consolidate

**Separate the durable from the dated.** Preferences, working style, key relationships, and recurring workflows are durable — keep and sharpen them. Specific projects, deadlines, and one-off tasks are dated — if the date has passed or the work is done, retire the file or fold the lasting takeaway (e.g. "user prefers X format for launch docs") into a durable one.

**Merge overlaps.** If two files describe the same person, project, or preference, combine into one and keep the richer file's path.

**Fix time references.** Convert "next week", "this quarter", "by Friday" to absolute dates so they stay readable later.

**Drop what's easy to re-find.** If a memory just restates something you could pull from the user's calendar, docs, or connected tools on demand, cut it. Keep what's hard to re-derive: stated preferences, context behind a decision, who to go to for what.

## Guardrails

- Do not delete a memory file without first checking whether its content is still active or useful
- Do not merge files from different domains (e.g. people + projects) even if they seem related — keep them separate for clarity
- If unsure whether a fact is stale, keep it and add a note like `(unverified since YYYY-MM-DD)`
- Never surface or repeat sensitive information (passwords, private details) — skip those entries

## Phase 3 — Tidy the index

Update `MEMORY.md` so it stays under 200 lines and ~25KB. One line per entry, under ~150 chars: `- [Title](file.md) — one-line hook`.

- Remove pointers to retired memories
- Shorten any line carrying detail that belongs in the topic file
- Add anything newly important

Finish with a short summary: how many files you touched and what changed.
