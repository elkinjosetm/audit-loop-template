# Audit baseline — findings the owner won't fix

This file is a suppression list read by the `/audit` command. Before
printing a report or filing a GitHub issue, `/audit` compares every
surviving finding against the entries below and drops any match. Use this
file only for findings the owner has explicitly reviewed and decided
**not** to fix — never to pre-emptively hide findings that haven't been
reported and rejected at least once.

The normal lifecycle for an entry: `/audit` files an issue → owner reviews a
checkbox finding and decides it's not worth fixing → owner adds an entry here
(and may close that finding's issue, or leave it open with the box unchecked —
either way the finding will not be re-filed once it's baselined) → future
`/audit` runs suppress it.

## Entry format

One entry per suppressed finding — not per file; a file may carry more than
one entry if it has more than one baselined violation. Each entry is a level-3
heading naming the file, followed by four required fields:

```markdown
### <path>

- **standard:** <the same "standard" text the finding cited — a doc heading,
  or `ADR-NNNN §clause`, or `convention @ path:line, path:line`>
- **what:** <one-line paraphrase of the violation, close enough to the
  finding's original "what" that a human (or the auditor) can tell it's the
  same issue>
- **reason:** <why the owner is not fixing this — accepted risk, deliberate
  exception, permanently low priority, etc.>
- **added:** <YYYY-MM-DD>
```

### Matching rule

`path` is the file only — a line number, if included, is ignored, because
line numbers drift as the file changes around them. A candidate finding is
suppressed only when **all three** hold:

1. same `path` (same file as the entry),
2. same `standard` (same cited rule/ADR/convention as the entry), and
3. the same underlying violation — the finding's `what` and the entry's
   `what` describe the same root cause (semantic match, judged by the
   auditor reading both — not a byte-for-byte string match).

A finding that matches on only one or two of the three is a **different**
finding and is not suppressed. When in doubt, do not suppress — an
unsuppressed finding just gets re-reported for the owner to triage again; a
wrongly-suppressed finding disappears silently.

## Baseline entries

_(none yet — entries are added here after the owner reviews a filed audit
issue and rejects a finding as "won't fix")_
