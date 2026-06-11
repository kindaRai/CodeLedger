---
name: codeledger
description: >
  Use this skill for every coding task in this project. CodeLedger maintains
  a lightweight index and per-file nodes so you can identify relevant files
  without re-reading the whole codebase. Always check for the index before
  doing any work.
---

# CodeLedger

## What nodes are for

Nodes are for **triage and discovery** — deciding which files matter for a task
and understanding how they connect. They are not a substitute for source code
when editing. The rule is:

- **Discovery**: read nodes, never source
- **Editing**: read the actual source file of any file you are about to modify.
  Never edit a file based on its node alone.

---

## Step 1 — Check for index

Before doing anything else, check if `.claude/codeledger/index.md` exists.

---

## Path A — No index found

Work as you normally would. Then:

1. Complete the task first
2. After the task, create a node for every file that was **relevant to the
   task** — files you read to understand or change behaviour. Skip files you
   only glanced at incidentally (lockfiles, unrelated configs, files opened
   and immediately dismissed).
   Node location: `.claude/codeledger/nodes/<encoded-file-path>.md`

   **Path encoding:** replace each `/` with `__` (double underscore).
   Example: `src/auth/login.ts` → `src__auth__login.ts.md`
   Keep the original filename (including its extension and any dashes or
   underscores) intact. This makes the encoding reversible: split on `__`
   to recover the path. Never use single dashes as separators — dashes occur
   in real filenames and make the mapping ambiguous.
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

## Last verified
<YYYY-MM-DD>
```

Nodes describe the **current state** of the file only. Never include diff
markers (`NEW`, `SIGNATURE CHANGED`, `UPDATED`) or references to previous
versions — a node is a snapshot, not a changelog. When updating a node,
rewrite affected entries as if writing them for the first time.

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

The Status column is required. Valid values: `active`,
`removed YYYY-MM-DD`.

---

## Path B — Index exists

1. Read `index.md`

2. **Orphan check** — run this only when either (a) a node you expect from the
   index is missing, (b) a node's source file turns out not to exist, or
   (c) the check hasn't been run in this session yet AND the task touches
   multiple files. For trivial single-file tasks with a healthy index, skip it.

   When you do run it, check both directions. Do not open node files during
   this step except as noted — only check existence on disk.

   **Index → nodes (missing node):** For every file listed in the index with
   status `active`, check whether its node file exists in
   `.claude/codeledger/nodes/`. If missing, flag it — it will be generated in
   step 6 or step 11 when that file becomes relevant.

   **Nodes → index (missing index entry):** List all node filenames. For every
   node with no corresponding index row:
   - Derive the source path by splitting the node filename on `__`
   - For legacy nodes using the old single-dash encoding (or if the derived
     path doesn't exist on disk), open **only the `## Node path` line** of the
     node to get the true path, then rename the node file to the `__` encoding
   - If the source file exists: add an `active` row to the index (summary and
     tags filled when the node is next read during a relevant task)
   - If it no longer exists: add the row with status `removed YYYY-MM-DD`

3. Determine if the user's query is solvable with the current codebase
   - If not solvable or unclear: ask the user for clarification before proceeding

4. From the index, identify which files are relevant to the task

5. Read only those node files from `.claude/codeledger/nodes/`

6. **Staleness check** — for each relevant node, compare the source file's
   modification time against the node's `## Last verified` date (use
   `stat`/`Get-Item` — do not open the source file for this). If the source
   is newer, the node may be stale: read the source file, rewrite the node to
   match reality, and update `Last verified`. Files are sometimes edited
   outside Claude; never trust a node older than its source.

7. If a node references an imported file that also seems relevant, read that
   node too. If that node is missing, generate it from the source file before
   continuing

8. Generate a list of action items based on the query and the node contents

9. If the changes are large (multiple files, structural changes, or deletions):
   present the action items to the user and ask for confirmation before
   proceeding. For small changes, proceed directly.

10. Make the changes. **Always read the current source of any file before
    editing it** — nodes inform which files to open, not what their exact
    contents are.

11. Update all touched node files to reflect what changed:
    - Function added → add it to Functions
    - Function removed → mark it `[deprecated — removed YYYY-MM-DD]` rather
      than deleting it
    - Function behaviour changed → rewrite its description in place, with no
      reference to the old behaviour
    - Update `Last verified` to today's date

12. If a source file read during this task has no node yet, create one now

13. Update `index.md` if anything changed:
    - New file added to project → add a row with status `active`
    - File deleted from project → set status to `removed YYYY-MM-DD`; don't
      delete the row
    - Tech stack or project summary changed → update those sections

---

## Pruning

Soft deprecation is for recovery and context, not permanent archival. During
any task where you're already editing a node or the index:

- Drop `[deprecated]` function entries older than 90 days
- Drop `removed` index rows older than 90 days

Don't run pruning as a standalone pass — fold it into edits you're making anyway.

---

## Always

- Nodes are for triage; source is for editing. Never modify a file you haven't
  read in its current form, and never read source for pure discovery when a
  fresh node exists
- Keep node summaries concise — one paragraph max, bullet points for functions
- Nodes are snapshots of current state, never changelogs — no NEW/CHANGED
  markers, no "previously did X" phrasing
- Tags are dynamic — add new ones as needed, don't restrict to a fixed list
- If a node feels outdated after a change, rewrite the summary — accuracy
  matters more than preservation
- Soft-delete (deprecate) rather than hard-delete, then prune per the
  Pruning section