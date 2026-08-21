# audit-loop — agent guide

Centralized plugin + operations hub for the architecture-audit loop. See
`README.md` for the full component map and `docs/runbook.md` for operations.

@CONTEXT.md

## Hard rules

- **Use the glossary.** All docs, issues, and PR bodies use the canonical
  terms in `CONTEXT.md`; the `_Avoid_` lists are binding (e.g. never
  "accepted finding" — it collides with "approved finding").
- **Runtime files are fleet-shipped.** `agents/arch-auditor.md`,
  `commands/audit.md`, and `commands/audit-fix.md` are vendored into every
  consumer; a change on `main` opens update PRs across the fleet. Keep them
  portable — no project-specific rules baked in, ever.
- **`state/audit-digest.json` is workflow-owned.** Only `audit-digest.yml`
  commits it; never edit it by hand.
- **`consumers.txt` is the membership.** One `owner/repo` per line;
  registration there — not vendored files — is what makes a repo a consumer.
- **Templates ship at install time only.** Changing `templates/` never
  retro-updates existing consumer baselines.
- Docs are written in English.

## Agent skills

### Issue tracker

Issues are tracked in this repo's GitHub Issues via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Domain docs

Single-context: one `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
