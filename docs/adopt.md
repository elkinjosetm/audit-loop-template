# audit-loop — adoption guide

Step-by-step instructions for forking this template and standing up your own
audit-loop **hub** — the central source of truth for the auditor, the
commands, all credentials, and the cross-repo automation (see `CONTEXT.md`
for the full glossary; every term below is used canonically). This guide is
a checklist with judgment notes, not a rewrite of `README.md` or
`docs/runbook.md` — where either already explains something in depth, this
guide links to it and summarizes in a line.

Placeholder used throughout: `<hub-slug>` = your fork's `owner/repo`, e.g.
`my-org/audit-loop`.

## 0. What you get

`arch-auditor` derives the standard it audits against from each target
repo's own documentation and established code patterns — never from this
hub. Each `/audit` run reports its survivors as one new **audit issue** in
the consumer, one checkbox per finding. Checking a finding's box is the
**triage gate** — the only signal that authorizes `/audit-fix` to touch it.
`/audit-fix` turns checked findings into scoped **fix PRs**, one per
finding, and a human merging that PR is the **review gate** — automation
never merges. The hub's **digest** sweeps the whole fleet after every
session that files something and sends at most one notification per
consumer with news, so triage stays a short daily habit instead of a
firehose.

**Trust model, in one paragraph:** a **routine** runs under the Claude
account that created it, with exactly the GitHub access that session has —
an MCP server scoped to the routine's own source repos — never a separate
service identity. The GitHub **App** you create in step 2 can act only on
the repos it is installed on; its stored private key can only mint
short-lived (~1 hour) installation tokens; and filing always goes through
the hub's `audit-file.yml` **relay**, which validates the target against
`consumers.txt` before writing anything. Nothing in the loop ever merges a
pull request — every fix ships as a PR a human reviews and merges by hand.

**Invariant vs configurable.** Never edit when adopting: the standard's
derivation order in `agents/arch-auditor.md`, the five finder lenses,
adversarial (REFUTE-mode) verification, cross-lens and historical dedupe,
the baseline mechanism, and the two human gates (triage, review). Yours to
set per instance: the App's credentials (step 2), the digest channel and
its secrets (step 3), the fleet listed in `consumers.txt` (step 4), and
each consumer's routine cadence and prompts (step 5).

## 1. Fork this template

Fork this repository into the org or personal account that will own the
hub. **The relay hard-enforces same-owner:** `audit-file.yml` rejects any
target whose owner doesn't match the hub's own owner (`target owner is not
the hub owner`), and the App token you'll mint in step 2 is owner-scoped —
so the fork's owner must be the owner of every repo you want audited. To
cover an org's repos, fork into that org, not into your personal account.

**Enable Actions on the fork.** GitHub disables Actions workflows on forked
repos by default — nothing in the loop runs, including the smoke-test
dispatch in step 3, until you turn them on. On the fork, open the
**Actions** tab and click **"I understand my workflows, go ahead and enable
them."**

Every runtime file resolves the hub's own slug at run time —
`/audit-install` reads it from an env var, from the marketplace source it
was installed from, or asks directly — so there is **no search-and-replace**
needed in `agents/`, `commands/`, or `.github/workflows/` to make the fork
work.

The two `.claude-plugin/*.json` files (`marketplace.json`, `plugin.json`) do
carry the original author's name and GitHub URL in their `owner`/`author`
fields. That is cosmetic marketplace-listing metadata, not machinery —
update it if you want the listing to show your own identity, skip it if you
don't care.

**Visibility.** Keep the hub fork public. If you make it private, set
**Settings → Actions → General → Access** so your other repos can call its
reusable workflows — a consumer's `audit-close.yml` stub (`uses:
<hub-slug>/.github/workflows/audit-close.yml@main`) needs to read the hub
(a fork of a public repo is itself public and can't be flipped private, so
this only applies if you used "Use this template" or imported the repo
instead of forking it).

Don't worry yet about `audit-fleet-update.yml` showing up in the Actions
tab: it only fires automatically on a push to `main` that touches one of
the three vendored source files (`agents/arch-auditor.md`,
`commands/audit.md`, `commands/audit-fix.md`), and it has a job-level guard
(`if: vars.AUDIT_APP_ID != ''`) that shows **neutral, not failed**, until
step 2 sets that variable. A plain fork with no such push shows no runs at
all in the tab — that's expected, not a signal to chase.

## 2. Create the GitHub App

All credentials in the system live on the hub; consumers hold none (see
`README.md`'s "Hub setup" for the full rationale). Create the App once:

1. On the hub fork: **Settings → Developer settings → GitHub Apps → New
   GitHub App**. Name it (e.g. `audit-loop-hub`), set the homepage to the
   hub fork's URL, and **deactivate the webhook** — nothing in the loop uses
   one.
2. Repository permissions: **Contents: read+write**, **Pull requests:
   read+write**, **Issues: read+write** (Metadata comes automatically).
   Restrict install to **only this account**.
3. Note the **App ID**, then **Generate a private key** — this downloads a
   `.pem` file.
4. **Install the App** on your account with **All repositories** — every
   current and future consumer is covered automatically, so adding a
   project later never means touching credentials again.
5. Wire the hub:

   ```bash
   gh variable set AUDIT_APP_ID -R <hub-slug>           # the App's numeric ID (not secret)
   gh secret set AUDIT_APP_PRIVATE_KEY -R <hub-slug> < ~/Downloads/your-app.private-key.pem
   ```

   (GitHub names the downloaded key after the App's slug plus the download
   date, not literally `audit-loop-hub.*` — substitute whatever file landed
   in your downloads.) The `-R <hub-slug>` flag targets the hub explicitly
   regardless of your shell's current directory.

6. **Delete the downloaded `.pem` from your machine** once it's stored in
   the secret. Nothing long-lived with repo access should exist outside
   GitHub's own secret store — the token every workflow actually uses is
   minted per run and expires in about an hour.

Once `vars.AUDIT_APP_ID` exists, `audit-fleet-update.yml` starts running for
real the next time one of the three vendored files changes on `main` — but
don't treat that as your confirmation signal; the digest smoke test in step
3 is the stated way to confirm credentials work.

## 3. Pick the digest channel

The digest (`.github/workflows/audit-digest.yml`) is channel-selectable via
`vars.DIGEST_CHANNEL`. Pick exactly one:

- **Pushover (default — unset, or explicitly `pushover`):**

  ```bash
  gh secret set AUDIT_PUSHOVER_TOKEN -R <hub-slug>     # dedicated Pushover application token
  gh secret set AUDIT_PUSHOVER_USER -R <hub-slug>      # Pushover user key
  ```

- **Slack:**

  ```bash
  gh variable set DIGEST_CHANNEL -R <hub-slug> --body slack
  gh secret set AUDIT_SLACK_WEBHOOK_URL -R <hub-slug>  # a Slack incoming webhook URL
  ```

Any other value of `DIGEST_CHANNEL` fails the run outright instead of
silently sending nothing, so a typo is caught immediately rather than
discovered as "no notifications ever arrived."

Smoke-test the wiring — **this is the confirmation that your App
credentials and digest channel actually work**, not `audit-fleet-update.yml`
showing up green:

```bash
gh workflow run audit-digest.yml -R <hub-slug> -f force_push=true
```

`force_push` sends a notification even with no news, so this succeeds
against your still-empty fleet and reports no news (`Sin novedades en
ningún proyecto`, in the default copy — see below) — that is the pass
condition.

**Localization note:** every string the digest sends (`Auditoría`,
`hallazgos nuevos`, `sin triage`, `fix PR nuevo`, `Sin novedades…`) is
Spanish and hardcoded directly in the compute and send steps of
`.github/workflows/audit-digest.yml` — there is no separate localization
file. Edit those strings in your fork if you want another language (see
`docs/runbook.md`'s "Notifications" section).

## 4. Onboard the first consumer

Pick a repo you want the loop to audit. **It must share an owner with the
hub fork** — the relay rejects any target under a different owner, and the
App token is minted owner-scoped, so a repo under another account or org can
never be a consumer of this hub (see step 1).

1. **Install the plugin from your own fork's slug** (once per machine):

   ```bash
   claude plugin marketplace add <hub-slug>
   claude plugin install audit-loop@audit-loop
   ```

2. In a Claude Code session **on the consumer repo**, run `/audit-install`.
   It:
   - vendors `agents/arch-auditor.md`, `commands/audit.md`,
     `commands/audit-fix.md` into the consumer's `.claude/`, each
     content-hash stamped with `hub=<hub-slug>`;
   - writes the `audit-close.yml` workflow stub with its `uses:` line
     substituted to `<hub-slug>/.github/workflows/audit-close.yml@main`;
   - seeds `docs/audit-baseline.md` (header only, no entries) from
     `templates/audit-baseline-header.md`;
   - creates the `audit` label;
   - registers the consumer as a line in the hub's `consumers.txt` — this
     is what turns on digest coverage and fleet updates (see `CONTEXT.md`'s
     Consumer and Fleet entries) — or prints the line to add by hand if it
     can't push to the hub; verify the line actually landed, since
     registration is the membership;
   - gates any per-PR preview-deploy job in the consumer's own CI away from
     `audit-fix/**` branches (a cost guard only — every check job stays);
   - prints the remaining steps as guidance, because it has neither the
     write access nor the standing to do them on your behalf.
3. **Commit and merge** the vendored files, the workflow stub, and any CI
   gate edits on the consumer's default branch. Routines and Actions can't
   see any of it until it lands on `main`.

`/audit-install` derives nothing project-specific: it never writes the
consumer's own standards docs (`CLAUDE.md`, `CONTEXT.md`, a design spec,
`docs/adr/`) — authoring those stays the consumer's own job. If the
consumer has none of them yet, that is not an install-time failure; the
first `/audit` run reports it as finding #1 instead (see `README.md`'s "How
the auditor derives 'the standard'").

## 5. Create the two routines per consumer

Two scheduled routines, created with `/schedule` (or the routines API — see
`commands/audit-install.md` step 7c and `docs/runbook.md`), both bound to
the consumer's default branch. A routine belongs to whatever Claude account
creates it and runs under that account's GitHub access (see the trust model
in step 0) — create each one from the account that should own the
automation, personal or your company's; that's the only difference it
makes:

| Routine | Cadence (reference) | Prompt |
|---|---|---|
| `<repo> daily audit` | daily, early morning | `Run /audit and let it record the findings.` |
| `<repo> daily audit-fix` (optional, once you trust the loop) | daily, late afternoon | `Run /audit-fix and open PRs for the approved findings.` |

Required for both: **the hub fork (`<hub-slug>`) as a second source repo.**
Cloud routine sessions reach GitHub only through a GitHub MCP server scoped
to the routine's own sources — without the hub in scope, a session cannot
trigger the `audit-file.yml` relay, and the run ends with a
relay-unreachable report instead of filing anything (`docs/runbook.md`,
"The daily routine"). Model `sonnet-5`, `allowed_tools` including `Task`
(both commands fan out subagents).

**Gotcha:** routine creation auto-attaches any MCP connectors already on
your account, even though the loop needs none. Strip them right after
creating each routine with a `{clear_mcp_connections: true}` update.
Routines can only be **deleted** from https://claude.ai/code/routines —
there is no delete endpoint via the API.

Daily is the reference cadence, not a requirement — it is safe at that
frequency because every piece absorbs it: small per-run issues, zero
re-filed noise from historical dedupe, `/audit-fix` no-ops without checked
boxes, and the digest pushes at most once per consumer with news. Retune
freely; frequency lives entirely in the routine, never in the plugin. If
GitHub Actions minutes are a budget concern, see the runbook's alternating
3-audit/3-fix-day cadence example in "Replication — installing the loop in
another project."

## 6. Operate: the triage loop, from the owner's seat

Once a consumer's routines are live, day to day is entirely at the two
human-owned gates — nothing else in the loop asks you for anything:

1. An **audit issue** arrives (one per run with survivors, label `audit`,
   one checkbox per finding). The digest notifies you once per consumer
   with news, every issue and PR number linked.
2. **Triage gate:** open the issue and check the box on every finding you
   want fixed. Leave the rest unchecked, or **baseline** a finding
   permanently with an entry in the consumer's `docs/audit-baseline.md` — a
   baselined finding is suppressed from all future reports.
3. The next `/audit-fix` run opens one **fix PR** per checked finding,
   quoting the finding verbatim, never merging and never touching `main`
   directly.
4. **Review gate:** review each PR like any other and merge the ones you
   want. Once every finding in an issue is checked and has a merged fix PR
   quoting it, `audit-close.yml` closes the issue on its own.
5. If a fix PR conflicts or goes stale, **close it unmerged** to
   *regenerate* it — the box stays checked, and the next `/audit-fix` run
   re-implements the finding from current `main`. To **abandon** a finding
   instead, uncheck its box or baseline it. Resolving a merge conflict by
   hand is never the default move here — see `docs/runbook.md`'s
   "Conflicted or stale fix PRs."

For the full vocabulary behind every term above — audit issue, tracked
finding, approved finding, option finding, resolved finding, regenerate,
abandon, baseline — see `CONTEXT.md`. For everything about running the loop
beyond this checklist (the full issue lifecycle, fleet updates, CI-minute
conventions, and the relay's troubleshooting steps), see
`docs/runbook.md`.
