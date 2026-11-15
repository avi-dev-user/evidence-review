# evidence-review

A Claude Code skill for deep code review where every serious finding is backed by evidence: a
reproduction script, a query plan, a traced list of readers, or the exact file and commit that
shows it.

It works on anything: a whole project with no diff, a single path, uncommitted changes, a branch,
or a GitHub PR / GitLab MR. Project reviews produce a structured report with IDs, severity, status
and evidence labels that can be updated as fixes land. Change reviews produce a maintainer-style
review: what was confirmed working, blocking, non-blocking, nits, pre-existing issues and a
verdict.

## Install

```bash
git clone https://github.com/avi-dev-user/evidence-review ~/.claude/skills/evidence-review
```

Start a new Claude Code session so the skill is picked up.

## Use

```
/evidence-review                        # whole project, writes a report file
/evidence-review src/payments           # one path, plus what it calls one level deep
/evidence-review diff                   # uncommitted changes
/evidence-review branch main            # current branch against main
/evidence-review pr 123                 # a GitHub PR or GitLab MR
/evidence-review --update docs/code-review-2026-01-10.md   # re-verify an existing report
```

The skill is manual-only (`disable-model-invocation: true`): it is expensive and may execute the
code under review, so it never starts on its own.

## What it does

1. Resolves the scope, gathers project rules, intent and context, and asks all open questions in
   one message (or uses stated defaults when no one can answer).
2. Plans a budget: excludes generated and vendored files, fans out to subagents on large repos,
   reads in risk order, and records what was read in full, skimmed and skipped.
3. Runs the project's own typecheck, lint, build, tests with coverage and dependency audit.
4. Maps entry points, trust boundaries, data stores, outbound calls and real data shapes.
5. Reads the code with a reviewer lens: intent, layer and placement, reuse, correctness,
   concurrency, security and tenant isolation, validation, performance, contracts, operations,
   tests, tooling, drift, and conditional dimensions (migrations, privacy, frontend, LLM code).
6. Verifies candidates against real dependencies at small scale, with a cap on effort per
   finding, and labels each one: `yes`, `partly`, `artifact`, `no` or `not tried`.
7. Classifies by severity (and blocking / non-blocking / nit for changes) and writes the result.

## Safety

- Code you did not write is not installed, built or tested without asking first. In a
  non-interactive run it is reviewed statically.
- Test suites that would reach anything beyond localhost are not run.
- PRs are checked out in a separate worktree, never over your working tree.
- Nothing is posted to GitHub or GitLab unless you say so in the conversation.
- Reports never contain secret values or real personal data.
- A report describing exploitable issues is not written to a published docs folder.

## Cost and limits

Measured on real runs: a small PR took 5 to 7 minutes and about 125k to 140k tokens; a full audit
of a 30-file TypeScript service took about 12 minutes and 235k to 240k tokens.

Runs are not deterministic. The serious findings were stable across runs, while some Low and
Medium findings appeared in one run and not the next. For the widest coverage, run twice and merge.

## Files

| File | Role |
|---|---|
| `SKILL.md` | The procedure: scope, context, budget, safety, verification, classification, output |
| `reviewer-lens.md` | The questions asked of the code, with invented examples of the expected depth |
| `report-template.md` | Shape of a project report, including how fixed findings are recorded |
| `pr-review-template.md` | Shape of a change review |

## License

MIT. See [LICENSE](LICENSE).
