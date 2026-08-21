---
description: Implement owner-approved audit findings as scoped pull requests — one PR per checked finding, never merges, flags money/auth/send-path fixes.
argument-hint: "[optional: audit issue number to act on; defaults to sweeping ALL open `audit`-labeled issues, oldest first]"
---

# /audit-fix

Turn **owner-approved** audit findings into pull requests. Sweep **all open**
`audit`-labeled issues (or only the issue number in `$ARGUMENTS`), act on
**only the findings whose checkbox is checked** — wherever they live — and
open **one scoped PR per approved finding**. This is the self-improving half
of the audit loop. A checked box is an approval regardless of which issue it
sits in or how old it is; the sweep exists so approvals in older issues never
rot unnoticed.

**Two gates, both human-owned — never bypass either:**

1. **Triage gate (input):** the owner checks a finding's box in the audit issue.
   That check is the sole authorization to touch that finding. An **unchecked**
   box means "not approved" — never implement, branch, or PR an unchecked
   finding, even if it looks trivial or correct.
2. **Review gate (output):** every fix ships as a PR against the default branch.
   This command **never merges** and **never pushes to the default branch
   (`main`) directly.** Human review + merge of the PR is the second gate.

Run this manually. (Converting it into a scheduled post-triage routine is a
later, documented step — see `docs/runbook.md` — not something this command
does on its own.)

## Hub resolution

Resolved once, up front, before anything else in this pipeline runs — every
later step that names "the hub" or `$HUB` means the slug resolved here.
Resolve `HUB` deterministically, first match wins:

- env `AUDIT_LOOP_HUB`, if set;
- the `hub=` field of the vendored marker on the last line of this repo's
  `.claude/commands/audit.md`;
- neither present → **fail loud before any filing step**: report that the
  vendored copies predate the hub-aware stamp, instruct the owner to re-run
  `/audit-install` (or set `AUDIT_LOOP_HUB`), and file nothing. Never guess a
  slug, never sniff remotes for it.

## Pipeline at a glance

```
select ALL open `audit` issues, oldest first (or only $ARGUMENTS)
  → parse ONLY checked (- [x]) findings from every selected issue
  → group findings that are literally the same fix
  → per group: ONE fresh implementer subagent → branch off main →
    implement (scoped to the finding) → verify (typecheck/lint/tests) →
    commit → push branch → open 1 PR
  → report: one line per PR opened, plus skipped/unchecked counts
```

If **no** boxes are checked, this command does nothing but say so — that is a
valid, correct outcome.

## Procedure

### 0. Preflight

- Confirm `gh` is authenticated and the working tree is **clean**
  (`git status --porcelain` empty). If dirty, stop and report — do not stash or
  discard the owner's uncommitted work.
- Identify the repository's default branch (`main` for this repo). Every fix
  branch is cut fresh from an up-to-date default branch so PRs are independent
  and mergeable in any order:

  ```bash
  git fetch origin
  DEFAULT=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
  ```

### 1. Select the audit issues

If `$ARGUMENTS` is a number, act on that single issue. Otherwise sweep **every
open** `audit`-labeled issue, **oldest first** (FIFO — older approvals get
implemented before newer ones):

```bash
gh issue list --label audit --state open --json number,title,url,body,createdAt \
  --limit 200 --jq 'sort_by(.createdAt)'
```

Confirm each issue actually carries the `audit` label before acting on it.
Fetch full bodies — you will parse the checkbox findings out of them and quote
them verbatim in PR bodies, so keep the exact text. Issues with zero checked
boxes are skipped silently (they still count in the report's tally). Steps 2–4
below apply per finding, across all selected issues; every branch name and PR
reference uses the issue the finding actually came from.

### 2. Parse only the checked findings

Audit issues use the checkbox format written by `/audit`
(`commands/audit.md`, step 7):

```markdown
- [ ] **[<severity>]** <plain-language title>
  `<path>:<line>`

  - **What:** <what the code does>

  - **Standard:** <doc heading, or ADR-NNNN §clause, or convention @ sites>

  - **Fix:** <concrete change>

    <details><summary><em>Alternatives considered</em></summary> …optional… </details>
```

(The blank lines between sub-bullets are part of the format — tolerate them
when parsing. When quoting a finding verbatim in a PR body, prefix **every**
line of the item with `> `, including the blank ones, so the blockquote stays
one block.)

Select **only** the list items whose box is checked — `- [x]` or `- [X]`.
Everything with `- [ ]` is **out of scope**: it is not approved and must not be
touched. Extract for each approved finding: severity, title, `path:line`, the
**What**/**Standard**/**Fix** sub-bullets, and the finding's **verbatim
markdown** (all lines of the list item, including its nested bullets, for
quoting later — the `<details>` block is excluded from the quote).

**The Fix bullet is the entire mandate.** A finding may carry a collapsed
`<details><summary><em>Alternatives considered</em></summary>` block under its
Fix — **ignore it entirely**: alternatives are triage-time context for the
owner, who takes one by replacing the Fix text before checking the box. Never
implement an alternative, never blend it into the fix. And if the Fix bullet
itself still contains more than one action ("do X, or Y") — a multi-action
Fix that escaped `/audit`'s filing gate — that finding is ambiguous —
**skip it, open no branch, and report it as needing re-triage**; never pick
among options yourself.

**Option findings — `Fix (choose one)`.** An option finding carries, instead
of a plain Fix, a **Fix (choose one)** bullet with nested option checkboxes
(one complete fix per option; format defined in `commands/audit.md`, step
7b). The nested option boxes are triage controls, never findings — only
**top-level** checkbox items are findings, so an option box never enters the
approval scan above on its own. For a **checked** finding that carries an
option block:

- **exactly ONE option checked** → that option's text is the Fix mandate.
  Implement it verbatim, and quote the finding in the PR with the option
  block intact — checked box and all — so the reviewer sees precisely which
  option was approved.
- **ZERO options checked** → the approval is incomplete: the owner approved
  the finding but never picked its fix. Skip it, open no branch, and report
  it as `approved but no option chosen — check exactly one option box in
  #<N>`.
- **SEVERAL options checked** → contradictory. Skip it, open no branch, and
  report it as needing re-triage (uncheck all but one option).

**Filter already-resolved approvals (idempotency).** A checked box stays
checked forever, and the sweep revisits old issues every run — so before
grouping, drop any approved finding that is already handled:

- a fix PR quoting this finding already exists — **merged** (done) or **open**
  (in review; opening a second PR would duplicate it). Find them with
  `gh pr list --search "Audit finding from #<N>" --state all` and match by the
  quoted finding, not just the issue number. A **closed-but-not-merged** fix
  PR does NOT count as resolved: closing a fix PR while leaving the box
  checked is the owner's "regenerate this" signal (e.g. the PR went stale or
  conflicted as `main` moved) — re-implement it from fresh `main`. An owner
  who wants a finding *abandoned* unchecks the box or baselines it instead.
  A PR filed through the hub relay minutes ago may not be indexed by search
  yet — when the candidate branch `audit-fix/issue-<N>-<slug>` already exists
  on the remote, additionally run a head lookup
  (`gh pr list --head "audit-fix/issue-<N>-<slug>" --state all`) before
  deciding the finding is unimplemented.
- the violation no longer exists in code — read the cited `path:line` first;
  if the described problem is already fixed (manually, or as a side effect of
  other work), there is nothing to implement.
- the finding is baselined in `docs/audit-baseline.md` (matched by the
  baseline's own three-part rule: same path, same standard, same underlying
  violation). Baselining is the abandon signal and it wins over a
  still-checked box — the owner may baseline without ever revisiting the
  issue to uncheck.

**Orphaned-branch recovery.** If the remote branch already exists but the
search *and* the head lookup both come up empty (no PR, open or merged, quotes
this finding), a previous run's relay filing failed after the branch push.
Recover instead of skipping: reuse the same branch name, re-implement the
finding from current `main`, and finish that group's push with
`git push --force-with-lease origin HEAD` in place of a plain push (the stale
remote branch would otherwise reject a non-fast-forward). Never delete or
force-push over a remote branch that already has an open PR.

Skipped-as-resolved findings are reported in step 5 (`already merged` /
`in review` / `already fixed in code` / `baselined`), never re-implemented. This is what
makes the sweep safe to run repeatedly — including from a schedule.

If zero findings are checked **across all selected issues**, skip to step 5 and
report "no approved findings — nothing to do." Do not open any branch or PR.

### 3. Group findings that are literally the same fix

Default: **one branch + one PR per approved finding.** Merge two (or more)
approved findings into a single PR **only** when they resolve to the *same
edit* — e.g. the same duplicated helper appears twice and both boxes point at
extracting it once, or two lines in one file that a single change fixes
together. When in doubt, keep them separate: separate PRs are easier to review
and revert. Grouping for convenience or to "save PRs" is not allowed — the test
is "is this literally one fix?", not "are these related?".

### 4. For each group: one subagent — branch → implement → verify → commit → push → PR

**Never implement in your own context.** Delegate each group to a fresh
implementer subagent (one `Task` call per group); you — the orchestrator —
only brief, sequence, and report. One finding = one agent = one clean context
window: a mass-triage sweep (weeks of accumulated issues, many boxes checked
at once) must not degrade as a single context fills up, and finding B's
implementer never sees finding A's diff.

Run the agents **sequentially, one at a time** — they share the working tree,
and each must start from (and return to) a clean default branch so branches
never stack on one another.

**Model tier per agent:** finding classified risk-critical (money / auth /
send — see Risk flagging below) → `opus`; everything else (code or docs) →
`sonnet`. Never below sonnet: the agent writes code and opens a PR.

**Brief each agent completely** — it starts with zero context. Include: the
finding's verbatim markdown (its Fix bullet is the entire mandate), the source
issue number, the branch name to use, your risk classification (and the
caution block if it applies), the PR body template below, the resolved hub
slug substituted **literally** into every command in the briefing (never left
as `$HUB` for the subagent to resolve), the relay trigger and poll-back
procedure (marker generation, the `kind=pr` workflow inputs, the size guard,
and the classified timeout/relay-failure report), the
`--force-with-lease` rule for orphaned-branch recovery above, and steps a–g as
its contract. Require it to return **either** the PR URL found by poll-back
**or** a classified trigger-accepted report (marker, branch name, hub run
status) **or**, if it stopped for its own reasons (scope growth, ambiguous
Fix, breakage it didn't cause), a re-triage report and **no branch left
behind**. Verify the returned PR exists — **or** accept a trigger-accepted
report as a valid terminal outcome; a subagent that errored mid-way must leave
the tree clean. A pushed branch with a pending relay run is not an error.

Steps each implementer agent follows:

**a. Cut a fresh branch off the default branch:**

```bash
git switch "$DEFAULT" && git pull --ff-only
git switch -c "audit-fix/issue-<N>-<short-slug>"
```

Use a short slug derived from the finding (`<N>` = audit issue number).

**b. Implement — and stay inside the finding's scope.** Make exactly the change
the finding's `fix:` describes, at the `path:line` it cites. Do **not**:

- fix other findings (checked or not) in the same PR,
- refactor, reformat, or "tidy" code the finding does not name,
- expand beyond the file(s)/symbol(s) the fix requires.

The PR diff must be explainable as "this and only this finding." If honoring the
fix genuinely requires a broader change than the finding describes, **stop that
finding**, leave no branch, and report it as needing re-triage — do not silently
grow the diff. Follow the repo conventions the finding cites — its cited
standard (a doc heading, ADR clause, or established convention) is the source
of truth for what "correct" looks like in this repo; never import a
convention from another project.

**c. Verify before committing.** Run the repo's own test scripts/CI
conventions and fix what your change broke — typecheck, lint, and tests, using
however this repo's own docs / `package.json` / CI config define them (do not
assume a generic default command is correct, especially in a monorepo that
routes to more than one runner). If a check fails for a reason your diff did
not cause, note it in the PR body rather than expanding scope to fix unrelated
breakage.

**d. Commit** with an English message following the repo convention (imperative
subject, why-not-what body when non-obvious), ending with the trailer used
across this repo:

```
Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
```

**e. Push the branch — never the default branch:**

```bash
git push -u origin HEAD
```

**f. File exactly one PR through the hub relay** — never `gh pr create`, with
no fallback if the relay is unreachable. The relay is the hub's
`audit-file.yml` workflow, triggered via `workflow_dispatch`. Build the body
exactly as described in "PR body template" below, write it to a scratch file,
then:

1. Generate a fresh marker: `MARKER=$(uuidgen | tr 'A-Z' 'a-z')`.
2. Derive the target repo from git:

   ```bash
   TARGET=$(git remote get-url origin \
     | sed -E 's#^(git@github\.com:|https://github\.com/)##; s#\.git$##')
   ```

   If the body scratch file is larger than 60,000 bytes, this is a bug (PR
   bodies aren't expected to approach the cap) — trim only by shortening
   generated prose, never by cutting the finding quote or the scope line.
3. Trigger the relay:

   ```bash
   gh workflow run audit-file.yml -R "$HUB" --ref main \
     -f kind=pr -f target_repo="$TARGET" -f marker="$MARKER" \
     -f title="audit-fix: <short finding summary>" \
     -f head="audit-fix/issue-<N>-<short-slug>" -f base="$DEFAULT" \
     -F body=@<scratch-path>
   ```

   `-F body=@<file>` reads the body from the file — never build it as an
   inline shell string. If this environment has **no `gh`** (cloud routines
   reach GitHub only through a GitHub MCP server), trigger the same workflow
   with that server's workflow-run tool instead: repo `$HUB`,
   workflow `audit-file.yml`, ref `main`, same inputs (`kind=pr`,
   `target_repo`, `marker`, `title`, `head`, `base`, `body` — body passed
   whole). This requires the hub repo to be in the session's GitHub scope;
   routines list the hub as a second source (see the hub's runbook).

   If the trigger itself is refused (non-zero `gh` exit, MCP tool error, or
   the hub is not in this session's GitHub scope), **do not** fall back to
   `gh pr create`. Report: the pushed branch name, the finding it implements,
   and `Relay unreachable (<exact error>) — nothing was filed. The branch is
   pushed; open the PR by hand from it, or re-run once hub access is
   restored. If this persists, investigate hub access before the next run.`
   Then stop this finding — no PR, no further retries of the same trigger.
4. On an accepted trigger, poll every 10 seconds for up to 3 minutes (18
   attempts), by head branch (not the search index, which lags):

   ```bash
   gh pr list --head "audit-fix/issue-<N>-<short-slug>" --state all \
     --json url,body --limit 10 \
     --jq ".[] | select(.body | contains(\"audit-loop:run $MARKER\")) | .url"
   ```

   (No `gh`? Same poll via the GitHub MCP server: list this repo's PRs by
   that head branch and match the marker string in their bodies.)

   The first non-empty result is this finding's PR URL — return it.
5. **On timeout** (no result after 3 minutes): do not retry the trigger (a
   second one could double-file). Classify the outcome with this
   environment's GitHub tooling (`gh` shown; MCP equivalent: list this
   workflow's recent runs on the hub):

   ```bash
   gh run list -R "$HUB" --workflow audit-file.yml --limit 5 \
     --json status,conclusion,url,createdAt
   ```

   Report, verbatim in structure: the trigger was accepted and the
   marker; then, if the most recent matching run **failed** — the relay
   rejected the inputs, include the run URL, and say the PR will not appear;
   if a run is **queued/in_progress** — the relay is genuinely pending, the PR
   will likely still appear, include the run URL; if **no run is visible** —
   say the trigger may not have fired and point to
   `https://github.com/$HUB/actions/workflows/audit-file.yml`.
   In every case, report the pushed branch name so the owner can open the PR
   by hand, and add: **"Do not re-run `/audit-fix` for this finding until the
   relay run's outcome is confirmed at the link above — a re-run before then
   can double-file."**

**Never** merge the resulting PR, never enable auto-merge, never push to
`main`.

**g. Return to the default branch** (`git switch "$DEFAULT"`) before the next
group.

### 5. Report (in conversation)

Print one line per PR opened (`#<finding> → <PR url>`), grouped by source
issue, then a tally per swept issue plus a total:

```
issue #<N₁>: 2 PRs opened (3 findings checked; 2 implemented, 1 grouped)
issue #<N₂>: 1 PR opened (1 finding checked); issue #<N₃>: skipped (0 checked)
Total: 3 PRs; 0 unchecked findings touched. Nothing merged.
```

Always state explicitly that unchecked findings were left untouched and nothing
was merged.

**Then trigger the digest sweep — only if this run filed at least one PR.**
The hub's digest (`audit-digest.yml`) is on-demand: no schedule, so the owner
hears about the new fix PRs only when a session tells it to sweep. Trigger it
once for the whole run (never per PR), after every group's relay poll-back
has resolved, fire-and-forget — no inputs, no polling:

```bash
gh workflow run audit-digest.yml -R "$HUB" --ref main
```

(No `gh`? Same trigger via the GitHub MCP server's workflow-run tool: repo
`$HUB`, workflow `audit-digest.yml`, ref `main`, no
inputs.) If the trigger is refused, just report it: nothing is lost — the
next session's sweep announces these PRs. A run that filed nothing triggers
nothing.

End with an **issue-status snapshot per swept issue**: count every finding in
the issue and where each stands — fix PR merged, fix PR open (from this run or
a prior one; find them with
`gh pr list --search "Audit finding from #<N>" --state all`), baselined in
`docs/audit-baseline.md`, or still undecided (unchecked). Two shapes:

```
All 3 findings of #<N> are merged or baselined — #<N> can be closed.
#<N> not closable yet: 1 finding undecided, 2 fix PRs awaiting review.
```

This line is a point-in-time snapshot — PRs merged after this run won't update
it. The next `/audit` run re-reconciles every open audit issue and comments
on the issue itself when it becomes closable.

## PR body template

The body is multi-line markdown — build it in a scratch file first; step 4f
passes that file whole as the relay's `body` input (`-F body=@<file>` or the
MCP equivalent), it is never passed to `gh pr create` (that command is
retired — every PR is filed through the hub relay).

Body contents (in this order):

1. **Link, do not auto-close, the audit issue.** Reference it as
   `Audit finding from #<N>` — **never** use `Closes #<N>` / `Fixes #<N>` /
   `Resolves #<N>`. The audit issue holds many findings; auto-closing it when
   one finding's PR merges would drop every other finding on the floor.
2. **Risk flag** (see below) when applicable — immediately under the
   audit-issue link so it is impossible to miss.
3. **Quote the finding verbatim** as a blockquote — the exact checkbox item
   (severity, title, `path:line`, What/Standard/Fix bullets) so a reviewer
   sees precisely what was approved and can compare it to the diff.
4. **Scope statement:** one line asserting the diff is limited to this finding.
5. **CI-lite marker (only when the whole diff is non-behavioral).** If the
   fix PR's diff changes **nothing that executes** — only code comments,
   documentation, or user-facing string text with no logic change — add the
   literal marker `[ci-lite]` on **its own line** in the body. A consumer
   whose CI opts into the convention reads this marker and runs a reduced
   pipeline for that PR (skipping heavy check jobs it deems unnecessary for a
   text-only change); a consumer that ignores it runs its full pipeline
   unchanged, so the marker is always safe to emit. **When in ANY doubt that
   the diff is non-behavioral — a touched line that could alter runtime
   behavior, a comment that is actually a directive (a pragma, an annotation,
   a config value read at runtime), an "obviously cosmetic" edit you are not
   certain about — do NOT add the marker.** Omitting it costs only CI minutes;
   adding it wrongly could let a behavioral change skip its checks. A
   risk-critical fix (money / auth / send — see below) is behavioral by
   definition and never carries the marker.

Skeleton:

```markdown
Audit finding from #<N> (do not auto-close — that issue tracks multiple findings).

<!-- risk flag block goes here when applicable -->

> - [x] **[<severity>]** <plain-language title>
>   `<path>:<line>`
>
>   - **What:** <what the code does>
>
>   - **Standard:** <cited standard>
>
>   - **Fix:** <concrete change>

Scope: this PR changes only what the finding above describes.

[ci-lite]  <!-- ONLY when the whole diff is non-behavioral; omit otherwise -->

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Risk flagging — money / auth / send paths

Some fixes touch code the project treats as high-consequence. **Implement them
the same way, but flag them prominently** so the human reviewer knows to look
hard. A finding is risk-critical when its `path` or cited `standard` touches:

- **Money / computation** — amounts, rates, cents, totals, consecutive
  numbering, or any other computation the target repo's own docs (CLAUDE.md /
  CONTEXT.md / ADRs) treat as money-critical.
- **Auth / session / credentials** — login, session policy, credentials,
  secrets, exposure boundary, or any other path the target repo's own docs
  treat as auth-critical.
- **Send / notify path** — an irreversible send/notify workflow (email, SMS,
  push, webhook, etc.) or its sending-state machine, wherever the target
  repo's own docs define one.

Derive the concrete paths and the specific ADR numbers/clauses for each
category from the target repo's own standards docs at run time — never
hardcode one project's paths or ADR numbers here.

For any such PR, put this block near the **top** of the body (right under the
audit-issue link), so it is impossible to miss:

```markdown
> [!CAUTION]
> **Risk-critical fix — review carefully.** This change touches the
> <money | auth | send> path. Verify money/auth correctness and, if this is a
> send/notify path, that it cannot risk a duplicate or wrong send before
> merging.
```

Name which path(s) it touches and which of the target repo's own ADR/clause it
cites. When unsure whether a fix is risk-critical, treat it as if it is and add
the flag.

## Boundaries

- **Never runs `gh pr create`.** Every PR is filed through the hub relay
  (a `workflow_dispatch` trigger of `audit-file.yml` on the hub), with no
  direct-filing fallback even when the relay is unreachable — a refused
  trigger is reported (branch name +
  relay-unreachable line), never worked around.
- **Never merge and never push to the default branch (`main`).** Every fix is a
  PR; a human merges it. No auto-merge, no direct commits to `main`.
- **Only checked findings.** `- [ ]` findings are never branched, implemented,
  or PR'd. The checkbox is the owner's approval; absence of a check is a refusal.
- **One PR per finding**, grouped only when the fix is literally identical.
- **One subagent per finding, sequential.** The orchestrator never implements
  in its own context and never runs two implementers at once (shared working
  tree). These boundaries bind the subagents too — brief them in.
- **Scoped diffs.** A PR's changes stay within the finding it resolves; if the
  fix can't be done in scope, stop that finding and report it, don't grow the
  diff.
- **Every PR links its audit finding** (non-auto-closing reference) and **quotes
  it verbatim**; risk-critical PRs carry the caution flag.
- **Do not modify the audit issue** — do not check/uncheck boxes, edit, or close
  it. Reading is the only interaction with it.
- **Never touch the owner's uncommitted work.** Stop on a dirty tree.
