# Spike verdict — can a cloud routine load this plugin from its marketplace?

Ran 2026-07-02, before building the plugin (see the runbook for the design
this verdict produced). Question: can an unattended cloud routine session
load `/audit` / `/audit-fix` from this marketplace, so consumer repos need
zero vendored files?

## Results

| Route | Result |
|---|---|
| Trigger API fields `enabled_plugins` / `extra_marketplaces` | **Dead (for now).** The routines API validates the shape (`[{name, source: {source: "github", repo}}]`) but does not persist the values — create and update both echo `[]` back. Server-side feature not enabled. |
| `.claude/settings.json` (`extraKnownMarketplaces` + `enabledPlugins`), private repo, cloud routine | Plugin **not loaded** |
| Same, repo flipped **public**, cloud routine | Plugin **not loaded** |
| Same settings, local headless `claude -p`, no prior install | Plugin **not loaded** |
| `claude plugin marketplace add` + `claude plugin install` (user scope), then local headless `claude -p` | **Loaded** ✔ |

## Root cause

External marketplace plugins declared in a project's `.claude/settings.json`
require a **one-time interactive install/trust step per user** ("doesn't
load until the team member installs it" — Claude Code docs, discover-plugins,
v2.1.195). There is no documented non-interactive pre-grant (no env var, no
CLI flag, no settings key). Cloud routine sessions are headless and start
fresh, so they can never perform that step — repo visibility is irrelevant.

## Decision

**Plan B (hybrid), per the handoff:** the plugin is the source of truth and
serves local/interactive use; `/audit-install` vendors the runtime files
(`arch-auditor` agent, `/audit`, `/audit-fix`) into each consumer repo with a
content-hash marker, and cloud routines resolve the commands from those
vendored copies. Drift is detectable by hash; updates are a re-run of
`/audit-install`.

## Possible future upgrade (untested)

The cloud **environment setup script** (editable only in the claude.ai UI,
runs before Claude Code launches) could run
`claude plugin marketplace add <hub-slug> && claude plugin
install audit-loop@audit-loop`, installing at user scope inside the sandbox
before the session starts — the same mechanism that made the local headless
test pass. If that works, vendoring becomes unnecessary for cloud and this
verdict should be revisited. Also worth re-testing the trigger API fields
periodically; the schema exists, so persistence may be enabled later.

## Method notes

- Probe: a throwaway `/audit-spike` command whose only job was to file a
  GitHub issue ("loaded OK" vs "NOT loaded") from the routine session, plus a
  fallback instruction in the routine prompt for the not-available case.
- A throwaway routine (delete via https://claude.ai/code/routines — the API
  cannot delete).
- Gotcha confirmed: routine creation auto-attached the Notion MCP connector;
  stripped with `{clear_mcp_connections: true}`.
