---
name: codeledger
description: >
  Use this skill for every coding task in this project. CodeLedger maintains
  a lightweight index and per-file nodes so you never read redundant code.
  Always check for the index before doing any work.
---

# CodeLedger

## Step 1 — Check for index

Before doing anything else, check if `.claude/codeledger/index.md` exists.

---

## Path A — No index found

Work as you normally would. As you read each file to complete the task:

1. Complete the task first
2. For every file you read during the task, create a node at:
   `.claude/codeledger/nodes/<file-path-with-dashes>.md`
   Example: `src/auth/login.ts` → `.claude/codeledger/nodes/src-auth-login.md`
3. After the task is done, create `.claude/codeledger/index.md`

**Node format:**
```
# <filename>

## Summary
<one paragraph describing what this file does>

## Functions
- functionName(params) — what it does

## Non-function code
<describe any code outside of functions: constants, middleware, config, class definitions, etc.>

## Imports
- path/to/file — why it's imported

## Imported by
- path/to/file — why it imports this file

## Tags
<comma-separated, dynamic — e.g. backend, auth, api, frontend, config, utils>

## Node path
<file path relative to project root>
```

**Index format:**
```
# Project Index

## Summary
<what this project does overall>

## Tech Stack
<languages, frameworks, databases, key libraries>

## Files
| File | Tags | Summary |
|------|------|---------|
| src/auth/login.ts | auth, backend | Handles login and session creation |
| ... | ... | ... |
```

---

## Path B — Index exists

1. Read `index.md`
2. Determine if the user's query is solvable with the current codebase
   - If not solvable or unclear: ask the user for clarification before proceeding
3. From the index, identify which files are relevant to the task
4. Read only those node files from `.claude/codeledger/nodes/`
5. If a node references an imported file that also seems relevant, read that node too
6. Generate a list of action items based on the query and the node contents
7. If the changes are large (multiple files, structural changes, or deletions): present the action items to the user and ask for confirmation before proceeding. For small changes, proceed directly.
8. Make the changes
9. Update all touched node files to reflect what changed
10. Update `index.md` if the file list, tech stack, or project summary changed

---

## Always

- Never read source files directly if a node exists for them — the node is your source of truth
- If you read a source file that has no node yet, create one after the task
- Keep node summaries concise — one paragraph max, bullet points for functions
- Tags are dynamic — add new ones as needed, don't restrict to a fixed list
- If a node feels outdated after a change, rewrite it — accuracy matters more than preservation
