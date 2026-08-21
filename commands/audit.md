---
description: Audit the codebase against its own documented standards — five lenses, cross-lens dedupe, adversarial verification, baseline + historical dedupe, files survivors as one new audit issue per run, via the hub relay.
argument-hint: "[optional scope: paths/globs to limit the audit, e.g. apps/web]"
---

# /audit

Run a full conformance & architecture audit, print a prioritized report **in
this conversation**, and file the surviving findings as **one new audit
issue** (label `audit`) through the hub relay — a `workflow_dispatch` trigger
of `audit-file.yml` on the hub, filed by the App — unless
nothing new survives, in which case file nothing. A run with survivors always
files exactly one new issue; a run with nothing new touches nothing. No
session ever runs `gh issue create`, and no run ever edits an existing audit
issue's body — filing is create-only, which is what makes the async relay
race-free. Do not edit code. The analysis brain is the `arch-auditor`
subagent, which stays strictly read-only (including no GitHub access); the
orchestrator does all GitHub interaction itself: reads and the closability
comment with its own GitHub access, filing via the relay trigger — never
delegated to a subagent.

Scope: `$ARGUMENTS` if given, otherwise the whole application source (all app +
package workspaces), full audit — not just the diff.

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
find (5 lenses) → cross-lens dedupe → adversarial verification
  → baseline suppression (docs/audit-baseline.md)
  → historical dedupe (open `audit`-labeled issues)
  → in-conversation report of survivors
  → survivors > 0: trigger the hub relay to file one new audit issue, poll for its URL
  → survivors = 0: touch nothing
```

## The five lenses

1. **Standards conformance** — i18n/locale, design tokens & type scale, shared
   UI-primitive reuse, money/units the repo declares, query/error/toast
   patterns, identifier-vs-user-copy language policy.
2. **Extractable primitives / DRY** — logic or markup duplicated 2+ times that a
   shared util/component should own.
3. **Dead code** — unused exports, unreachable branches, debris from removed
   features, commented-out code.
4. **ADR conformance** — code that contradicts a durable decision (money path,
   timezone, fail-loud, persistence, auth/exposure, etc.).
5. **Module depth** — shallow modules, deletion-test pass-throughs, testability-
   only extractions with no locality (detection only, no redesign).

The lens *contents* are not defined here — `arch-auditor` derives every concrete
rule from this repo's own docs at run time. Keep this command portable: name the
lenses, do not hardcode any project-specific standard.

## Model tiers (assign deliberately)

| Phase | Subagent | Tier | Why |
|---|---|---|---|
| Finders — mechanical lenses (dead code, standards scan) | `arch-auditor` MODE=FIND | **haiku** | grep-shaped scouting, cheap |
| Finders — reasoning lenses (DRY, ADR conformance, module depth) | `arch-auditor` MODE=FIND | **sonnet** | needs judgment, not deep risk analysis |
| Cross-lens dedupe | orchestrator (you) | — | mechanical merge, no subagent |
| **Verification / refuter** | `arch-auditor` MODE=REFUTE | **opus** (never below sonnet) | gates what gets reported; a cheap model here rubber-stamps |
| Baseline suppression, historical dedupe, relay trigger + poll-back | orchestrator (you) | — | text/JSON matching + GitHub calls, no subagent; never delegated |

Pass the tier as the `model` on each `Task` call. **Hard rule: the verification
pass is never haiku.** Scouting may be cheap; the pass that decides what
survives must not be.

## Procedure

### 0. Preflight

Confirm at least one standard doc exists (root agent guide, domain glossary,
design spec, or an ADR log). If none exist, run a single `arch-auditor` and let
it emit the missing-docs finding, print that as the whole report, and stop —
there is no standard to audit against.

### 1. Fan out one finder per lens (parallel)

Spawn **five** `arch-auditor` subagents in a single batch (one message, five
`Task` calls) so they run in parallel. Each gets:

- `MODE: FIND`
- `LENS: <one of the five>` (exactly one per finder — do not combine lenses)
- `SCOPE: $ARGUMENTS` (or the default full source)
- the tier from the table above.

Collect every FINDING block from all five.

### 2. Cross-lens dedupe (you, the orchestrator)

Two findings at the **same `path:line`** raised by different lenses are **one**
finding — merge them: keep the higher severity, union the cited standards, keep
the clearest fix. Findings at different lines stay separate even if related.
Produce one deduped candidate list.

### 3. Adversarial verification (parallel, strong model)

Hand the deduped candidates to `arch-auditor` MODE=REFUTE at the verification
tier. Batch by lens or in small groups (keep each refuter's list short enough to
verify carefully); run the batches in parallel. The refuter tries to **kill**
each candidate.

Keep only `verdict: CONFIRM` findings. **Drop** every candidate that is KILLed,
that cannot cite a real documented standard or a proven 2+-site convention, or
that is marked `[unanchored]`. Apply any severity corrections the refuter made.
Do not re-add a dropped finding.

### 4. Baseline suppression (you, the orchestrator)

Read `docs/audit-baseline.md` (its own header documents the entry format and
the matching rule — follow it exactly, do not reinvent matching logic here).
If the file doesn't exist yet, skip this step (nothing to suppress).

For every CONFIRM-ed finding from step 3, check it against every baseline
entry. Suppress (drop) a finding only when **all three** of the baseline's
matching-rule fields agree: same `path` (file, line ignored), same `standard`,
same underlying violation described by `what` (semantic match — read both and
judge, don't string-match). Partial matches (e.g. same file, different
standard) are NOT suppressed. Record the count suppressed for the tally.

### 5. Historical dedupe against open `audit` issues (you, the orchestrator)

Fetch the **open** audit issues only — closed ones are irrelevant here and
skipping them keeps this step O(open issues) forever, no matter how much
audit history accumulates:

```bash
gh issue list --label audit --state open --json number,url,body --limit 200
```

This fetch is what makes a finding **tracked**: per the glossary, a tracked
finding is one recorded in *any* open audit issue, not just the most recent
one. A repo may still have the single accumulating issue it used before this
change — to this step that issue is simply one more open audit issue, so no
migration is needed.

Parse every finding checkbox item (format defined in step 7: title +
`path:line` + What/Standard/Fix sub-bullets; **top-level** items only — the
nested checkboxes of a `Fix (choose one)` option block are triage controls,
not findings) into `(path, standard, what)`
tuples — `what` is the title plus the **What** bullet — this is the set of
findings already tracked and not yet resolved. (Closed issues never suppress
— and never need to be read: once an issue is closed its findings are
presumed fixed, so one that reappears in the code is a regression and must be
re-filed as new, not silently swallowed. Triage discipline keeps the open set
small, so this step never grows with the loop's age.)

Apply the **same matching rule as step 4** (same path, same standard, same
`what` semantically) between each surviving finding and this already-tracked
set. Drop any match. Record the count deduped for the tally.

What remains after steps 4 and 5 is the **new-findings list** — what gets
reported and, if non-empty, filed.

**Closability check (same fetch, while the open issues are parsed).** This
is the semantic fallback behind the deterministic auto-close Action (a
repo may ship one that closes an issue the moment its last approved
finding's PR merges); this pass additionally sees baselined findings and
manual fixes. Reconcile each open audit issue's findings against reality: a
finding is resolved when its fix PR is **merged** (find fix PRs with
`gh pr list --search "Audit finding from #<N>" --state all`) or when it is
baselined in `docs/audit-baseline.md`. If EVERY finding of an open audit
issue is resolved, comment once on that issue —
`All findings here are merged or baselined — this issue can be closed.` —
and note it in the step 6 tally. Skip the comment when the issue's latest
comment already says this (keeps repeated runs idempotent). Never close the
issue yourself and never touch its body or checkboxes — closing is the
owner's call; the comment is only the reminder at the moment it becomes true.

### 6. Prioritized report (in conversation)

Print the new-findings list **most-severe first** (critical → high → medium →
low). Lead every finding with a **plain-language title** — one sentence a
human can act on from the list alone (e.g. "ADR-0024 still says 2h idle
logout; code ships 15 min"), never the raw `what` clause. Detail goes in
labeled sub-bullets:

```markdown
**#<n> [<severity>]** <plain-language title>
`<path>:<line>`
- **What:** <what the code does — 1–3 full sentences>
- **Standard:** <doc heading, or ADR-NNNN §clause, or convention @ sites>
- **Fix:** <concrete change>
```

**Fix discipline:** the **Fix** is always exactly ONE actionable change — the
auditor's recommendation. Never write "do X, or alternatively Y" in the Fix.
When other valid fixes exist, list them after the Fix as clearly subordinate
notes (in the filed issue they go in the collapsed block defined in step 7b).
When no defensible recommendation exists — the choice genuinely belongs to
the owner, e.g. conform the code vs. amend the standard it violates — do NOT
smuggle the dilemma into the Fix as an "or": mark it as an **option finding**
and report each option as its own complete one-action fix (in the filed issue
it becomes the `Fix (choose one)` block defined in step 7b).

End with a one-line tally that accounts for the whole pipeline, e.g. `4 new
findings: 1 critical, 1 high, 2 medium (from 19 candidates; 7 dropped in
verification, 3 suppressed by baseline, 5 already open in issue #112)`. If
the closability check (step 5) flagged issues, add one line per flagged
issue: `Issue #112 is closable — all findings merged or baselined
(commented on the issue).`

If the new-findings list is empty, say so plainly and state which stage
consumed everything (verification / baseline / historical dedupe) — an empty
report is a valid, good result, and **no issue is filed** (step 7 is skipped
entirely).

### 7. File this run's findings as one new audit issue (via the hub relay) — only if the new-findings list is non-empty

Skip this step entirely when step 6 produced zero new findings — do not
trigger anything, and do not touch any existing issue. The hub relay
(`audit-file.yml` in the `audit-loop` hub) ensures the `audit` label exists on
every filing — no label-creation step is needed here.

**a. Preflight — make sure no relay run is still pending.** Relay latency
opens a window where findings from an earlier trigger are in flight but not
yet in an open issue; filing again in that window would double-file once both
land. Before building anything:

1. Check for pending relay runs (MCP equivalent: list this workflow's recent
   runs on the hub and count the not-completed ones):
   ```bash
   gh run list -R "$HUB" --workflow audit-file.yml --limit 10 \
     --json status --jq '[.[] | select(.status != "completed")] | length'
   ```
2. If the count is non-zero, wait for it to clear: poll every 15s, up to 2
   minutes.
3. If it cleared: **re-fetch the open audit issue list once more** (repeat the
   step 5 query) and re-apply baseline + historical dedupe (steps 4–5) before
   triggering — a relay run that just completed may have filed an issue the
   earlier fetch missed.
4. If still non-zero after 2 minutes: **abort filing**. Do not trigger.
   Report the pending run URL(s) and state plainly that this run's findings
   were not filed and will be re-found (and filed) by the next run.

**b. Build the issue body.** One checkbox item per finding, most-severe
first, exact format (this format is also what step 5 parses back out of prior
issues — do not deviate from it):

```markdown
- [ ] **[<severity>]** <plain-language title>
  `<path>:<line>`

  - **What:** <what the code does — 1–3 full sentences>

  - **Standard:** <doc heading, or ADR-NNNN §clause, or convention @ sites>

  - **Fix:** <concrete change>
```

Keep the **blank line before each sub-bullet** exactly as shown: it makes the
nested list render *loose* (GitHub adds vertical spacing between the
What/Standard/Fix paragraphs). Without the blank lines GitHub renders a tight
list — dense findings become a wall of text.

The checkbox line carries the severity and a **plain-language title** — one
sentence the owner can triage from without expanding anything; never paste
the raw `what` clause there. The backticked `path:line` and the
**What/Standard/Fix** sub-bullets are indented 2 spaces so they nest inside
the same checkbox list item. The checkbox itself is the owner's approval
gate: checking it later authorizes a future fix pass to act on that finding —
leave every box unchecked when filing.

**Alternatives block (only when the finding has more than one valid fix).**
The **Fix** bullet still carries exactly one action — the recommended one
(fix discipline, step 6). The remaining alternatives go in a collapsed
`<details>` block nested under the Fix (indented 4 spaces, blank lines around
the inner markdown so GitHub renders it), one bullet per alternative with the
condition under which the owner would prefer it, and a single small-print
hint at the end:

```markdown
  - **Fix:** <recommended change>

    <details><summary><em>Alternatives considered</em></summary>

    - <alternative change> — only if <condition when this is preferable>.

    <sub>To take an alternative, replace the **Fix** bullet above with it
    before checking the box.</sub>

    </details>
```

Checking the box always authorizes the **Fix** bullet as written; the owner
takes an alternative by replacing the Fix text with it before checking.

**Option block (only for an option finding — the choice belongs to the
owner).** When the fix discipline (step 6) found no defensible single
recommendation, the finding carries a **Fix (choose one)** bullet instead of
a plain **Fix**. Its nested checkboxes are the fix options and work like
radio buttons: while triaging, the owner checks the finding's own box (the
approval) **and exactly one option box** (the mandate):

```markdown
  - **Fix (choose one):** check exactly ONE option below, then check the
    finding's box. `/audit-fix` implements the checked option verbatim and
    skips this finding while zero or several options are checked.

    - [ ] <first complete, self-contained change>

    - [ ] <second complete, self-contained change>
```

Every option is a complete one-action fix on its own — never "see above",
never a delta on a sibling option, and never an option that itself contains
an "or". Option boxes are triage controls, not findings: only **top-level**
checkbox items are findings (step 5's parse of prior issues and
`/audit-fix`'s approval scan both rely on this).

**Filing gate — no multi-action Fix ever leaves this step.** After building
the body and before triggering the relay, re-scan every finding's **Fix**
bullet for coordinated alternatives ("or", "either … or", "alternatively",
"— or"). A Fix still carrying more than one action is a bug in this run:
restructure it — demote the non-recommended actions to the Alternatives
block, or convert the finding to an option block — and only then trigger.
`/audit-fix` skips a multi-action Fix as ambiguous, and in an unattended
routine that skip is silent: the finding stalls approved-but-unimplemented
in an open issue no run will ever resolve, which is exactly the failure this
gate exists to prevent.

Prefix the findings block with one line of context (run date, scope), e.g.:

```markdown
Findings from `/audit` (run 2026-07-01, scope: full source). Check a box to
approve fixing that finding in a future pass.

- [ ] **[critical]** Invoice totals bypass the integer-cents money path
  `apps/server/src/...:42`

  - **What:** ...

  - **Standard:** ADR-0019 §...

  - **Fix:** ...
```

**c. Trigger the hub relay with this run's findings.** Write the body built in
step b to a scratch file first — the body always travels as a file (or a
whole-file tool argument), never as an inline shell string. The relay is the
hub's `audit-file.yml` workflow, triggered via `workflow_dispatch`.

1. Generate a fresh run marker:
   ```bash
   MARKER=$(uuidgen | tr 'A-Z' 'a-z')
   ```
2. Derive the target repo from git (no GitHub call needed):
   ```bash
   TARGET=$(git remote get-url origin \
     | sed -E 's#^(git@github\.com:|https://github\.com/)##; s#\.git$##')
   ```
3. **Size guard.** If `<scratch-body-path>` is **> 60,000 bytes**: drop
   whole trailing findings from the body (most-severe-first, so these are the
   least severe), append the line `_+<N> findings omitted (relay size cap) —
   the next /audit run will re-find and file them._`, and re-check. Repeat
   until under the cap. Never truncate mid-finding — the omitted findings are
   safe to drop: they are in no open issue, so the next run's pipeline
   re-finds them.
4. Trigger the relay:
   ```bash
   gh workflow run audit-file.yml -R "$HUB" --ref main \
     -f kind=issue -f target_repo="$TARGET" -f marker="$MARKER" \
     -f title="Audit findings — $(date +%F)" -F body=@<scratch-body-path>
   ```
   `-F body=@<file>` reads the body from the file — never inline it. If this
   environment has **no `gh`** (cloud routines reach GitHub only through a
   GitHub MCP server), trigger the same workflow with that server's
   workflow-run tool instead: repo `$HUB`, workflow
   `audit-file.yml`, ref `main`, and the same inputs (`kind=issue`,
   `target_repo`, `marker`, `title`, `body` — body passed whole, never
   assembled by string-pasting findings into a shell command). This requires
   the hub repo to be in the session's GitHub scope; routines list the hub as
   a second source (see the hub's runbook). Never retry after an accepted
   trigger, and never send more than one trigger for this run's findings.
5. **If the trigger itself is refused** (non-zero `gh` exit, MCP tool error,
   or the hub is not in this session's GitHub scope): this is a relay
   failure, not a fallback trigger. Print the complete findings list (the
   full body built in step b) in-conversation, prefixed with: `Relay
   unreachable (<exact error>) — nothing was filed. These findings are in no
   open issue; the next /audit run will re-find them. If this persists, this
   environment cannot reach the hub — investigate hub access before the next
   run.` Do **not** fall back to `gh issue create` — direct filing is retired
   unconditionally, with no exception for a refused trigger.

**d. Poll for the filed issue and report it.** On an accepted trigger, poll
every 10s, up to 3 minutes (18 attempts):

```bash
gh issue list --label audit --state open --json url,body --limit 50 \
  --jq ".[] | select(.body | contains(\"audit-loop:run $MARKER\")) | .url"
```

(No `gh`? Same poll via the GitHub MCP server: list this repo's open issues
with the `audit` label and match the marker string in their bodies.)

The first non-empty result is the filed issue — report its URL as the final
line of output.

If polling times out with no match, **do not retry the trigger** (a second
one could double-file). Classify the outcome instead, with this environment's
GitHub tooling (`gh` shown; MCP equivalent: list this workflow's recent runs
on the hub):

```bash
gh run list -R "$HUB" --workflow audit-file.yml --limit 5 \
  --json status,conclusion,url,createdAt
```

Report, verbatim in structure:
- that the trigger was accepted and the run marker;
- **if the most recent matching run has `conclusion: failure`** — the relay
  **rejected** the payload (most likely: this repo is missing from
  `consumers.txt`, or a validation failure); include the run URL and say the
  issue will *not* appear;
- **if a run is `queued`/`in_progress`** — the relay is genuinely pending; the
  issue will likely still appear; include the run URL;
- **if no run is visible** — say the trigger may not have fired and
  point to `https://github.com/$HUB/actions/workflows/audit-file.yml`;
- in all cases: **"Do not re-run `/audit` (or `/audit-fix`) until the relay
  run's outcome is confirmed at the link above — a re-run before then can
  double-file."**
- Note that if the relay truly failed, nothing is lost: the findings are in no
  open issue, so the next `/audit` run re-files them.

**e. Trigger the digest sweep — only if this run filed an issue.** The hub's
digest (`audit-digest.yml`) is on-demand: it has no schedule, so the owner is
notified only when a session tells it to sweep. After the poll-back above
resolves (URL found, or the timeout was classified and reported), trigger it
once, fire-and-forget — no inputs, no polling:

```bash
gh workflow run audit-digest.yml -R "$HUB" --ref main
```

(No `gh`? Same trigger via the GitHub MCP server's workflow-run tool: repo
`$HUB`, workflow `audit-digest.yml`, ref `main`, no
inputs.) The sweep diffs the whole fleet against its snapshot and pushes one
scoped notification per repo with news, so triggering it is always safe —
never more than once per run, and never when this run filed nothing. If the
trigger is refused, just report it: nothing is lost — the next session's
sweep announces this run's issue.

Because the issue is App-authored, GitHub's native notifications also fire —
the relay itself sends nothing extra.

Never split one run's findings across multiple relay triggers. Never `gh issue
create`. Never `gh issue edit` any issue's body.

## Boundaries

- Read-only analysis: the `arch-auditor` subagent (both modes) must never
  Edit/Write project files, run mutating Bash, or touch GitHub. All mutation
  (the relay trigger, plus the single closability comment defined in step 5)
  happens only in the orchestrator's own steps — never delegated to a
  subagent.
- Never edit code, in any step.
- Issues are create-only: this command never creates issues or PRs with `gh`
  directly — not even as a fallback when the relay is unreachable (step 7c
  defines the failure report instead) — and never edits any existing issue's
  body, checkboxes, or order. The only mutations this command performs itself
  are the closability comment (step 5), the relay trigger (step 7c/7d), and
  the digest-sweep trigger (step 7e).
  Multiple open audit issues are normal; filing this run's findings never
  touches any other issue. Zero new findings → nothing triggered, reported
  plainly in conversation.
- Every reported (and every filed) finding carries `path:line` + a cited
  documented standard + a fix. No generic nits, no uncited opinions.
