# Audit loop — runbook

How the codebase audit loop works and how to operate it: the `arch-auditor`
subagent, the `/audit` command, the daily routine that runs it unattended,
and the `/audit-fix` command that turns approved findings into PRs.

```
daily routine ──▶ /audit ──▶ one new audit issue per run with findings
                                   │  (label `audit`, filed by the hub relay / App)
                                   ▼  owner triage: check boxes / edit Fix /
                                   │  baseline rejects
                                   ▼
daily routine ──▶ /audit-fix ──▶ one PR per approved finding, filed by the relay
                                   │  (no-op when no boxes are checked)
                                   ▼  owner reviews & merges
                 last approved fix merges ──▶ audit-close.yml closes the issue
                 (the next /audit run re-reconciles as the semantic fallback)

session ──workflow_dispatch──▶ audit-file.yml ──▶ issue/PR filed in the
                                   consumer as the App

session ──workflow_dispatch (final step)──▶ audit-digest.yml, one sweep
                                   over ALL consumers
                                   └──▶ 📱 at most one push per project with
                                        news, titled per repo, items linked
```

## How the auditor derives standards

`arch-auditor` (`agents/arch-auditor.md`) hardcodes **no project
rules**. At run time it derives "the standard" from the repo's own docs, in
priority order:

1. Root `CLAUDE.md` — hard rules (language policy, tooling, money/persistence
   conventions).
2. `CONTEXT.md` — domain glossary and domain-level invariants.
3. `docs/DESIGN_SPEC.md` — design tokens, type scale, UI conventions.
4. `docs/adr/` — durable decisions; a finding citing an ADR names the exact
   clause. Superseded ADRs don't count against code following the superseding
   one.
5. Established code patterns — a convention proven at 2+ sites can anchor a
   finding even when no doc names it (cited as `convention @ sites`).

Every finding must cite its standard; anything that can't is dropped in
verification, not reported. If a repo has **none** of those docs, the auditor
emits that as its first finding instead of inventing a standard — that is
what makes the agent file portable (see Replication below).

## Running `/audit` manually

Run `/audit` (optionally scoped: `/audit apps/web`) in a Claude Code session
on the consumer repo. It fans out five finder lenses, adversarially verifies every
candidate, suppresses baselined and already-tracked findings, prints a
prioritized report, and — only when something new survives — triggers
the hub relay, which files those findings as **one new audit issue**
for this run (label `audit`, one checkbox per finding, App-authored). Zero
new findings → nothing triggered, stated plainly in the report. The final
line of the report is either the filed issue's URL, found by polling the
hub after the trigger, or a classified timeout report — trigger accepted
but not yet confirmed — which always includes the instruction not to re-run
`/audit` until the relay run's outcome is confirmed at the linked hub
Actions run. A manual run and a routine run are the same code path, so
debugging the routine is just running the command by hand.

## The audit issue lifecycle

- **Per-run:** each run with new findings files one new audit issue via the
  hub relay (all boxes unchecked, App-authored); a run with nothing new files
  nothing. Audit issues are create-only — no later run ever edits an existing
  issue's body, checkboxes, or order, and a consumer may have several open at
  once.
- **Triage (owner):** check the box on findings to fix; edit a finding's
  **Fix** bullet first to take one of its collapsed alternatives; reject a
  finding permanently by adding a `docs/audit-baseline.md` entry.
- **Tracked:** findings in *any* open audit issue are never re-filed by later
  runs (historical dedupe). Closed issues never suppress — a finding that
  reappears after its issue closed is treated as a regression and re-filed.
- **Closed the moment the last approved fix merges** —
  `.github/workflows/audit-close.yml` fires on every merged `audit-fix/**`
  PR: if every finding in the issue is checked AND has a merged fix PR
  quoting it, the issue closes itself with a comment. Safe because both
  human gates already passed per finding (checkbox approval + PR
  review/merge); any unchecked finding blocks the auto-close.
- **Scheduled fallback:** each `/audit` run still reconciles every open audit
  issue semantically (it also sees baselined findings and manual fixes,
  which the deterministic Action can't); when everything is resolved it
  comments "this issue can be closed" for the owner to close by hand.
- **Migration:** if you are migrating an older install that still has a
  single accumulating issue open from before this per-run model, it needs no
  explicit migration step — it is simply an open audit issue to this system;
  historical dedupe suppresses against it until it drains and closes, after
  which only per-run issues exist. A fresh install never has one.
- **Cadence knob (optional):** if open audit issues ever feel noisy, lower
  the routine's cadence (weekly, biweekly) instead of changing anything
  here — frequency lives entirely in the routine.
- Don't close an issue early yourself — an unresolved remainder resurfaces
  as new findings after close (regression semantics).

## The daily routine

The routine is deliberately **dumb**: a thin scheduled trigger whose prompt
is one line. All intelligence lives in the repo's vendored copies of the
plugin's agent and commands (`.claude/agents/`, `.claude/commands/`, kept in
sync by the hub's fleet-update PRs) plus the repo's own
`docs/audit-baseline.md`, so the routine never needs updating when the audit
logic evolves, and its behavior is always reproducible by running `/audit`
manually.

### Creating it

In a Claude Code session on the consumer repo, run `/schedule` and create a routine
with:

- **Prompt:** `Run /audit and let it record the findings.`
- **Repo/branch:** this repository, default branch (`main`) — the routine
  clones the repo, so the vendored `.claude/` files written by
  `/audit-install` must be **merged** before the routine can work. (Cloud
  routine sessions do not install marketplace plugins declared in
  `.claude/settings.json` — external plugins need a one-time interactive
  install a headless session can't perform; that's why the loop vendors.)
- **Second source: the hub.** Add `<hub-slug>` as an additional
  source repo — the session's GitHub MCP scope follows the sources, and
  without the hub in scope the session cannot trigger the filing relay (see
  Requirements below).
- **Cadence:** daily — 06:00 `America/Bogota` is the reference setup, so
  triage is the first coffee of the day. Daily doesn't flood anything: each
  run files at most one small issue and dedupe re-files nothing, and the
  notification digest sweeps only when a session filed something — at most
  one push per project with news, however many
  repos or runs there are. Frequency lives in the routine, not in the repo;
  retune it (weekly, biweekly) by editing the routine only.

Requirements: the routine runs under the owner's account with the GitHub
connection. **Cloud routine environments have no `gh` CLI** — GitHub access
is a GitHub MCP server scoped to the repos in the routine's *sources*
(verified live 2026-07-03; its Actions tooling can run existing workflows
but cannot send a `repository_dispatch` — this is why the relay is a
`workflow_dispatch`). Two consequences:

- **Every routine lists two sources**: the consumer repo itself **and** the
  hub (`<hub-slug>`). Without the hub in sources, the session
  cannot trigger the relay — the run ends with a relay-unreachable report
  and nothing is filed.
- The vendored commands show `gh` invocations; in a routine the session
  fulfils the same steps through the MCP server's equivalents (list issues,
  run workflow, list workflow runs). No other credentials are needed —
  `/audit` is read-only on code and writes only the relay trigger plus the
  closability comment.

**Gotcha:** routine creation auto-attaches any MCP connectors on the owner's
account even though the audit loop needs none — strip them right after
creating the routine with a `{clear_mcp_connections: true}` update. Routines
can only be **deleted** from https://claude.ai/code/routines — the API has no
delete endpoint.

### Verifying it

After creating the routine, trigger a run (or wait for the first scheduled
one) and confirm: the run completes, and either a new `audit` issue exists or
the run log reports zero new findings. That single unattended pass is the
acceptance check for the routine.

## Notifications — the central digest

Audit issues and fix PRs are filed by the `audit-loop-hub` App through the
relay (see "The relay — how filing works" below), so the owner *does* get
GitHub's native web/email notifications for them — App activity isn't "your
own activity", so nothing is suppressed. What the hub repo's
`audit-digest.yml` adds on top is **cross-repo aggregation with per-project
delivery**: the workflow is on-demand (`workflow_dispatch` only, no crons) —
each `/audit` and `/audit-fix` session triggers one sweep as its final step,
after everything it filed through the relay has landed, so a sweep can never
run mid-filing and news is announced minutes after it exists. Each sweep
covers every repo in `consumers.txt` (news orphaned by a session that died
before dispatching is picked up by the next session's sweep, whichever repo
it audited) and sends at most **one** Pushover push per project that has
news, titled with that repo's name. Every issue/PR number in the message is
a link to the item (Pushover HTML mode), e.g.:

```
Auditoría — example-repo-a
3 hallazgos nuevos (#42 — 7 sin triage) · 1 fix PR nuevo (#57)

Auditoría — example-repo-b
2 fix PRs nuevos (#12 #13)
```

An audit-day sweep typically carries that repo's new findings; a fix-day
sweep carries the fix PRs the `/audit-fix` session just opened — announced
right when the session finishes, with no cron hour to keep phase-locked to
the routines' schedule.

The message strings above ("Auditoría", "hallazgos nuevos", "sin triage",
"fix PR nuevo", "Sin novedades…") are Spanish in this template and hardcoded
in the compute and send steps of `.github/workflows/audit-digest.yml`; edit
them there directly to localize the digest to another language.

Noise budget, by design:

- **News-only.** The digest diffs against its last snapshot
  (`state/audit-digest.json`, committed by the workflow itself): it speaks
  only when unchecked findings appeared or a fix PR opened since the last
  run. Your own triage (checking boxes, merging, closing) never triggers a
  push. Nothing new anywhere → no push at all.
- **Bounded.** At most one push per project with news per sweep, and sweeps
  happen only when a session filed something — a run that files nothing
  triggers no sweep at all.
- **Project-scoped.** Every push is titled with the repo it reports on; no
  generic pushes.
- **Priority normal, never emergency** — emergency priority stays reserved
  for anything whose weight depends on staying rare.
- Uses a **dedicated Pushover application**, so audit pushes have their own
  sound and can be muted without touching your other notifications.

**Ops note:** the native App-activity notifications (one per filed issue or
PR, per repo) are expected, not a bug — the owner can keep them, or tune
per-repo watch/notification settings if they feel redundant next to the
digest. Either way, the digest stays the canonical, bounded channel.

Setup (once, on the hub repo only — see the README's "Hub setup" for the
step-by-step): the `audit-loop-hub` GitHub App (repo access via ephemeral
per-run installation tokens; the stored key can only mint tokens) wired as
`AUDIT_APP_ID` + `AUDIT_APP_PRIVATE_KEY`, plus `AUDIT_PUSHOVER_TOKEN` and
`AUDIT_PUSHOVER_USER`. Consumers need **no** notification setup at all —
being listed in `consumers.txt` is the whole subscription. Test with
`gh workflow run audit-digest.yml -f force_push=true` in the hub —
`force_push` pushes even with no news.

This template ships no per-repo notification model: there is no
`audit-notify.yml` reusable workflow and no per-consumer Pushover secrets —
notifications are centralized in this digest from the start. If you are
migrating an older install that still carries the leftover
`audit-notify.yml` stub and its two Pushover secrets, re-running
`/audit-install` on it deletes the stub and prints the secret-deletion
commands.

## The relay — how filing works

Consumer sessions never run `gh issue create` or `gh pr create` — there is
**no** fallback to filing that way, ever, even when the relay itself is
unreachable. Filing is always create-only, done by the hub, through one
trigger contract:

- **Trigger.** `/audit` (survivors > 0) and `/audit-fix` (per approved
  finding) each trigger the hub's `audit-file.yml` via `workflow_dispatch`
  (`gh workflow run` where `gh` exists; the GitHub MCP server's workflow-run
  tool in cloud routines — the transport every session type can use, since
  the MCP tooling cannot send a `repository_dispatch`). The inputs carry
  `kind` (`issue` | `pr`), the target repo, a fresh run **marker** (a
  lowercase UUID), the title/body, and — for PRs — the already-pushed head
  branch and base. A size guard drops whole trailing (least severe) findings
  from an oversized issue body rather than truncating mid-finding; a
  still-oversized or refused trigger is a relay failure, not a retry.
- **Validation and authorship.** The hub's `audit-file.yml` workflow
  (`.github/workflows/audit-file.yml`) validates the inputs, rejects any
  `target_repo` not listed in `consumers.txt` (registration is the
  membership — the same gate as everywhere else in this system), mints an
  ephemeral App installation token scoped to that one repo, and creates the
  issue (ensuring the `audit` label exists) or the PR. Everything filed
  through the relay is authored by the `audit-loop-hub` App, never by the
  owner's own token.
- **Marker and poll-back.** The hub stamps the marker into the created
  item's body as an HTML comment. The triggering session polls (every 10 s,
  up to 3 minutes) for an item containing its marker and reports that URL as
  the final line of its output. A poll timeout doesn't mean failure — the
  session instead reads the hub's `audit-file.yml` run list and reports a
  classified outcome (rejected / still pending / no run seen), always with
  the instruction not to re-run `/audit` or `/audit-fix` until that run's
  outcome is confirmed, since a second trigger could double-file.
- **Preflight (`/audit` only).** Before triggering a new issue, `/audit`
  checks for any non-completed `audit-file.yml` run and waits for it to
  clear (re-suppressing against a freshly re-fetched issue list if one just
  finished) before building its own trigger — this closes the small window
  where a just-filed issue hasn't landed yet and a second run could re-file
  the same findings.

**Troubleshooting a missing filed item:** check the `audit-file` runs at
`https://github.com/<hub-slug>/actions/workflows/audit-file.yml`.
A run with `conclusion: failure` means the relay rejected the inputs — most
commonly the target repo isn't (yet) registered in `consumers.txt`, or an
input failed validation. A run that's `queued`/`in_progress` means it's
still pending; give it a few more seconds and re-check. A pushed
`audit-fix/*` branch with no PR anywhere (open or merged) means a previous
run's relay filing failed after the branch push — the next `/audit-fix` run
recovers it automatically, reusing the branch and pushing with
`--force-with-lease`.

## Fleet updates — how consumers stay current

The vendored `.claude/` copies in each consumer are build artifacts of this
plugin. When `agents/arch-auditor.md`, `commands/audit.md`, or
`commands/audit-fix.md` change on the hub's `main`, `audit-fleet-update.yml`
re-vendors them into every repo in `consumers.txt` and opens one update PR
per repo that is behind — merging that PR is the whole update. Operational
notes:

- The workflow applies the same hash/marker rules as `/audit-install`: a
  vendored file with local edits (body no longer matches its marker) is never
  overwritten — the PR body lists it under "Not touched". Reconcile by hand,
  or delete the file and re-run `/audit-install` to re-adopt the managed copy.
- One branch per hub commit (`audit-loop-update/<sha>`); re-runs are
  idempotent (an existing open update PR is skipped).
- Adding a repo to `consumers.txt` and running the workflow manually
  (`gh workflow run audit-fleet-update.yml`) brings it up to date on demand.

## Conflicted or stale fix PRs — regenerate, don't rebase

Fix branches are cut from fresh `main`, but a PR can conflict later (you
merged a sibling PR touching the same file, or `main` moved on). There is no
conflict-resolver agent, deliberately: resolving a conflict means making merge
judgments the loop reserves for humans. Instead, fix PRs are **regenerable**:

- **Close the conflicted PR without merging** (delete its branch). The box
  stays checked, and a closed-unmerged PR does not count as resolved — so the
  next `/audit-fix` run re-implements the finding from current `main`: a
  fresh, conflict-free PR replaces manual rebase surgery.
- **Trivial conflict and you want it now** → resolve in the GitHub UI like any
  PR; both paths are fine.
- The triage semantics this defines: *close the PR* = "regenerate it";
  *uncheck the box (or baseline the finding)* = "abandon it".

## CI minute conventions — the `[ci-lite]` marker and the close-stub gate

Two portable, opt-in conventions let a consumer's CI spend fewer runner
minutes on fix PRs without weakening either human gate. Both are shipped in
the vendored files, so nothing in a consumer needs hand-wiring beyond opting
its own CI in.

- **The `[ci-lite]` marker (emitted by `/audit-fix`).** When a fix PR's diff
  is **entirely non-behavioral** — only code comments, documentation, or
  user-facing string text with no logic change — `/audit-fix` adds the literal
  marker `[ci-lite]` on its own line in the PR body it files through the relay.
  The command emits it conservatively: any doubt that the diff is truly
  non-behavioral, and the marker is omitted; risk-critical fixes (money / auth
  / send) never carry it. A consumer whose CI opts into the convention can read
  the marker off the PR body and run a **reduced pipeline** for that PR
  (skipping heavy check jobs it deems unnecessary for a text-only change);
  a consumer that ignores the marker runs its full pipeline unchanged, so
  emitting it is always safe. Opting in is the consumer's own CI decision —
  the loop only guarantees the marker's presence and its meaning, never which
  jobs a consumer skips. Because a required check skipped this way still counts
  as satisfied for merging, the review gate is untouched.
- **The close-stub branch gate.** The consumer's `audit-close.yml` stub (from
  `examples/consumer-stubs/audit-close.yml`, written at install time) carries a
  job-level `if:` that only reaches the reusable close workflow for fix PRs
  (head branch `audit-fix/**`) or a manual `workflow_dispatch`. On every other
  closed PR the stub's job is skipped and bills **zero** minutes, so unrelated
  PR-close events never dispatch the close logic. The reusable workflow
  re-checks the same branch (and `merged == true`) internally — the stub gate
  is defense in depth and a cost guard, not a change to which audit issues
  auto-close. This ships to **future installs only** (templates ship at install
  time — see the note below). If you are migrating an older install whose
  stub predates this gate, re-run `/audit-install` (or apply the
  fleet-update PR) to pick it up — it never updates automatically.

## Replication — installing the loop in another project

The loop's agent and command logic is centralized in the `audit-loop`
Claude Code plugin (`<hub-slug>`); nothing about installing it
into a new repo is project-specific:

1. Install the plugin locally, once per machine:
   `claude plugin marketplace add <hub-slug>` then
   `claude plugin install audit-loop@audit-loop`.
2. Run `/audit-install` in the target repo. It vendors the agent and the
   `/audit` + `/audit-fix` commands into the repo's `.claude/` (stamped with
   a version hash so updates and drift are detectable), writes the
   `audit-close.yml` workflow stub, seeds `docs/audit-baseline.md` from the
   plugin's template, creates the `audit` label, registers the repo in the
   hub's `consumers.txt` (digest notifications + fleet-update PRs), gates
   any per-PR preview deploy (and its teardown) in the repo's CI away from
   `audit-fix/**` branches — fix PRs keep every check but don't spend deploy
   minutes on a preview nobody opens — and prints the steps below that it
   cannot do for you.
3. Credentials: nothing to do. The hub's GitHub App is installed with "All
   repositories", so a new repo is covered the moment it exists. (If the
   installation was narrowed to selected repos, add the new one at
   github.com/settings/installations.) No secrets are added to the consumer
   itself, ever.
4. Create the 2 cloud routines, pointed at the target repo's default
   branch: an `/audit` routine and an `/audit-fix` routine. **Daily is the
   reference cadence** (see "The daily routine" above) — e.g. audit at
   06:00 `America/Bogota`, fix daily as well so boxes checked after an audit
   ship as PRs the next fix run. If GitHub Actions minutes are a concern, an
   alternating cadence is a budget-conscious retune, not the reference — e.g.
   3 audit days + 3 fix days a week (America/Bogota, no DST): audit
   `0 14 * * 1,3,5` UTC (09:00 Mon/Wed/Fri), fix `0 14 * * 2,4,6` UTC (09:00
   Tue/Thu/Sat). The digest needs no cron of its own: each session triggers a
   sweep as its final step, and each sweep pushes per project with news.
   Model `sonnet-5`, `allowed_tools` including `Task` (both commands fan out
   subagents).

   **Gotcha:** routine creation auto-attaches any MCP connectors on the
   owner's account even though the audit loop needs none — strip them
   right after creating each routine with a `{clear_mcp_connections: true}`
   update. Routines can only be **deleted** from
   https://claude.ai/code/routines — the API has no delete endpoint.

Check the target repo's standards docs after installing: the auditor reads
root `CLAUDE.md` / `CONTEXT.md` / a design spec / `docs/adr/` (adjust
nothing — whatever exists is the standard; if nothing exists, the first
audit run will tell you so as finding #1, and writing those docs becomes the
first fix). Run `/audit` manually once to validate signal quality; tune by
writing docs/ADRs (never by editing the agent), and baseline what you
reject.

Naming convention for routines: routines live in one flat account-wide list,
so the repo prefix is the identifier — `<repo> daily audit` and
`<repo> daily audit-fix`. Each routine is also structurally bound to its own
`git_repository`, so a collision is a display problem, never a wrong-repo
audit. Triage never piles up across projects: whatever accumulates lands in
a handful of small per-run audit issues per repo and one digest line per
day.

## `/audit-fix` — implementing approved findings

`/audit-fix` (`commands/audit-fix.md`) is the self-improving half of the
loop: it turns **owner-approved** audit findings into pull requests. It is the
step _after_ `/audit` has filed an issue and _after_ the owner has triaged it.

### The two gates

The loop has two human-owned gates; `/audit-fix` sits between them and bypasses
neither.

1. **Triage gate (approval to touch a finding).** The audit issue lists each
   finding as a checkbox. **Checking the box is the approval.** `/audit-fix`
   acts on **only** the checked (`- [x]`) findings and never touches unchecked
   (`- [ ]`) ones. So the owner's job before running the command is simply: open
   the open `audit` issue(s) — approvals may live across several — check the
   boxes worth fixing, leave the rest unchecked.
2. **Review gate (approval to merge a fix).** Every fix ships as a **pull
   request** against `main`. `/audit-fix` **never merges** and **never pushes to
   `main` directly** — a human reviews and merges each PR.

### Usage

1. Open any open `audit`-labeled issue(s) and check the box on every finding
   you want fixed. To take a finding's collapsed **alternative** instead,
   replace its **Fix** bullet with the alternative before checking. Leave
   anything you don't want touched unchecked (or baseline it in
   `docs/audit-baseline.md` if it should never be re-reported).
2. Run the command:

   ```
   /audit-fix
   ```

   It defaults to sweeping **all open** `audit` issues, oldest first — checked
   boxes are honored wherever they live, so approvals in older issues never
   rot. To restrict it to a single issue, pass its number: `/audit-fix 123`.
3. The command opens **one PR per checked finding** (grouping only when two
   findings are literally the same fix), each on its own branch cut fresh from
   `main` and each implemented by its **own fresh-context subagent** (opus for
   money/auth/send fixes, sonnet otherwise) — so a mass triage after weeks
   away doesn't degrade as one context fills up. It reports the PR URLs plus
   an issue-status snapshot (how many findings are merged / in review /
   undecided — point-in-time; the next `/audit` run keeps the durable
   reconciliation).
4. Review each PR and merge the ones you want. Merging is entirely manual.

### What each PR contains

- A **non-auto-closing link** to the audit issue (`Audit finding from #<N>` — it
  never uses `Closes/Fixes #<N>`, because one issue holds many findings and must
  not be closed by a single fix).
- The **finding quoted verbatim**, so a reviewer can compare the approved
  finding to the actual diff.
- A **scope statement** — the diff stays within what the finding describes.
- For fixes touching **money, auth, or an outbound send path**, a prominent caution flag
  at the top of the body. These are implemented like any other fix but flagged
  for careful human review — the target repo's own standards docs (ADRs,
  CLAUDE.md/CONTEXT.md) define exactly which paths and rules apply here.

### Guardrails (what it will not do)

- Never implements an **unchecked** finding.
- Never **merges** anything and never **pushes to `main`**.
- Keeps each PR's diff **scoped** to its finding; if a fix can't be done in
  scope — or a Fix bullet still contains more than one action — it stops that
  finding and reports it for re-triage rather than guessing.
- Never edits or closes the audit issue itself.
- Stops on a dirty working tree rather than disturbing uncommitted work.

### Optionally scheduling `/audit-fix` as a second routine

Once you trust the loop, `/audit-fix` can be scheduled as a **second** daily
cloud routine that runs **after** triage — the mirror of the `/audit` routine.
The trade-off to understand before scheduling it: `/audit` recording findings
is safe to automate (it only writes a GitHub issue), but `/audit-fix` writes
code and opens PRs, so an automated run is only as safe as your triage
discipline. It stays safe because the two gates still hold under automation:

- The **triage gate is still fully manual** — a scheduled run acts on exactly
  the boxes you checked, and does nothing if you checked none. So the routine
  is only ever "implement what I already approved", and on days with no
  approvals it is a no-op.
- The **review gate is still fully manual** — the routine opens PRs; it never
  merges. You review and merge by hand regardless.

Create it exactly like the `/audit` routine (see "The daily routine" above)
with the prompt `Run /audit-fix and open PRs for the approved findings.`,
scheduled in the late afternoon so the whole workday is the triage window:
the audit routine files any new findings and its sweep notifies → you check
boxes during the day → the fix routine ships the approved ones (or you run
it by hand) and its own sweep announces the new PRs → you review and merge.
No fixed digest hours — each sweep runs as the last step of the session that
triggers it.
