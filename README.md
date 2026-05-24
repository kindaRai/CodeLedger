# CodeLedger

A self-building codebase index for Claude Code. CodeLedger maintains a lightweight index and per-file node documents so Claude never reads redundant code across sessions.

## How it works

**First task (no index):**
Claude works normally, documents every file it touches, then builds the index after the task is done.

**Every task after:**
Claude reads the index first, fetches only the relevant node files, and codes with full context — without touching files it doesn't need.

## Install

Copy the `skills/codeledger` folder into your project:

```
your-repo/
└── .claude/
    └── skills/
        └── codeledger/
            └── SKILL.md
```

Or for personal install (all projects):

```
C:\Users\YourName\.claude\skills\codeledger\SKILL.md   # Windows
~/.claude/skills/codeledger/SKILL.md                   # Mac/Linux
```

## What gets created

```
.claude/codeledger/
├── index.md          ← project summary, tech stack, file list
└── nodes/
    └── src-auth-login.md   ← one file per source file touched
```

Both are plain markdown — git-tracked, human-readable, and diffable in PRs.

## Notes

- Tags are dynamic — Claude adds new ones as it encounters new patterns
- Nodes are updated after every task that touches the file
- The index is updated whenever files are added, removed, or significantly changed
