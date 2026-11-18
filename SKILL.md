---
name: evidence-review
description: Deep, evidence-based code review of anything - a whole project with no diff, a path, uncommitted changes, a branch, or a GitHub PR / GitLab MR. Reproduces findings instead of guessing, judges layer choice and reuse like a strict maintainer, and writes a structured report with IDs, severity, status and verification.
license: MIT
compatibility: Agent Skills (SKILL.md) format. Built and tested on Claude Code; meant to work in any coding agent that loads SKILL.md skills and can run shell commands (Codex CLI, Cursor, GitHub Copilot and others). Needs git; gh or glab for PRs/MRs.
disable-model-invocation: true
argument-hint: "[path | diff | branch [base] | pr <number|url>] [--update <report.md>]"
---

# Evidence review

You are doing the review a senior engineer does when they are accountable for what ships: they
read what is in scope, run the tooling, reproduce what they suspect, and say plainly what they
could not check. Two standards define the bar:

- **The project audit.** The code in scope read in full, build and tests run, each serious finding
  reproduced with a throwaway script, a clear severity scale, statuses that evolve as fixes land,
  and an honest "what is in good shape" section.
- **The strict maintainer.** First asks whether the change belongs in this layer at all, points at
  the existing helper or extension point by file and line, traces every reader of shared state
  before calling something safe, separates blocking from non-blocking, and pins down contracts
  before anything depends on them.

Depth comes from evidence, not from volume. A short report where every finding is real beats a
long one padded to look thorough.

## 1. Resolve the scope

Arguments given: "$ARGUMENTS" (empty quotes mean no arguments). If that still reads as a literal
placeholder, your agent does not substitute arguments: take them from the user's request instead.

| Argument | Scope | Mode | Output |
|---|---|---|---|
| none | The whole repository, working tree as is | project | Report file |
| a path | That path in full, plus the definitions it calls one level deep | project | Report file |
| `diff` | Staged + unstaged changes: `git diff HEAD` | change | In chat |
| `branch [base]` | `git diff <base>...HEAD` (three dots: from the merge-base) | change | In chat |
| `pr <n\|url>` | The PR/MR head against its target branch | change | In chat |
| `--update <file>` | Re-verify an existing report; project mode only, with or without a path | update | Edits that file |

- **Base branch** for `branch`: `git symbolic-ref --short refs/remotes/origin/HEAD`; if that
  fails, `main`, then `master`; if neither exists, it is an open question (section 2).
- **Dirty tree in project mode:** review the working tree as is, and record
  `HEAD <sha> + uncommitted changes` as the revision. Modified files get `path:line`, not pinned
  links.
- **No arguments and a dirty tree:** that is an open question (project or diff?).
- **Monorepo:** for a path, run the commands of the package that owns it (nearest
  `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml` / `*.csproj`) and read rules files
  from the repo root down to that package.

**Change mode still reviews the surrounding code.** The unit of review is the changed hunks *plus*
the full functions they sit in, every caller of a changed symbol, every reader and writer of any
state the change touches, the tests that cover it, and the docs that describe it. A diff read in
isolation is how the expensive bugs get approved.

### Getting a PR or MR

Never check out over the user's working tree. Always use a separate worktree in a scratch
location:

- GitHub: `git fetch origin pull/<n>/head:pr-<n>`; GitLab: `git fetch origin
  merge-requests/<n>/head:mr-<n>`. Then `git worktree add <scratch>/pr-<n> pr-<n>`.
- Metadata: `gh pr view <n> --json title,body,comments,reviews,baseRefName,headRefName` or
  `glab mr view <n> --comments`. If neither CLI is available or signed in, review the diff only
  and state in the header that the discussion history was not read.
- To run the base for comparison (section 4), add a second worktree at the merge-base. Never swap
  the head worktree back and forth.
- Remove the worktrees at the end (`git worktree remove`).

## 2. Context, and one round of questions

Collect, and keep for the report header:

- Revision (full SHA), branch, remote URL. With no remote, skip the rest of this bullet and use
  `path:line`. Otherwise run `git fetch --quiet` first, then
  `git branch -r --contains <sha>`; if the fetch fails, treat the SHA as unpushed. Links are
  pinned to the SHA only when it is pushed and the host is GitHub or GitLab (convert SSH remotes
  to https):
  - GitHub `https://github.com/<o>/<r>/blob/<sha>/<path>#L<a>-L<b>`
  - GitLab `https://<host>/<group>/<repo>/-/blob/<sha>/<path>#L<a>-<b>`
  - anything else: `path:line`.
- **Project rules.** Read every instruction file that applies to the code in scope, and follow
  imports inside them (a line such as `@AGENTS.md` pulls in that file):
  1. In the package and the repository: `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, lint,
     format and type configs, PR templates.
  2. Rules files in parent directories above the repository root.
  3. The user-level instruction file of the agent you run in, for example `~/.claude/CLAUDE.md`
     or `~/.codex/AGENTS.md`. Skip it if it does not exist.

  When two files disagree, the one closest to the code wins: package over repository,
  repository over parent directories, all of them over the user-level file. The user-level file
  is the reviewer's own taste, not a rule of someone else's project: when the code belongs to
  another project (an upstream PR, a repository the user does not maintain), file nothing that
  rests only on it. Findings are judged against these rules first, general taste second. List
  the files read in the report's Rules row.
- What the system is for: README, docs, deploy manifests, CI config, entry points.
- For a PR: its description, linked issues, existing review comments and their resolution, and
  the target branch. Read earlier rounds of review so you do not re-raise settled points, and
  check that requested changes were actually made.
- **Intent.** State in one sentence what problem the code or change is solving. If you cannot,
  that is an open question, because the answer may change how it should be wired.

**Report language**, first match wins: the user's explicit request; the project's rules file;
rules in a parent directory; the audience (an upstream open-source PR gets English); the
language of the project's existing docs. If none applies, it is an open question.

**Open questions.** Gather every open question from sections 1 and 2 (scope, base branch,
audience, a concern the requester wants covered, language, intent, a rule the rules files
leave undecided) and ask them **together, in
one message**. Do not ask anything you can find yourself. If no one can answer (a non-interactive
run, or the user said not to ask), use these defaults and list each one under "Assumptions" in the
report header: project mode for no arguments; maintainers as the audience; English; no named
concern; intent as stated in the PR description or README.

A concern the requester names gets its own section at the top of the report, whatever its
severity.

## 3. Budget and triage

Decide what "in scope" means before reading anything line by line.

- **Exclude from line-by-line reading:** lockfiles, `vendor/`, `node_modules/`, `dist/`/`build/`,
  generated code, minified files, binaries, snapshots, large fixtures, API collections. Grep or
  skim them for what matters (a pinned vulnerable version, a secret, an endpoint list).
- **Count what is left,** counting non-test source files only. Above roughly 40 files or 8,000
  lines, split the code into areas and fan out read-only subagents, one per area, to collect
  *candidate* findings with file:line and reasoning. This is required at that size, not optional:
  one context cannot read it all, and a silent skim is worse than an honest split. Every candidate
  is verified in the main thread (section 7). If your agent has no subagents, review one area per
  pass, keep a written candidate list between passes, and say in Scope how the work was split.
- **Tests:** read in full the tests that cover the riskiest paths and every test whose mocks a
  finding depends on; skim the rest for setup and coverage, and say so in Scope.
- **Order by risk:** entry points and authentication first, then everything that writes to a data
  store, then outbound calls, then the rest.
- **Record coverage honestly.** The report's Scope says what was read in full, what was skimmed
  and what was skipped. Never claim a full read you did not do.

## 4. Baseline health

### Safety before running anything

- **Code you did not write** (a PR from a fork or an unknown contributor, a repo you were just
  handed): installing, building and testing executes it, including install hooks and test setup.
  List the commands you intend to run and what they execute, and ask first. If no one can answer,
  do not execute: review statically and say so in the header.
- **Check where the tests point.** Read the test setup and env files. If the suite would reach
  anything beyond localhost (a shared database, a staging API, a real queue), do not run it;
  record why.
- **Keep the checkout clean.** Install with lockfile-respecting commands (`npm ci`,
  `pip install -r` with hashes where available, `uv sync --frozen`). Send build and coverage
  output outside the checkout, or confirm it is gitignored (test runners often write coverage into
  the repo by default: pass the output directory, e.g. `jest --coverageDirectory <scratch>`). At
  the end, `git status` must show no change you made.
- Never run anything against production or shared environments. Production checks become
  recommendations with the exact command to run (for example a query plan on the live database).

### What to run

Find the real commands in `package.json` scripts / `pyproject.toml` / `Makefile` / CI config,
then run and record each with its exact result: typecheck, lint, build, tests with coverage per
file, and the ecosystem's dependency audit (`npm audit`, `pip-audit`, `cargo audit`,
`govulncheck`, `bundle audit`, `dotnet list package --vulnerable`; `osv-scanner` as a fallback).
Audits need network and a lockfile; if either is missing, record it. Most audit tools exit
non-zero when they find advisories: that is a result, not a failed command.

- A project with no tests, build or linter: record "none configured" once. It is one finding,
  not one per file.
- Outdated dependencies are reported only when they carry a known vulnerability, are past end of
  life, or block a needed upgrade.
- A private dependency you cannot install: record it, review against the code that is available
  (an installed copy, a vendored copy), and say which library behavior went unverified.

In change mode, run the suite on the head, and where something fails also on the base (in its
own worktree), so every failure is labeled **new** or **pre-existing**.

## 5. Map the system

Before reading line by line, write down for yourself:

- Entry points: HTTP routes, protocol handlers (WebSocket, gRPC, telephony and the like), CLI
  commands, queue consumers, cron jobs, migrations, startup.
- Trust boundaries: who can reach each entry point, how the caller is authenticated, and where
  the tenant/user identity comes from.
- Data stores, their indexes, and which paths read and write each collection or table.
- Outbound calls: timeouts, TLS, retries, what an ambiguous failure means.
- Multi-step operations whose middle can fail.
- **The real shapes of the data.** Collect example values from the README, API docs, fixtures,
  seed data and API collections: path formats, date formats, ID formats, casing. Every parser and
  path helper is later run against all of them.

In change mode, map the blast radius instead: grep for every caller and every reader of what the
change touches, with file:line.

## 6. Deep read

Read [reviewer-lens.md](reviewer-lens.md) now. It is a checklist of questions, not a quota:
most sections have nothing to say about most files, and a section with nothing to say gets
nothing. Report only candidates with a concrete triggering scenario.

Read every file in scope fully, not excerpts. Include the parts of libraries the code relies on
where behavior at the seam matters (how a client library behaves after the peer disconnects, how
a framework handles an error after the response was sent). Many real bugs live there.

**Do not exempt the good code.** The parts that look well engineered get the same mechanical
checks as the rest: query plans for their hot queries, the real data shapes through their
parsers, readers of their shared state. Careful-looking code hides the bugs nobody looked for.

Keep a running list of candidates, each with: location, what goes wrong, the concrete scenario
(inputs and state that trigger it), and how you would prove it.

Also keep a list of things that are done well, with the evidence. "Confirmed working" claims need
the same rigor as findings: you checked the readers, ran the test, read the library code.

## 7. Verify

For every candidate that could be High or Critical, and any Medium where it is cheap:

- Reproduce it with a throwaway script in a scratch directory, never in the repo. Prefer real
  dependencies at small scale (an in-memory or containerized database, a stubbed upstream at the
  HTTP boundary) over mocks of the code under review.
- For performance claims, measure: query plans (`explain()`, `EXPLAIN ANALYZE`), timings before
  and after, at a data size where the problem shows.
- For "this is safe" claims about shared state, list every reader and writer.
- **Cap the effort:** about 15 minutes or 5 attempts per candidate. Past that, label it and move
  on with the reason. Stop reproducing once every Critical/High candidate has a label.

Label each finding's evidence:

- **yes**: reproduced by running code, including the project's own linter or formatter.
- **partly**: part of the claim reproduced; the section says which part.
- **artifact**: shown by a file, manifest, image layer or commit, or by a search that proves
  presence or absence (a grep, a path that does not exist), where running code is not possible
  or not needed (a missing deploy variable, a secret in history, a dangling reference).
- **no**: attempted and could not reproduce; say why it is still reported, or drop it.
- **not tried**: reasoning only, including a judgment made by reading the code.

Then be honest:

- A candidate with no plausible trigger in this deployment is dropped, not filed as Low.
- An inference stays labeled as an inference ("probably", "I could not check this from here").
- A test that passes only because a mock returns something the real code never returns is a
  finding about the test, not evidence about the code.

## 8. Classify

Severity, judged by impact and reachability in this deployment. When reachability depends on
something you cannot check (network ACLs, who can reach a port), file the severity as if it is
reachable, and state next to it what would lower it.

- **Critical**: exploitable now, with serious impact.
- **High**: data loss, cross-tenant or cross-user access, or a broken core feature.
- **Medium**: a real bug or risk that needs specific conditions.
- **Low**: hygiene, maintainability, docs, dependency drift, comment and style rules. A comment
  that contradicts the code on a security, locking or data-integrity path is at least Medium:
  the next person to change that code will trust it.

For a library, judge by the impact on a typical consumer. For a single-user tool, data loss is
still High, and a security finding needs a realistic attacker.

In change mode, also tag each finding:

- **Blocking**: a new Critical or High, or a new Medium on the main path of the change.
- **Non-blocking**: worth doing, fine as a follow-up.
- **Nit**: taste only, take it or leave it. A violation of a written project rule is
  non-blocking, not a nit, unless the rules file says it blocks.
- **New** or **pre-existing**: pre-existing issues are reported separately and never block the
  change under review.

Be proportionate. A rare edge case in a change that is strictly better than what it replaces is
non-blocking. A real limitation the author can defer is named, and the author is asked to note it
in the description.

Statuses (project and update modes): **Open**, **Postponed** (accepted for now, with the reason and
the trigger to revisit), **Fixed** (with a "How it was fixed" subsection), **Closed** (no further
work planned, with the reason).

## 9. Write it

**Project mode:** use [report-template.md](report-template.md).

- Location: where the project already keeps reviews; otherwise
  `docs/code-review-<YYYY-MM-DD>.md`, with `-2`, `-3` if that exists. Before writing, check
  whether that path is gitignored and whether `docs/` is published (GitHub Pages, a docs site).
  A report describing exploitable issues must not land somewhere public; if it would, write it
  outside the repo and say where.
- Do not commit it. Add the header the project's rules require for new files.

**Change mode:** answer in chat using [pr-review-template.md](pr-review-template.md). Offer to
save it to a file. **Never post a review, comment, approval or request-changes to GitHub/GitLab
unless the user explicitly says to in this conversation**, and show the exact text before
posting.

**Update mode:** re-check each finding against the current code.

- Change statuses. For each fixed finding, find the commit with `git log -L <a>,<b>:<file>` or
  `git log -S '<symbol>'` since the report's revision; if it cannot be identified, write
  "commit not identified".
- Append "How it was fixed": the commit, the approach, what was deliberately deferred and why,
  and how it was verified.
- Update the summary, the at-a-glance table and "Done so far". Add newly found issues with new
  IDs and an "Updated" line in the header.
- Never renumber existing IDs or rewrite history in the report.

## 10. Self-check before delivering

- Every finding has a location, a concrete failure scenario, a recommendation, and an evidence
  label.
- Every Critical/High has a label other than "not tried", or says clearly why.
- Recommendations point at existing code to reuse where it exists, by file and line.
- No credentials, tokens or real personal data (names, emails, phone numbers, addresses, real
  customer IDs) in the report. Say a secret exists and where, never its value. Synthetic values
  and examples taken from the project's own docs are fine.
- Scope states what was read in full, skimmed and skipped. Assumptions are listed.
- The summary ranks the most serious problems first and states build health in numbers.
- The "good shape" section is specific, not polite filler.
- The order of work is something a team could start on tomorrow.
- `git status` in the user's tree shows nothing you changed, and scratch worktrees are removed.
- You did not change the code under review. Fixes are a separate request.
