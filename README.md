# CodeLedger

A self-building codebase index for Claude Code. CodeLedger maintains a lightweight markdown index and per-file node documents so Claude can identify the relevant files for a task without re-reading the whole codebase across sessions.

## How it works

**First task (no index yet)**
Claude works normally, documents every file relevant to the task, then builds the index after the task is done.

**Every task after**
Claude reads the index first, identifies the relevant node files, and works from those summaries — opening source files only for the ones it's actually going to edit.

Nodes are for **triage and discovery**: deciding which files matter and how they connect. When Claude edits a file, it always reads the real source first — summaries inform *which* files to open, not what their exact contents are.

## Install

### As a plugin (recommended)

```
/plugin marketplace add <your-github>/codeledger
/plugin install codeledger@codeledger
```

### Manual

Drop the skill into your project:

```
your-repo/
└── .claude/
    └── skills/
        └── codeledger/
            └── SKILL.md
```

Or install globally (applies to all projects):

```
C:\Users\YourName\.claude\skills\codeledger\SKILL.md   # Windows
~/.claude/skills/codeledger/SKILL.md                   # Mac/Linux
```

## What gets created

```
.claude/codeledger/
├── index.md          ← project summary, tech stack, file list with status
└── nodes/
    └── src__auth__login.ts.md   ← one node per source file touched
```

Node filenames encode the source path by replacing each `/` with `__` (double underscore), keeping the original filename — dashes included — intact. So `src/my-utils/parse-config.ts` becomes `src__my-utils__parse-config.ts.md`, and the mapping is unambiguous in both directions.

Both files are plain markdown — git-tracked, human-readable, and diffable in PRs.

## How nodes and the index look

### Index (`index.md`)

```markdown
# Project Index

## Summary
Analogital is a PWA that converts handwritten notes into searchable digital documents.
Users register/login (JWT-style session tokens); upload is rate-limited to 5 pages/day.
Admin users (granted via access token at signup) have a debug dashboard to view user
stats and purge accounts. Mobile: camera upload → hybrid OCR (Vision + Gemini) →
sentence-level bboxes → tagging. Desktop (≥1024px): warm-parchment shell with sidebar
(Notes/Categories/Graph tabs), split-view note editor, and force-directed knowledge graph.

## Tech Stack
Python 3.11+, Flask, SQLite, Pillow, Google Cloud Vision (OCR + bbox),
Gemini 2.5 Flash (transcription + tagging + crop), React 18, Vite 5,
Tailwind CSS 3, React Router v6, vite-plugin-pwa, mermaid 11

## Files
| File | Tags | Summary | Status |
|------|------|---------|--------|
| backend/app.py | backend, api, flask, auth, admin | Flask entry point — auth endpoints, admin endpoints, rate-limited upload, notes/bbox/crop/categories/threads/links | active |
| backend/database.py | backend, database, sqlite | SQLite CRUD: notes (incl. delete_note with cascade) + categories + threads + note_links | active |
| backend/users_db.py | backend, auth, database, users, rate-limit | Separate users.db: users + sessions + usage tables; session token auth; 5-pages/day enforcement | active |
| backend/ocr.py | backend, ocr, google-cloud-vision, gemini | Hybrid OCR: Vision bboxes + Gemini text; enable_bbox flag skips Vision + call 2 | active |
| frontend/src/App.jsx | frontend, routing, auth | Router root: AuthProvider wrapper; RequireAuth/RequireAdmin/GuestOnly guards; forks to desktop/mobile routes | active |
| frontend/src/components/CategoriesPanel.jsx | frontend, component, categories, threads | Sub-tab panel for managing categories and threads with note assignment | active |
| frontend/src/components/GraphView.jsx | frontend, component, graph, canvas | Force-directed knowledge graph: pointer+link tools, 420-frame simulation, info panel | active |
| backend/old_ocr.py | backend, ocr | Legacy single-call OCR pipeline | removed 2026-05-02 |
| ... | ... | ... | ... |
```

Deleted files keep their row with a `removed` status instead of vanishing, so Claude retains context about what used to exist. Stale entries are pruned after 90 days.

### Node (`.claude/codeledger/nodes/backend__app.py.md`)

```markdown
# app.py

## Summary
Flask application entry point for Analogital. Configures CORS, initializes both SQLite databases (notes + users) on startup, and exposes the full API surface. Includes auth endpoints (register/login/logout/me), admin endpoints (users list, daily detail, purge), and rate-limits `/upload` to 5 pages/day per authenticated user. `ocr_text` is treated as immutable ground truth — bbox mutation endpoints update only `bbox_data`.

## Functions
- `_get_current_user()` — reads `Authorization: Bearer <token>` header, returns user dict or None
- `_require_auth()` — returns (user, None) or (None, 401 response)
- `_require_admin()` — returns (user, None) or (None, 401/403 response)
- `auth_register()` — POST /auth/register: {email, password, access_token?}; access_token == ADMIN_ACCESS_TOKEN → is_admin=True; returns {token, user}
- `auth_login()` — POST /auth/login: {email, password}; returns {token, user}
- `auth_logout()` — POST /auth/logout: deletes session token
- `auth_me()` — GET /auth/me: returns {id, email, is_admin, pages_today, daily_limit}
- `admin_users()` — GET /admin/users: admin-only; returns all user stats
- `admin_user_detail(user_id)` — GET /admin/users/<id>: admin-only; returns daily breakdown for one user
- `admin_user_delete(user_id)` — DELETE /admin/users/<id>: admin-only; purges user + sessions + usage
- `upload()` — POST /upload: requires auth; checks daily page limit (5); saves image, runs OCR pipeline, records usage (page_count + char_count), returns note data
- `notes_list()` — GET /notes
- `note_detail(note_id)` — GET /notes/<id>
- `note_delete(note_id)` — DELETE /notes/<id>
- `note_update(note_id)` — PATCH /notes/<id>
- `note_bbox_update(note_id)` — PATCH /notes/<id>/bbox
- `note_bbox_merge(note_id)` — POST /notes/<id>/bbox/merge
- `_transcription_pos(bbox_text, ocr_text) -> int` — approximate char-index lookup for merge ordering
- `note_crop(note_id)` — POST /notes/<id>/crop
- `serve_upload(filename)` — GET /static/uploads/<filename>

## Non-function code
Loads `.env`. Creates `static/uploads/`. Calls `init_db()` and `init_users_db()` at module level. Imports `werkzeug.security` for password hashing.

## Imports
- database — all note/category/thread/link CRUD
- ocr — extract_text_with_bbox
- preprocessing — preprocess_image
- tagging — generate_tags
- users_db — auth + usage functions
- werkzeug.security — generate_password_hash, check_password_hash

## Imported by
- (entry point, not imported)

## Tags
backend, api, flask, upload, routing, auth, admin, rate-limit

## Node path
backend/app.py

## Last verified
2026-06-11
```

### Node (`.claude/codeledger/nodes/frontend__src__components__CategoriesPanel.jsx.md`)

```markdown
# CategoriesPanel.jsx

## Summary
Panel with two sub-tabs — "Categories" and "Threads" — each showing a CRUD list of
user-defined groups (color + name) with note assignment UI. Threads have an extra
"Graph" button to filter the graph view to that thread.

## Functions
- ColorPicker({ value, onChange }) — 6-color swatch row for category/thread color selection
- ItemRow({ item, noteIds, notes, onDelete, onAssign, onUnassign, onViewGraph }) — collapsible row: header with count + controls, body with assigned/unassigned note lists
- noteLabel(note) — extracts first-line label from ocr_text (uses note.id, not note_id)
- CreateForm({ placeholder, onSubmit }) — inline form with text input + color picker + submit
- CategoriesPanel({ notes, categories, threads, onRefresh, onViewGraph }) — main export

## Non-function code
- PALETTE — 6 hardcoded hex colors for picker
- Sub-tab state via useState('categories')

## Imports
- react (useState) — hooks
- ../api — createCategory, deleteCategory, assignNoteCategory, unassignNoteCategory,
           createThread, deleteThread, assignNoteThread, unassignNoteThread

## Imported by
- frontend/src/components/AppSidebar.jsx — renders in the "categories" tab section

## Tags
frontend, component, categories, threads, crud, sidebar-panel

## Node path
frontend/src/components/CategoriesPanel.jsx

## Last verified
2026-06-11
```

## Staying fresh

Every node carries a `Last verified` date. Before relying on a node, Claude compares it against the source file's modification time — if the file was edited outside Claude (by you, a teammate, or another tool), the node is treated as stale, re-read from source, and rewritten. Nodes always describe the file's *current* state: no changelogs, no "NEW"/"CHANGED" markers, no references to previous versions.

## Notes

- Tags are dynamic — Claude adds new ones as it encounters new patterns
- Nodes are updated after every task that touches the file; `Last verified` is bumped on each update
- The index is updated whenever files are added, removed, or significantly changed
- Removed files and deprecated functions are soft-deleted (kept with a dated marker), then pruned after 90 days

## When CodeLedger helps — and when it doesn't

CodeLedger pays off on **medium-to-large codebases** worked across **many sessions**, where re-orienting Claude is the dominant cost. The real value is the index plus the import graph: Claude knows *which* files to touch without exploring. On very small projects, or for one-off tasks, the node-maintenance overhead can exceed the savings — detailed nodes for small files are a meaningful fraction of the source itself.