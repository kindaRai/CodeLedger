# CodeLedger

A self-building codebase index for Claude Code. CodeLedger maintains a lightweight index and per-file node documents so Claude never reads redundant code across sessions.

## How it works

**First task (no index yet)**
Claude works normally, documents every file it reads, then builds the index after the task is done.

**Every task after**
Claude reads the index first, identifies the relevant node files, and works with full context — without touching files it doesn't need.

## Install

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
├── index.md          ← project summary, tech stack, file list
└── nodes/
    └── src-auth-login.md   ← one node per source file touched
```

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
| File | Tags | Summary |
|------|------|---------|
| backend/app.py | backend, api, flask, auth, admin | Flask entry point — auth endpoints, admin endpoints, rate-limited upload, notes/bbox/crop/categories/threads/links |
| backend/database.py | backend, database, sqlite | SQLite CRUD: notes (incl. delete_note with cascade) + categories + threads + note_links |
| backend/users_db.py | backend, auth, database, users, rate-limit | Separate users.db: users + sessions + usage tables; session token auth; 5-pages/day enforcement |
| backend/ocr.py | backend, ocr, google-cloud-vision, gemini | Hybrid OCR: Vision bboxes + Gemini text; enable_bbox flag skips Vision + call 2 |
| frontend/src/App.jsx | frontend, routing, auth | Router root: AuthProvider wrapper; RequireAuth/RequireAdmin/GuestOnly guards; forks to desktop/mobile routes |
| frontend/src/components/CategoriesPanel.jsx | frontend, component, categories, threads | Sub-tab panel for managing categories and threads with note assignment |
| frontend/src/components/GraphView.jsx | frontend, component, graph, canvas | Force-directed knowledge graph: pointer+link tools, 420-frame simulation, info panel |
| ... | ... | ... |
```

### Node (`.claude/codeledger/nodes/backend-app.md`)

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
```

### Node (`.claude/codeledger/nodes/frontend-src-components-CategoriesPanel.md`)

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
```

## Notes

- Tags are dynamic — Claude adds new ones as it encounters new patterns
- Nodes are updated after every task that touches the file
- The index is updated whenever files are added, removed, or significantly changed
- Claude never re-reads a source file if a node exists for it — the node is the source of truth
