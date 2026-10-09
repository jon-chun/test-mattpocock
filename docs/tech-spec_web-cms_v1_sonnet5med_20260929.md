# Tech Spec — Web CMS (IPHS 400 MP2)

**Client:** Knox County Historical Society (default brief + 3 added specifics, see §1)
**Stack:** FastAPI + Jinja2 + SQLite + argon2, Markdown (mistune/markdown-it) + bleach/nh3 sanitizer
**Workflow:** `/grill-with-docs → /to-spec → /to-tickets → /implement (/tdd) → /code-review`
**Scope:** CMS application architecture **and** the process/deliverable scaffolding the rubric grades (CONTEXT.md, ADRs, tickets, notes/, docs/)
**Repo:** this repo (`test-mattpocock`) becomes the working repo; template structure below is created as part of setup, not assumed pre-existing.

---

## 1. Client brief (input to the grill, not a substitute for it)

Knox County Historical Society, three volunteers:
- **Karen** (secretary) — posts weekly meeting announcements and event notices. Comfortable with email only; will never remember a slug or Markdown syntax by heart, so the editor needs a visible preview and auto-slug.
- **Tom** (president) — writes longer research pieces every 1–2 months; wants them to read as permanent, citable pages, not perishable posts.
- **Added specifics (beyond the manual's default):**
  1. The society has an annual **membership roster page** that only Tom may edit (no volunteer may delete it — a hard constraint, candidate for an ADR).
  2. Karen currently double-posts meeting notices to Facebook and a paper newsletter; the CMS must make "what's the next meeting" answerable from the homepage without a click-through.
  3. Past volunteers have accidentally published half-finished drafts; Karen has asked for an explicit "are you sure" confirmation on publish, not just a draft/publish toggle.

These specifics exist to seed the grill — the actual grill session (`/grill-with-docs`) will confirm, override, or extend them, and that transcript is the graded artifact, not this document.

---

## 2. Domain model (informs CONTEXT.md, not a replacement for it)

| Term | Definition for this client |
|---|---|
| **Post** | A dated item (meeting notice, event, research piece). Fields: `id, title, slug, body_md, status(draft/published), author_id, created_at, updated_at, published_at` |
| **Page** | Static content that appears in nav (About, Membership Roster). Same fields as Post minus a chronological listing; plus `nav_order` |
| **User** | `id, email, password_hash, role(admin/editor), active(bool)` |
| **Role** | `admin` = Tom (full access incl. user management, membership page); `editor` = Karen (posts + non-restricted pages) |
| **Slug** | URL-safe title fragment, auto-generated, editable; uniqueness enforced at DB level |
| **Publish** | One-way render: DB rows with `status=published` → static HTML in `site/`, relative paths only |

Restricted-page rule ("membership page: admin only"): modeled as a `Page.admin_only: bool` flag, not a separate table — avoids a premature abstraction for a rule that only ever applies to one page today. If a second restricted page appears later, promote to a permissions table then, not now. **Candidate ADR**: "why a boolean flag, not a permissions table, for page-level restriction."

---

## 3. Application architecture

```
app/
├── main.py               # FastAPI app, router mounting
├── models.py              # SQLModel/SQLAlchemy: User, Post, Page
├── auth.py                 # session cookies, argon2 hash/verify, login_required/admin_required deps
├── routes/
│   ├── public.py           # GET / , GET /{slug}, GET /posts (published only)
│   ├── admin_auth.py        # /admin/login, /admin/logout
│   ├── admin_dashboard.py    # /admin (counts)
│   ├── admin_posts.py         # CRUD for posts, filterable list
│   ├── admin_pages.py          # CRUD for pages
│   └── admin_users.py           # admin-only: create/role-change/deactivate
├── services/
│   ├── markdown.py              # render + sanitize (nh3/bleach allowlist)
│   ├── slugify.py
│   └── publish.py                 # renders site/ from published rows, relative paths
└── csrf.py                         # token issue/verify for all state-changing forms
templates/
├── base.html, admin/base.html
├── public/ (home, post, page, nav partial)
└── admin/ (login, dashboard, post_form, page_form, users)
tests/
├── conftest.py             # client_as("admin"|"editor"|None) fixture
├── test_auth.py, test_posts.py, test_pages.py, test_users.py, test_publish.py, test_security.py
scripts/
├── check_submission.py, seed_demo.py, usage_report.py (spend() stub)
.claude/
├── hooks/ctx_statusline.py, ctx_guard.py (stub), ctx_compact_log.py
├── skills/  (mattpocock, pinned)
└── settings.json
docs/adr/ (ADR-001, ADR-002 examples + ours)
notes/ (cms-field-notes.md, client-brief.md, token-budget-plan.md, usage-ledger.csv)
CLAUDE.md, SECURITY-CHECKLIST.md, README.md
```

### Auth & sessions
- Login sets a signed session cookie (server-side session id → user id); no JWT needed at this scale.
- `argon2-cffi` for hashing; verified against manual D1.
- Dependencies: `require_login`, `require_admin` — FastAPI `Depends`, return 403/redirect-to-login per route.

### CSRF
- Every admin POST form carries a hidden per-session token; middleware/dependency rejects mismatched or missing tokens (D2).

### Markdown & sanitization
- Author writes Markdown → rendered to HTML → passed through an allowlist sanitizer (strip `<script>`, `on*` attributes) **both** at preview time and at publish/export time (D4 requires both paths clean).

### Publish pipeline
- `cms publish`: query `status=published` rows → render templates → write `site/` with relative asset paths (no leading `/`) → `cms deploy` pushes `site/` to `gh-pages`.
- Nav is built from `Page` rows only (`nav_order`), never from posts.

---

## 4. The 7 capabilities → routes/tests map

| # | Capability | Key routes | Key tests |
|---|---|---|---|
| 1 | Login | `/admin/login`, `/admin/logout` | wrong password refused; hash format |
| 2 | Roles | all `/admin/*` | editor 403 on `/admin/users` |
| 3 | Posts | `/admin/posts[/{id}]` CRUD | create→publish→unpublish→delete persists across restart |
| 4 | Pages | `/admin/pages[/{id}]` CRUD | published page appears in nav |
| 5 | Admin console | `/admin` dashboard, `/admin/posts?status=` filter, preview in editor | counts correct; filter correct; preview renders sanitized MD |
| 6 | User mgmt | `/admin/users` (admin only) | create, role change, deactivate → login blocked |
| 7 | Publish | `cms publish`, `cms deploy` | only published rows in `site/`; relative paths; Pages URL reflects it |

---

## 5. Security baseline (maps to rubric D)

- D1 argon2/bcrypt hashes only, never plaintext.
- D2 CSRF token on every state-changing form.
- D3 access-control tests: every admin-only route asserts 403/redirect for editor and anonymous.
- D4 sanitize on both preview and export paths; test with `<script>` + `onerror=`.
- D5 `.env` / `*.db` gitignored; `.env.example` committed; no secrets in history.

---

## 6. Process scaffolding (rubric B, G, H)

- **CONTEXT.md**: living glossary, filled during `/grill-with-docs` — post, page, slug, role, draft/published, publish, membership-roster restriction.
- **docs/adr/**: ADR for the page-restriction design (§2), plus any other hard-to-reverse call surfaced in the grill.
- **Tickets** (7–9, tracer-bullet shaped, vertical slices touching model+route+template+test):
  1. T01 Login + session + role check on one protected route
  2. T02 Post CRUD (editor) + admin post list
  3. T03 Post publish/unpublish + public post page
  4. T04 Page CRUD + nav build from published pages
  5. T05 Membership-roster restriction (admin-only page)
  6. T06 Admin dashboard counts + status filter + Markdown preview
  7. T07 User management (create/role/deactivate)
  8. T08 `cms publish`/`cms deploy` + relative-path check
  9. T09 (stretch buffer / hardening: CSRF + sanitizer tests across all forms)
- **notes/**: `cms-field-notes.md` (WordPress Playground, ≥5 observations), `client-brief.md`, `token-budget-plan.md` (Haiku/low for research, Opus/high for grill+spec+tickets, Sonnet/medium for implement, Sonnet/high for review), `usage-ledger.csv` (auto-written by hook).
- **docs/transcripts/**, **docs/handoff/**, **docs/process/compaction-log.md**: one per session, per the manual's naming pattern.

---

## 7. Build order (respects "Blocked by" edges)

T01 → T02 → T03 → T04 → T05 (blocked by T02+T04) → T06 (blocked by T02–T04) → T07 (blocked by T01) → T08 (blocked by T03+T04) → T09 (blocked by all).

Each ticket: `/clear` → `/implement #N` (drives `/tdd`) → green → `/code-review` → post findings comment → commit `TNN: ... Closes #N` → confirm `gh issue view N` shows CLOSED with the review comment preceding the closing commit.

---

## 8. Open items resolved by this spec's assumptions (confirm/override in the actual grill)

- Page-restriction as a boolean flag, not a roles/permissions table (revisit only if a second restricted page appears).
- Single `role` enum column on `User` (admin/editor), not a many-to-many roles table — two fixed roles, no need for the general case.
- SQLite only, no migration framework beyond simple `CREATE TABLE IF NOT EXISTS` at this scale — reversible if wrong, not an ADR.
