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
2. After the task, for every file you read during the task, create a node at:
   `.claude/codeledger/nodes/<file-path-with-dashes>.md`
   Example: `src/auth/login.ts` → `.claude/codeledger/nodes/src-auth-login.md`
3. After all nodes are created, create `.claude/codeledger/index.md`

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
| File | Tags | Summary | Status |
|------|------|---------|--------|
| src/auth/login.ts | auth, backend | Handles login and session creation | active |
| ... | ... | ... | ... |
```

---

## Path B — Index exists

1. Read `index.md`

2. **Orphan check** — run both directions before doing any work. Do not open any files
   during this step — only check whether files exist on disk.

   **Index → nodes (missing node):** For every file listed in the index with status `active`,
   check whether its corresponding node file exists in `.claude/codeledger/nodes/`.
   If the node file is missing, flag it — it will be generated in step 6 or step 11
   when that file becomes relevant to the task.

   **Nodes → index (missing index entry):** List all filenames in `.claude/codeledger/nodes/`
   without opening them. For every node filename that has no corresponding row in the index:
   - Derive the source file path from the node filename (reverse the dash-to-slash conversion)
   - Check if that source file exists in the project
   - If it exists: add a new `active` row to the index (summary and tags will be filled
     when the node is read during a relevant task)
   - If it no longer exists: add the row with status `[removed — YYYY-MM-DD]`

3. Determine if the user's query is solvable with the current codebase
   - If not solvable or unclear: ask the user for clarification before proceeding

4. From the index, identify which files are relevant to the task

5. Read only those node files from `.claude/codeledger/nodes/`

6. If a node references an imported file that also seems relevant, read that node too.
   If that node is missing, generate it from the source file before continuing (same as step 2)

7. Generate a list of action items based on the query and the node contents

8. If the changes are large (multiple files, structural changes, or deletions): present
   the action items to the user and ask for confirmation before proceeding.
   For small changes, proceed directly.

9. Make the changes

10. Update all touched node files to reflect what changed:
    - If a function was added: add it to the Functions section
    - If a function was removed: mark it as `[deprecated — removed YYYY-MM-DD]` rather
      than deleting it. Do not remove deprecated entries.
    - If a function's behaviour changed: update its description in place

11. If a source file read during this task has no node yet, create one now

12. Update `index.md` if anything changed:
    - New file added to project → add a new row with status `active`
    - File deleted from project → change its status to `[removed — YYYY-MM-DD]`, do not
      delete the row
    - Tech stack or project summary changed → update those sections

---

## Always

- Never read source files directly if a node exists for them — the node is your source of truth
- Keep node summaries concise — one paragraph max, bullet points for functions
- Tags are dynamic — add new ones as needed, don't restrict to a fixed list
- If a node feels outdated after a change, rewrite the summary — accuracy matters more than preservation
- Never hard-delete anything from nodes or the index — deprecate instead