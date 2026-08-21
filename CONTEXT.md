# Audit Loop

A centralized architecture-audit system: one hub repo ships a portable auditor
to many consumer repos, files each run's surviving findings as one audit issue
in the consumer, turns owner-approved findings into scoped fix PRs, and
notifies the owner through one bounded digest channel.

## Language

### Topology

**Hub**:
The audit-loop repo itself — the single source of truth for the auditor, the
commands, all credentials, and every cross-repo automation.
_Avoid_: central repo, plugin repo

**Consumer**:
A repo registered in the hub's fleet list. Registration is the membership:
a repo carrying vendored copies but absent from the list is installed, not a
consumer — it gets no digest coverage and no fleet updates.
_Avoid_: client, subscriber

**Fleet**:
The set of all consumers, as listed in the hub.

**Vendored copy**:
A managed, hash-stamped copy of a hub runtime file living inside a consumer.
_Avoid_: installed file, synced file

**Drift**:
A vendored copy whose content no longer matches its hash marker because it was
edited locally. Drifted files are never overwritten by automation.

**Fleet update**:
The hub-driven re-vendoring that opens one update PR per consumer that is
behind.

**Routine**:
A scheduled unattended session with a one-line prompt; all audit intelligence
lives in the consumer's vendored copies, never in the routine.
_Avoid_: cron job, scheduled task

**Relay**:
The only filing path: a session triggers the hub's audit-file workflow (a
workflow_dispatch run), and that workflow — after checking the target against
the fleet list — creates the issue or fix PR in the target consumer as the
App. Sessions never create issues or PRs themselves, even when the relay
fails.
_Avoid_: direct filing, proxy filing

**Run marker**:
The UUID a session sends among the relay inputs and the hub stamps into the
created item's body (as an HTML comment), letting the session poll for the
item and report its URL.
_Avoid_: correlation ID, nonce

### Auditing

**The standard**:
The set of rules a repo is judged against, derived at run time from that
repo's own documentation and established code patterns — never from the hub.

**Lens**:
One of the five finder perspectives `/audit` fans out (standards conformance,
extractable primitives, dead code, ADR conformance, module depth).

**Finding**:
A specific violation of the standard, citing the exact rule it breaks. A
claim that cites no standard is not a finding.
_Avoid_: issue (reserved for the GitHub issue), violation, problem

**Candidate finding**:
A lens output that has not yet survived adversarial verification. Candidates
are never reported or filed.

**Confirmed finding**:
A candidate that survived REFUTE-mode verification. Only confirmed findings
reach suppression checks and filing.

**Regression**:
A finding that reappears after the issue that tracked it was closed. History
never suppresses it — it is re-filed as new.

### Triage

**Audit issue**:
The one issue a single audit run files, listing that run's surviving findings
(label `audit`). Audit issues are create-only: no later run ever edits one's
body, and a consumer may have several open at once.
_Avoid_: rolling issue, audit ticket, findings issue

**Tracked finding**:
A finding recorded in any open audit issue. Tracked findings are never
re-filed by later runs.

**Baseline**:
A consumer's permanent won't-fix list. A baselined finding is suppressed from
all future reports and issues.
_Avoid_: accepted findings, reject list

**Approved finding**:
A tracked finding whose checkbox the owner has checked — the only signal that
authorizes implementing it.
_Avoid_: accepted finding, selected finding

**Option finding**:
A finding whose fix is genuinely the owner's choice. Instead of a single Fix
bullet it carries a `Fix (choose one)` block of option checkboxes that work
like radio buttons: the owner checks exactly one option in addition to the
finding's own box. With zero or several options checked the finding stays
undecided — approved but not implementable. A Fix bullet with an inline "or"
is never valid; it is either restructured into this block or demoted to the
alternatives details block before filing.
_Avoid_: ambiguous finding, radio finding

**Resolved finding**:
A finding that no longer needs attention: its fix PR merged, or it was
baselined. Unchecked-and-unbaselined findings are undecided, not resolved.

**Triage gate**:
The first human-owned gate: checking a finding's box is the approval to touch
it. Nothing unchecked is ever implemented.

**Review gate**:
The second human-owned gate: every fix ships as a PR that a human reviews and
merges. Automation never merges.

### Fixing

**Fix PR**:
The one scoped pull request implementing a single approved finding, quoting it
verbatim and never auto-closing its audit issue. Its branch is pushed by the
session; the PR itself is filed through the relay.

**Regenerate**:
The triage action of closing a fix PR unmerged (deleting its branch) so the
next fix run re-implements the still-approved finding from current `main`.
_Avoid_: rebase, refresh

**Abandon**:
The triage action of withdrawing a finding: uncheck its box, or baseline it.

### Notification

**Digest**:
The hub's single notification channel — an on-demand sweep each session
triggers as its final step, after its filings landed. Every sweep covers the
whole fleet but notifies scoped per project: at most one push per consumer
with news, titled with that consumer's name, and only when there is news.
Every referenced issue or PR is linked.
_Avoid_: daily digest, digest cron

**News**:
What the digest reports: findings newly filed and fix PRs newly opened since
the last snapshot. The owner's own triage activity is never news.
