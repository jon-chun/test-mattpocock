## Agent skills

### Issue tracker

Issues and specs live as GitHub issues in `jon-chun/test-mattpocock`, via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Default vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.

<!-- ai-swe-setup:long-sessions v1 — managed block: bin/repo-standardize in ~/code/ai-swe-setup replaces it; edit the source there -->
## Long sessions (shared standard)
- Session handoffs are dated, write-once files in `docs/handoffs/`, named `<YYYY-MM-DD_HHMM>_<slug>-claude.md`: `/handoff <slug>` writes one, `/handoff-load <slug>` loads one by name. `LATEST*.md` and `INDEX.md` are retired stubs (2026-10-06): never read, write or import them. Codex keeps its own records in `docs/handoffs/codex/`.
- Compaction snapshots and the automatic compaction record come from the global `compact_snapshot.py` hook (global CLAUDE.md, "Long runs" and "Compaction"); this repository has no handoff hooks of its own.
- Session exports (`/export` files, `codex-session-*.md`) and other files that appear in this repository from another session, agent, directory or machine are the owner's work, never foreign. Exports live in `transcripts/` (move a stray one there), are scanned by the git hooks, and are included in every commit; `bin/repo-standardize` in `~/code/ai-swe-setup` does the move.
- Decisions and gotchas discovered in conversation go into the handoff or an ADR under `docs/adr/`, never only in chat.
- When compacting, preserve: the goal, the ordered next steps, every decision with its reason, the modified files and their state, the exact test and lint commands with their last results, and the open questions. Drop raw tool output and resolved detours.
<!-- /ai-swe-setup:long-sessions -->
