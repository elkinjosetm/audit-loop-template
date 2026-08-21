# audit-loop

This is the **template** for running your own audit-loop hub: fork it, create
your own GitHub App and credentials, then follow the setup sections below. A
complete step-by-step adoption guide is tracked in this repo's issues. This
template is extracted from a private reference instance that runs the loop in
production.

A centralized Claude Code plugin **and operations hub** for the
architecture-audit loop: one source of truth for the `arch-auditor` agent,
the `/audit` / `/audit-fix` commands, and the supporting GitHub Actions.
Consumer repos get **managed, hash-stamped vendored copies** written by
`/audit-install`; after that, this repo pushes updates out by itself — a
change to the runtime files on `main` triggers `audit-fleet-update.yml`,
which opens an update PR in every repo listed in `consumers.txt`. This repo
is also the only place notifications live: `audit-digest.yml` sweeps all
consumers on demand (each session triggers a sweep after it files) and sends
at most one Pushover push per project with news, so
consumers hold zero notification secrets and zero notify workflows. It is also the only place
GitHub write access lives for filing: consumer sessions never create issues
or PRs directly — they dispatch to this hub's `audit-file.yml` relay, which
files the item in the target consumer as the App.

## What this is

The audit loop was battle-tested inside a single project first: a read-only
auditor subagent that judges code only against standards a repo documents
about itself, a command that runs it and files each run's findings as one
audit issue **through the hub relay** (App-authored), and a second command
that turns owner-approved findings into scoped pull requests, also filed
through the relay. This repo extracts that loop out of any one project and
ships it as an installable plugin with a single setup command, so every repo that wants
the same audit discipline installs from one source of truth instead of
keeping an unmanaged copy.

Why vendored copies rather than plugin-loading everywhere: cloud routine
sessions (the unattended scheduled runs) do **not** install marketplace plugins
declared in a project's `.claude/settings.json` — external plugins need a
one-time interactive install that a headless session can't perform. So
`/audit-install` vendors the runtime files into each consumer repo with a
content-hash marker; the plugin stays the source of truth and updates are a
re-run away.

## Components

- **`arch-auditor` agent** — read-only architecture & conformance auditor.
  Runs in FIND mode (scan one lens) or REFUTE mode (adversarially try to kill
  a candidate finding). Never edits code, never files issues, never hardcodes
  a project's rules.
- **`/audit`** — orchestrates five `arch-auditor` finder lenses (standards
  conformance, extractable primitives/DRY, dead code, ADR conformance, module
  depth) in parallel, cross-lens dedupes, adversarially verifies every
  candidate, suppresses baselined and already-tracked findings, prints a
  prioritized report, and — only when something new survives — dispatches
  those findings to the hub relay, which files them as **one new audit
  issue** per run (label `audit`, one checkbox per finding, App-authored).
  Historical dedupe against all open audit issues keeps a daily cadence from
  re-filing anything, so it never runs `gh issue create` and never edits an
  existing issue's body.
- **`/audit-fix`** — sweeps open `audit`-labeled issues, acts on **only**
  checked findings, and dispatches one scoped PR per approved finding (branch
  pushed by a fresh implementer subagent, PR itself filed through the hub
  relay). Never merges, never pushes to the default branch directly, never
  runs `gh pr create`.
- **`/audit-install`** — the per-repo setup command: vendors the agent and
  the `/audit` / `/audit-fix` commands into the target repo's `.claude/`
  (each stamped with a content hash for update/drift detection), writes the
  `audit-close.yml` workflow stub, seeds `docs/audit-baseline.md`, creates
  the `audit` label, registers the repo in this hub's `consumers.txt`, and
  prints next-steps guidance for the routine setup (the one thing it can't do
  on your behalf).
- **Hub workflows** (`.github/workflows/`, run in THIS repo only):
  - `audit-digest.yml` — on-demand sweep (`workflow_dispatch` only, no
    crons): each `/audit` and `/audit-fix` session triggers one sweep as its
    final step, after its filings landed. Each sweep covers every consumer's
    open audit issues for new unchecked findings and newly opened
    `audit-fix/**` PRs (delta against a committed state snapshot) and sends
    at most **one** Pushover push per project with news, titled with that
    repo's name, every issue/PR number linked. Nothing new anywhere → no
    push (`force_push=true` overrides, as the secrets test).
  - `audit-fleet-update.yml` — fires when the vendored sources change on
    `main`; re-vendors them into every consumer (same hash/marker rules as
    `/audit-install`, never touching locally edited copies) and opens one
    update PR per repo that is behind.
  - `audit-file.yml` — relay: triggered (`workflow_dispatch`) by consumer
    sessions, validates the target against `consumers.txt`, and files the
    issue/PR in the consumer as the App (create-only). The trigger transport
    is `workflow_dispatch` because cloud routine environments have no `gh`
    CLI and their GitHub MCP tooling can run workflows but cannot send a
    `repository_dispatch` (verified 2026-07-03).
- **One reusable GitHub Actions workflow** for consumers:
  `audit-close.yml` (closes an audit issue the moment every finding in it has
  a checked box **and** a merged fix PR quoting it). Consumers call it with a
  thin stub, so a fix to the close logic lands everywhere at once. It needs
  no secrets. (The old per-repo `audit-notify.yml` reusable workflow is not
  part of this template — notifications are centralized in the hub digest.)
- **Runbook** at `docs/runbook.md` — the full operational doc: the issue
  lifecycle, the two human-owned gates, conflicted-PR handling
  (regenerate, don't rebase), and how to create and verify the two routines.

## How the auditor derives "the standard" (the portability design)

`arch-auditor` hardcodes **no project rules**. At run time it derives "the
standard" from whatever the target repo documents about itself, in priority
order:

1. The repo's root agent guide (e.g. `CLAUDE.md`) — hard rules: tooling,
   language policy, stack/build conventions.
2. The domain glossary (e.g. `CONTEXT.md`) — canonical terms, states,
   invariants.
3. A design spec (e.g. `docs/DESIGN_SPEC.md`) — UI/component conventions, if
   the repo has one.
4. The decision log (`docs/adr/`) — durable decisions; a finding citing an ADR
   names the exact clause. Superseded ADRs carry no force.
5. Established code patterns — a convention proven at 2+ call sites can anchor
   a finding even when no doc names it (cited as `convention @ sites`).

Every finding must cite one of these; anything that can't is dropped during
adversarial verification, never reported. If the target repo has **none** of
these docs, the auditor doesn't invent a standard — it emits that gap as
finding #1 instead ("repo has no documented standard to audit against"),
which is itself the first actionable fix. This is what makes the same agent
file work unmodified across every consumer repo: the standard always comes
from the repo being audited, never from this plugin.

## Repo layout

```
audit-loop-template/
├── .claude-plugin/
│   ├── marketplace.json   # marketplace listing (this repo doubles as its own marketplace)
│   └── plugin.json        # plugin manifest
├── agents/
│   └── arch-auditor.md    # read-only FIND/REFUTE auditor, portable
├── commands/
│   ├── audit.md           # orchestrator: 5 lenses → dedupe → verify → relay-file issue
│   ├── audit-fix.md        # orchestrator: checked findings → relay-filed scoped PRs
│   └── audit-install.md    # per-repo setup: workflow stubs, baseline seed, label, next-steps guidance
├── .github/
│   └── workflows/
│       ├── audit-digest.yml        # hub: cross-repo Pushover digest (on-demand, session-triggered sweeps)
│       ├── audit-fleet-update.yml  # hub: re-vendor into consumers via PR on main push
│       ├── audit-file.yml    # hub: relay — dispatch → validate against consumers.txt → file issue/PR as the App
│       └── audit-close.yml   # reusable: auto-close on last approved fix merged
├── consumers.txt           # the fleet: one owner/repo per line, read by hub workflows
├── state/
│   └── audit-digest.json   # digest's last-seen snapshot, committed by the workflow itself
├── templates/
│   └── audit-baseline-header.md  # header-only template /audit-install copies into docs/audit-baseline.md
├── examples/
│   └── consumer-stubs/
│       └── audit-close.yml   # authoritative caller stub: on: pull_request closed + workflow_dispatch
├── docs/
│   ├── agents/
│   │   ├── domain.md              # single-context domain-doc convention
│   │   └── issue-tracker.md       # GitHub Issues via `gh` convention
│   ├── runbook.md                 # full operational runbook
│   └── spike-plugin-loading.md    # spike notes: plugin loading in cloud routine environments
├── CLAUDE.md                # agent guide for this repo
├── CONTEXT.md                # glossary — canonical terms for docs, issues, PR bodies
├── LICENSE
└── README.md
```

## Hub setup (one-time, this repo only)

All credentials in the whole system live here — consumers hold none. Repo
access comes from a **GitHub App** the owner creates once (free on any
personal account): the stored private key can only mint installation tokens,
and the token that actually touches repos is generated per workflow run and
expires in ~1 hour — nothing long-lived with repo access is ever stored.

1. Create the App at Settings → Developer settings → GitHub Apps → New
   GitHub App: name e.g. `audit-loop-hub`, homepage = this repo's URL,
   **webhook deactivated**. Repository permissions: **Contents: read+write**,
   **Pull requests: read+write**, **Issues: read+write** (Metadata comes
   automatically). "Only on this account". If the App already exists from
   before the relay, raise **Issues** from read-only to read+write and accept
   the permission update on the existing installation — the relay cannot
   create audit issues without it.
2. Note the **App ID**, then "Generate a private key" (downloads a `.pem`).
3. Install the App on your account with **All repositories** — every current
   and future consumer repo is covered automatically; adding a project never
   requires touching credentials again.
4. Wire the hub:

```bash
gh variable set AUDIT_APP_ID           # the App's numeric ID (not secret)
gh secret set AUDIT_APP_PRIVATE_KEY < audit-loop-hub.*.private-key.pem
gh secret set AUDIT_PUSHOVER_TOKEN     # dedicated Pushover application token
gh secret set AUDIT_PUSHOVER_USER      # Pushover user key
```

Test the wiring with `gh workflow run audit-digest.yml -f force_push=true` —
`force_push` sends a push even with no news.

Digest delivery is channel-selectable via `vars.DIGEST_CHANNEL`: unset or
`pushover` (default, needs the two `AUDIT_PUSHOVER_*` secrets above) or
`slack` (needs `secrets.AUDIT_SLACK_WEBHOOK_URL` instead).

The first push to `main` skips `audit-fleet-update.yml` (job-level guard)
until `vars.AUDIT_APP_ID` is set — it shows neutral, not failed, until then.
5. **Give every routine access to the hub.** Cloud routine sessions reach
   GitHub only through a GitHub MCP server scoped to the repos in the
   routine's *sources* (no `gh` CLI — verified live 2026-07-03: the MCP's
   Actions tooling can run existing workflows but cannot send a
   `repository_dispatch`, which is why the relay is a `workflow_dispatch`).
   Each consumer routine must therefore list the hub repo
   (`<hub-slug>`) as a **second source** — without it the
   session cannot trigger the relay and the run ends with a
   relay-unreachable report instead of filing anything.

## Consumer setup

### 1. Install the plugin locally (once per machine)

```
claude plugin marketplace add <hub-slug>
claude plugin install audit-loop@audit-loop
```

This is what makes `/audit-install` (and local `/audit` runs of the plugin
itself) available in your sessions. Cloud routines never load the plugin —
they use the files vendored in step 2, merged to the default branch.

### 2. Run `/audit-install`

In a Claude Code session on the consumer repo, run `/audit-install`. It
vendors the agent and both commands into the repo's `.claude/` (hash-stamped),
writes the `audit-close.yml` workflow stub, seeds `docs/audit-baseline.md`
(header only, no entries) from this plugin's template, creates the `audit`
label, registers the repo in this hub's `consumers.txt` (which turns on
digest notifications and fleet-update PRs), gates any per-PR preview deploy
(and its teardown) in the repo's CI away from `audit-fix/**` branches — fix
PRs keep every check, they just don't spend runner minutes and hosting
compute deploying a preview nobody opens — and prints the remaining steps
below as guidance (it cannot create routines for you). It derives nothing
project-specific — nothing about the target repo's own standards docs or
domain gets written for you, that stays yours to author. If the repo has none
of the standards docs `arch-auditor` looks for, that's not caught at install
time — the first `/audit` run will flag it as finding #1 (see "How the
auditor derives 'the standard'" above).

### 3. The close workflow (written for you — nothing to wire)

`/audit-install` writes this caller stub for you. The canonical copy lives at
`examples/consumer-stubs/audit-close.yml` in this repo — read it there rather
than here to avoid drift; the excerpt below is trimmed to the `uses:` call
and omits the trigger block, permissions, and the cost-gate `if:`. It
needs no secrets beyond the default `GITHUB_TOKEN`, so there is nothing else
to configure on the consumer — notifications come from the hub's digest, not
from this repo:

**`.github/workflows/audit-close.yml`** (excerpt — see
`examples/consumer-stubs/audit-close.yml` for the full file):

```yaml
jobs:
  reconcile:
    # /audit-install substitutes the real hub slug here at install time.
    uses: HUB_SLUG/.github/workflows/audit-close.yml@main
    with:
      issue: ${{ inputs.issue }}
    secrets: inherit
```

**Not a new-install step.** This template ships no `audit-notify.yml`
reusable workflow — notifications are centralized in the hub digest. If you
are migrating an older install that still carries the
`.github/workflows/audit-notify.yml` stub and its `AUDIT_PUSHOVER_TOKEN` /
`AUDIT_PUSHOVER_USER` secrets, re-running `/audit-install` deletes the stub
and prints the secret-deletion commands.

### 4. Create the two routines

Two scheduled routines per consumer repo, both bound to that repo's default
branch. **Daily is the reference cadence** — safe because the pieces absorb
the frequency: each run with new findings files one small audit issue and
historical dedupe against all open audit issues keeps re-filed noise at zero
no matter how often `/audit` runs, `/audit-fix` no-ops unless boxes are
checked, and the digest sweeps only when a session filed something, pushing
at most once per project with news. If open audit issues ever feel noisy,
lower the routine cadence
instead — frequency lives in the routine, not in this plugin.

- **`/audit` routine** — prompt `Run /audit and let it record the
  findings.`, early morning.
- **`/audit-fix` routine** (optional, once you trust the loop) —
  prompt `Run /audit-fix and open PRs for the approved findings.`, on days
  after audit days, so boxes checked after an audit ship as PRs the next
  run. Digest sweeps need no scheduling: each session triggers its own
  sweep when it files.

Naming convention across multiple projects: `<repo> daily audit` and
`<repo> daily audit-fix`, since routines live in one flat account-wide list.
Each routine is structurally bound to its own repository, so a naming
collision is only a display problem, never a wrong-repo audit.

## What stays per-repo by design

The auditor is portable precisely because these are never part of this
plugin — they live in, and are authored by, each consumer repo:

- `docs/audit-baseline.md` — that repo's own baselined (won't-fix) findings.
- Its standards docs — root agent guide, domain glossary, design spec,
  `docs/adr/` — whatever that repo documents about itself.
- Its two routines (schedule, cadence, timezone) — frequency lives in the
  routine, not in this plugin.

Everything operational that is NOT per-repo lives in this hub: all secrets,
the notification digest, the fleet-update automation, and the `consumers.txt`
list itself.

See `docs/runbook.md` for the full operational detail: the finding format,
the issue lifecycle, the two human-owned gates, and how to handle a
conflicted or stale fix PR.
