---
name: memory
description: Persistent memory across Copilot sessions. Use this skill at the START of every task (to recall past context and decisions) and at the END of every task (to save a memory of what was done). Always invoke this skill automatically — no user prompt needed.
---

# Memory Skill

This skill gives you persistent memory across sessions. Follow both phases on every task.

---

## Phase 1 — Recall (run BEFORE starting any task)

Before doing any work, load your memory:

```bash
ls ~/.copilot/memories/ 2>/dev/null
```

If the directory is empty or missing, skip to Phase 2 after the task completes.

If memory files exist, read them all:

```bash
cat ~/.copilot/memories/*.md 2>/dev/null
```

Silently internalize the contents — do NOT repeat them back to the user verbatim. Use the recalled context to:
- Avoid re-doing things already completed
- Apply previously agreed-upon conventions or preferences
- Understand the history of the project or codebase
- Pick up ongoing work where it left off

---

## Phase 2 — Save (run AFTER completing any task)

After finishing a task, save a concise memory file.

### Naming

Generate a short kebab-case summary of what was done (3–6 words), then write the file:

```bash
cat > ~/.copilot/memories/memory-{task-summary}.md << 'EOF'
---
date: {ISO-8601-date}
task: {one-line description of what was done}
---

## What Was Done
{2–4 sentences describing the task, what changed, and why}

## Key Decisions
- {Any architectural, design, or preference decisions made}
- {Libraries, tools, or approaches chosen and why}
- {Things the user explicitly asked for or rejected}

## Files Changed
- {List of key files created or modified, if applicable}

## Follow-up / Notes
{Any unfinished items, known issues, or next steps the user mentioned}
EOF
```

### Rules
- **Always save a memory** — even for small tasks like answering a question or explaining code
- Keep files focused and concise — aim for under 30 lines
- Use the ISO-8601 date from the `<current_datetime>` value already present in your system context — no need to run `date`
- If a memory for the same task already exists, append a `-2`, `-3` suffix rather than overwriting
- Omit sections that don't apply (e.g. "Files Changed" for a pure Q&A task)

---

## Memory File Format Reference

```markdown
---
date: 2025-03-24T02:00:00Z
task: Created Express REST API for user auth
---

## What Was Done
Built a JWT-based auth API in Express with login, logout, and token refresh endpoints.
Used bcrypt for password hashing. Created in src/auth/.

## Key Decisions
- JWT with 1h expiry (user preference)
- bcrypt cost factor 12
- No session storage — stateless design

## Files Changed
- src/auth/index.ts (created)
- src/auth/middleware.ts (created)
- src/routes/index.ts (modified)

## Follow-up / Notes
User mentioned wanting to add OAuth later. Refresh token endpoint not yet implemented.
```
