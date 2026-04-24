# test-effectiveness-auditor

[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-orange)](https://claude.com/claude-code) [![license](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

Quantitatively answer the question **"how helpful are our automated tests at catching bugs?"** — not by proxy metrics like coverage percent or test count, but by replaying real bugs that already happened and checking whether the test suite, as it stood just before the fix, actually failed on the buggy commit.

## The Problem

Most teams measure test health by coverage % (e.g. `pytest --cov`). Coverage tells you which lines executed, not whether any assertion would have *failed* when the behavior was wrong. A line can be 100% covered by a test that would pass under the bug.

This skill inverts the question: take known bugs, rewind to the pre-fix commit, and observe whether the suite catches them. The honest baseline for "are tests worth it" is historical — bugs that made it to production despite the tests are direct evidence of gaps; CI failures that forced a code change before merge are direct evidence of catches. Everything else is speculation.

## What it does

Two methods, in priority order:

### Method 1 — Historical incident replay (primary)
1. Mine incident signals from `docs/findings/`, `docs/issues/`, `docs/diagnostics/`, `docs/audits/`, root-level `discovery_*.md` / `incident_*.md` / `postmortem_*.md`, and git log (`fix|bug|revert|hotfix|incident`).
2. For each incident, resolve `pre_fix_commit = fix_commit^`, create a git worktree at that SHA, run the project's canonical test command, capture exit code + failing test names.
3. Classify each incident as `caught` / `gap_testable` / `gap_hard` / `ambiguous` / `unrunnable` (see [`references/classification_taxonomy.md`](plugins/test-effectiveness-auditor/references/classification_taxonomy.md)).
4. Report per-incident table + catch rate + prioritised gap backlog.

### Method 2 — CI history analysis (secondary)
1. Pull the last 6 months of CI runs via `gh api` (GitHub Actions) or `gcloud builds list` (Cloud Build).
2. For each PR-blocking failure, classify as `real_catch` / `author_hygiene` / `flake` / `infra`.
3. Report the "effective catch rate" — what fraction of CI failures represented real-bug catches.

### Out of scope (deliberately)
- Mutation testing
- Test-layer ablation
- Fault injection
- Auto-writing tests

These are higher-cost methods whose ROI depends on first knowing the Method 1/2 baseline. The skill produces a human-reviewed gap backlog; the human decides which gaps to close.

## Install

Standalone:
```bash
claude plugin install wan-huiyan/test-effectiveness-auditor
```

Or via the bundle:
```bash
claude plugin install wan-huiyan/claude-ecosystem-hygiene
```

## Quick Start

```
You: audit our test suite — are we actually catching bugs?

Claude: [invokes test-effectiveness-auditor]
        Phase 1: discovers test command (pytest tests/), incident signals
                 (8 docs/findings + 195 fix-grep commits)
        Phase 2: proposes 8 candidate incidents for replay; you confirm
        Phase 3: replays each in a temp worktree, runs the suite
        Phase 4: classifies caught / gap_testable / gap_hard
        Phase 5: pulls CI history (or notes N/A if no CI tests)
        Phase 6: writes ~/Documents/<project>_test_effectiveness_audit.md
```

## Comparison

| | Without the skill | With the skill |
|--|------------------|----------------|
| Question answered | "Coverage is 73%" | "0 of 4 documented bugs were caught at pre-fix commit; here are the 3 cheapest tests that would close the cluster." |
| Effort | Run `pytest --cov` | One conversation |
| Action it produces | A number | A prioritised gap backlog |
| Honesty about hard-to-catch bugs | None | `gap_hard` classification + recommendation to use contract tests / monitors |

## Scope guarantees

- **Read-only relative to project source.** Creates temp worktrees, runs tests, reads git history. Never edits project code.
- **Never auto-writes tests.** Recommendations are produced as a human-reviewed backlog; the user decides which gaps to close.
- **Idempotent.** Running twice with the same project HEAD and same CI run window produces the same report. Keyed by (project head sha, incident id, CI run id).
- **Conservative on classification.** When the replay is inconclusive (environment error, missing dep, flake), the incident is classified `unrunnable` rather than forced into caught/gap.

## Limitations

- **Replay requires a runnable test environment.** Projects with heavy external-service dependencies or custom Docker test runners may report high `unrunnable` rates and need Method 2 only.
- **Classification uses heuristic textual matching for `caught`.** False negatives (tests that WOULD have caught but whose names don't textually match the incident) are possible. The skill is conservative: prefer `gap_testable` on ties and let the human re-classify on review.
- **CI history analysis requires `gh` CLI auth (GitHub) or `gcloud` auth + project id (Cloud Build).** If the user can't provide these, Method 2 is skipped — the report says so honestly.
- **Sample size matters.** A 5-incident audit is a directional signal, not statistically significant. The report flags this.

## Related

- [memory-hygiene](https://github.com/wan-huiyan/memory-hygiene) — clean up Claude Code's persistent memory + project docs taxonomy
- [claude-code-ab-harness](https://github.com/wan-huiyan/claude-code-ab-harness) — A/B-test your Claude Code stack
- [ecosystem-audit](https://github.com/wan-huiyan/claude-ecosystem-hygiene) — audit Claude Code ecosystem health (skills, sessions, ADRs)

## Version History

- v1.0.0 (2026-04-24): Initial release. Method 1 (historical replay) + Method 2 (CI history). Idempotent via per-incident cache.

## License

MIT
