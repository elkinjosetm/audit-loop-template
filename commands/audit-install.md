---
description: Set up this repo to consume the centralized audit-loop plugin — vendor the agent/command files with a version hash, write the close-workflow stub, seed the baseline file, create the audit label, register the repo in the hub's consumers.txt, gate any per-PR preview deploy away from fix-PR branches, and print the setup steps this command can't do itself.
---

# /audit-install

Set up **this** repo as a consumer of the centralized `audit-loop` plugin.
Idempotent: safe to run again on a repo that's already set up — every step
either no-ops, updates a plugin-managed file in place, or reports what's
already there. Re-running after a plugin update is also how a consumer repo
picks up new versions of the audit logic.

This command only touches files and `gh` state it can act on directly. It
never sets Actions secrets and never creates cloud routines — those require
access this command doesn't have (secrets are write-only, routines are an
account-level resource) — it prints exactly what to do for each in step 7.
Notification wiring is **not** part of consumer setup anymore: the hub's
digest (an on-demand sweep, not a schedule) covers every repo registered in
`consumers.txt` (step 5).

## Why vendoring (read before changing this design)

Cloud routine sessions do **not** install plugins declared in a project's
`.claude/settings.json` (external marketplace plugins require a one-time
interactive install per user, which a headless routine can never perform).
So the runtime files `/audit` and `/audit-fix` need must live **in the
consumer repo itself**. This command vendors them from the plugin and stamps
each copy with a content hash, so drift is detectable and updates are one
re-run of this command. The plugin remains the single source of truth;
vendored copies are build artifacts, never hand-edited.

## Procedure

### 1. Vendor the runtime files (with version hash)

**Resolve the hub slug `HUB` first — never hardcode it.** First match wins:

1. env `AUDIT_LOOP_HUB`, if set;
2. the marketplace source this plugin was installed from (`claude plugin
   marketplace list`, or the plugin's own metadata) — read the owning
   repo's `owner/repo` from there;
3. neither resolves it → ask the owner for the hub slug directly.

Vendor these three files from the plugin into this repo:

| Plugin source (under `${CLAUDE_PLUGIN_ROOT}`) | Vendored destination |
|---|---|
| `agents/arch-auditor.md` | `.claude/agents/arch-auditor.md` |
| `commands/audit.md` | `.claude/commands/audit.md` |
| `commands/audit-fix.md` | `.claude/commands/audit-fix.md` |

For each file:

1. Compute the source hash: `shasum -a 256 <plugin file>` — call its first
   12 hex chars `<HASH>`.
2. The vendored copy is the plugin file's content **plus one marker line
   appended at the end**:

   ```
   <!-- audit-loop:vendored sha256:<HASH> hub=<HUB> — managed by /audit-install; do not edit by hand -->
   ```

3. Idempotency / drift rules, in order:
   - Destination **doesn't exist** → write it. Report "vendored (new)".
   - Destination exists and its marker `<HASH>` equals the current plugin
     hash **and** the body (everything above the marker line) hashes to the
     same value → up to date. Report "already current", skip.
   - Marker hash **differs** from the current plugin hash, and the body
     still hashes to the marker's value (i.e. no local edits) → the plugin
     moved forward: overwrite with the new content + new marker. Report
     "updated `<old>` → `<new>`".
   - Body hash **doesn't match its own marker** (local edits), or the file
     has **no marker** (pre-existing unmanaged copy) → do **not** overwrite.
     Warn, print a diff against the plugin source, and leave reconciliation
     to the owner (they can delete the file and re-run to adopt the plugin
     version).

### 2. Write the close-workflow stub (and clean up the legacy notify stub)

Each consumer repo keeps only a thin stub that calls the reusable workflow
hosted on the hub — so the close logic lives in ONE place and every consumer
stays in sync automatically. Copy the stub with its `uses:` placeholder
substituted:

- `examples/consumer-stubs/audit-close.yml` → `.github/workflows/audit-close.yml`,
  replacing the `HUB_SLUG` placeholder in the `uses:` line with the `<HUB>`
  resolved in step 1.

Rules (compared against the **substituted** content, not the raw stub file):

- If it doesn't exist, create it.
- If it exists with **identical** (substituted) content, skip it and say so
  (already installed).
- If it exists with **different** content, do **not** overwrite it — warn
  that `.github/workflows/audit-close.yml` already exists and differs from
  the stub, print a diff, and leave it for the owner to reconcile by hand.

The stub keeps the event triggers (`pull_request: closed` /
`workflow_dispatch` with an `issue` input) — only the job body is a `uses:`
call against `<HUB>/.github/workflows/audit-close.yml@main`.
It needs no secrets beyond the default `GITHUB_TOKEN`.

**Legacy notify stub.** Notifications are now centralized in the hub's digest
(an on-demand sweep; `audit-digest.yml` on the hub) — consumers keep no
notify workflow and no Pushover secrets. If this repo still has
`.github/workflows/audit-notify.yml` from an earlier install, delete it, and
tell the owner to also delete the now-unused repo secrets:

```bash
gh secret delete AUDIT_PUSHOVER_TOKEN
gh secret delete AUDIT_PUSHOVER_USER
```

### 3. Seed `docs/audit-baseline.md`

If `docs/audit-baseline.md` already exists, skip this step entirely — never
overwrite an existing baseline, it may already hold the owner's baselined
(won't-fix) findings.

Otherwise, copy
`${CLAUDE_PLUGIN_ROOT}/templates/audit-baseline-header.md` to
`docs/audit-baseline.md` verbatim. That template is the header + entry-format
+ matching-rule section only, with **zero** entries — this repo's own
rejected findings get added here later, by hand, as `/audit` files issues and
the owner triages them.

### 4. Create the `audit` label (idempotent)

```bash
gh label create audit --description "Findings filed by the /audit command" --color "5319E7" || true
```

`gh label create` fails if the label already exists — the trailing `|| true`
is what makes this idempotent. Report whether the label was created or
already existed (check the command's own output/exit code).

The hub relay (`audit-file.yml`) also ensures this label exists on every
filing, so this step is a convenience — it just gives the first relay-filed
issue a pre-existing label — not load-bearing.

### 5. Register this repo in the hub's `consumers.txt` (idempotent)

The hub repo (`$HUB`, resolved in step 1) drives notifications
(`audit-digest.yml`) and vendored-file updates (`audit-fleet-update.yml`)
for every repo listed in its root `consumers.txt`. Register this repo there:

1. Fetch the current file:
   `gh api repos/$HUB/contents/consumers.txt --jq '.content' | base64 -d`
   (keep the `sha` from the same response — the update needs it).
2. If this repo (`owner/repo`) is already listed → report "already
   registered", done.
3. Otherwise append one line with `owner/repo` (keep comment lines and
   existing entries untouched, keep the list alphabetically sorted) and
   commit it straight to the hub's `main` via the contents API:
   `gh api -X PUT repos/$HUB/contents/consumers.txt
   -f message="Register <owner/repo> as audit-loop consumer"
   -f content="<new file, base64>" -f sha="<sha from step 1>"`.

If the `gh api` PUT fails (no push access to the hub from this environment),
don't retry blindly — fall back to printing the one-line addition as a manual
step in step 7's checklist.

### 6. Gate per-PR deploy jobs away from fix-PR branches (cost guard)

Fix PRs (head branches `audit-fix/**`) are small, human-reviewed diffs. They
must keep running the repo's full checks — that CI signal is part of the
review gate — but a **per-PR deploy** (an ephemeral preview app spun up for
every green PR) spends runner minutes and hosting compute on an environment
nobody opens, once per fix PR plus a teardown run when it closes. This step
gates those deploys away from fix PRs and leaves every check job untouched.

1. Scan this repo's `.github/workflows/*.yml` for **per-PR deploy jobs**:
   jobs on `pull_request`-family events that create an ephemeral per-PR
   resource — e.g. `superfly/fly-pr-review-apps`, or any deploy step naming
   an app/environment after the PR number — and their **teardown
   counterparts** (workflows on closed-PR events destroying that same per-PR
   resource).
2. For each deploy job found, extend its `if:` condition (preserving every
   existing clause) with:

   ```
   !startsWith(github.head_ref, 'audit-fix/')
   ```

   For teardown jobs the branch lives at
   `github.event.pull_request.head.ref` instead of `github.head_ref`. A
   skipped job bills zero minutes.
3. **Never touch build, test, lint, or any other check job.** The gate
   applies only to deploy/teardown jobs — fix PRs keep their full CI signal.
4. Idempotency and caution rules:
   - Gate already present in the job's condition → report "already gated",
     skip.
   - No per-PR deploy jobs found → report that and move on.
   - CI layout too unusual to edit with confidence (conditions that can't be
     safely extended, generated/templated workflows) → do **not** guess:
     print the exact condition to add as a manual item in step 7's checklist
     instead.
5. These are working-tree edits to **owner-managed** workflows, not
   plugin-managed files: report each edit with a diff and leave committing to
   the owner (step 7a reminds them) — the owner reviews the change like any
   other.

The gate only affects PRs opened after it lands. If earlier fix PRs left
orphaned preview apps behind, point the owner at the repo's scheduled
cleanup workflow if it has one, or at a one-off manual destroy.

### 7. Print next-steps guidance (this command cannot do these for you)

Print the following as a checklist, filled in for this repo, and stop:

**a. Commit and merge** — the vendored `.claude/` files, the workflow stub,
and any CI gates from step 6 must land on the default branch before the
routines or Actions can see them.

**b. No notification setup here** — the hub's digest (on-demand, not a
schedule) covers this repo once step 5's registration lands. All credentials
(the hub's GitHub App key, Pushover) live only on the hub repo. If this
repo still carries
`AUDIT_PUSHOVER_*` secrets from the old per-repo model, delete them (step 2
printed the commands).

**c. Two cloud routines** — create both with `/schedule` (or the routines
API), pointed at this repo's default branch, **with the hub (`$HUB`) as a
second source repo** — cloud sessions reach GitHub only through an MCP
server scoped to the routine's sources, and without the hub in scope the
session cannot trigger the filing relay:

| Routine | Cadence | Prompt |
|---|---|---|
| `<repo> daily audit` | daily, early morning | `Run /audit and let it record the findings.` |
| `<repo> daily audit-fix` | daily, late afternoon | `Run /audit-fix and open PRs for the approved findings.` |

Daily is the reference cadence: each run with new findings files one small
audit issue, and historical dedupe keeps re-filed noise at zero; `/audit-fix`
is a no-op unless the owner has checked boxes — so the daily fix routine only
ever implements what was already approved, and costs nothing when nothing is
approved. Schedule
the fix run in the late afternoon so the whole workday is the triage window
— any box checked during the day ships the same day (reference setup: audit
09:00, fix 17:00 — the digest has no hour of its own; each session triggers
a sweep as its final step). Retune frequency in the routine only,
never in the repo.

Both: model `sonnet-5`, `allowed_tools` including `Task` (both commands fan
out subagents). The routines resolve `/audit` / `/audit-fix` from the
**vendored** `.claude/commands/` files — that's why step 1 exists. After
creating each routine, strip any auto-attached MCP connectors with a
`{clear_mcp_connections: true}` update — the audit loop needs none. Routines
can only be **deleted** from https://claude.ai/code/routines — the API has
no delete endpoint.

**d. Keeping the vendored files current** — automatic: when the plugin's
runtime files change on the hub's `main`, `audit-fleet-update.yml` opens an
update PR against this repo (it skips files with local edits and lists them
in the PR body). Merging that PR is the whole update. Re-running
`/audit-install` locally after `claude plugin update audit-loop@audit-loop`
still works and produces the same result.

## Boundaries

- Never overwrites a vendored file with local edits or a pre-existing
  unmanaged file — warn with a diff and leave it.
- Never overwrites an existing, differing workflow stub — warn and leave it.
  The **only** file this command ever deletes is a leftover
  `.github/workflows/audit-notify.yml` from the deprecated per-repo
  notification model (step 2).
- Never overwrites an existing `docs/audit-baseline.md`.
- `consumers.txt` registration (step 5) is strictly append-one-line: never
  removes or rewrites other entries or comments, and never force-pushes past
  a failed contents-API call.
- Never sets or deletes Actions secrets and never creates routines — step 7
  is guidance, not action, because none of those are things this command has
  the access or the standing to do safely on the owner's behalf.
- The CI cost guard (step 6) edits **only** deploy/teardown jobs' `if:`
  conditions, never a check job, and only in the working tree — the owner
  reviews and commits. When the layout is unclear it prints guidance instead
  of editing.
- Derives nothing project-specific in what it ships: the vendored files,
  workflow stubs, baseline header, and label are identical for every
  consumer repo by design. (Step 6 necessarily adapts to the repo's own CI,
  but only ever by adding the same fixed branch condition.)
