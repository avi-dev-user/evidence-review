# Reviewer lens

Questions to ask of the code in scope. They are language-agnostic. This is a checklist, not a
quota: most sections have nothing to say about most files. File a finding only when you can name
the concrete inputs and state that trigger it.

Sections 1 to 14 apply to almost any codebase. Section 15 lists dimensions that apply only when
the project has that kind of code; skip them otherwise.

The examples are invented. They show the depth and shape expected, not what to look for.

## 1. Intent and scope

- What problem does this solve, for whom? Is there a broader goal behind it that would change how
  it should be wired? If unclear, that is an open question, not a reason to guess.
- Does the change contain independent pieces? A self-contained fix bundled into a large feature
  should be split into its own PR so it can merge on its own.
- Does the PR description match the actual commits? Stale descriptions that list work already
  merged elsewhere are a finding. Deliberate behavior changes must be stated in the description.

> Shape: "The cache-invalidation fix has no coupling to the reporting feature it ships with.
> Split it into its own PR and it can merge this week."

## 2. Layer and placement

The first design question, before line-level review.

- Does this belong in core at all, or in the plugin / service / adapter / config layer that
  already owns the decision? Core inferring something a plugin can state explicitly is usually
  wrong.
- Is there an existing extension point (a callback, a filter hook, a config key) that already
  solves this without touching core?
- Is the error handled at the layer that has the information? Catching after a global side effect
  has already fired is catching too late.
- Is the code reading state through a proper interface, or scraping stdout, parsing log strings,
  or monkeypatching? Those work only by accident of the current process model.
- Is behavior inferred indirectly (introspecting source code, guessing from names) where an
  explicit declaration would be reliable?
- **Technically correct is not the same as belonging in core.** Even when a core change is right
  by the spec, ask whether the plugin could declare it itself. When the change crosses a
  core/plugin boundary and CONTRIBUTING is silent, read how maintainers closed earlier PRs of the
  same kind (at most about 10 recent closed PRs). If the project keeps core minimal, a correct
  core change is still a layer finding. If there is no such evidence, say so; do not invent a
  policy.

> Shape: "Plugins already pass a `filter` callback to `load_entries()` (loader.py:88). Drop the
> unwanted entries there; no change to the core loader is needed."

## 3. Reuse and duplication

- Before accepting new code, search the same file, then the module, for an existing helper that
  does this. Name it with file and line.
- Is this the second or third copy of the same logic? Ask for one shared helper that all paths
  call, and note which gaps unifying would close as a side benefit.
- Near-identical handlers that differ in one parameter should be one handler.
- Is the same condition tested several different ways in different places?

> Shape: "The role check already exists in `require_role()` (auth.py:41) and the fallback lookup
> in `resolve_account()` (accounts.py:210-224). Call them instead of adding a third copy."

## 4. Correctness and data integrity

- Multi-step operations (write locally, call an external system, update locally): what happens
  when each step fails, including after the external system already succeeded? Rolling back local
  state after a remote success orphans data.
- Ambiguous outcomes (timeout, dropped connection, peer gone): does the code treat "unknown" as
  "failed"? It should keep state and reconcile.
- Is the destructive action done before the check that should guard it? Load and validate first,
  act second.
- Is anything taken from the client that should come from the server's own record (a path, an
  owner, a price)?
- Are batch results matched to requests by position without checking the counts agree?
- Idempotency: is a retry safe? Is there a clear "already done" result?
- Parsers and helpers against the real data shapes collected in the system map: every path
  format, date format and time zone, ID format and casing. Falsy-but-valid values (`0`, empty
  string) collapsed to null.
- Does an existence check fail open (error treated as "not there")? It should fail closed.
- Do two features share one record type in a way that corrupts its meaning (restores written to
  a deletion log, drafts counted as sales)?

> Shape: "The provider captures the payment, then the order update throws; the catch deletes the
> order row, so the charge has no order. Reproduced with a stubbed provider: one capture, zero
> orders left."

## 5. Concurrency and shared state

- Global flags, events, caches, singletons: list every reader and writer with file:line before
  declaring a change safe.
- Is shared state reset between units of work (requests, jobs, items)? What does a long-running
  process see on its second run?
- If the last item in a loop fails, is the state left set on exit?
- Check-then-act sequences that are not atomic.
- Does work continue after the client disconnected? Do retries overlap with the abandoned request?
- Thread pools and futures: are all futures joined before the next phase starts, or only
  submitted? Does a broad `except` swallow interrupts?
- Does a deliberate loss of concurrency deserve a note as a known cost?

> Shape: "The only readers of `shutdown_requested` are worker.py:118 and queue.py:52, both inside
> the polling loop. Nothing in the export path reads it, so resetting it per job is safe."

## 6. Security and tenant isolation

- Is every entry point authenticated? Ports exposed through a load balancer count, internal
  protocols count. If a gap is accepted, record the compensating control and when to revisit.
- Does the tenant or user identity come from the authenticated principal, or from free-form
  headers and body fields the caller controls?
- Is every query scoped to the tenant, including lookups and deletes by ID?
- Authorization per action, not only per login: can a user act on an object they can see but do
  not own?
- New capabilities that let a client choose server-side resources (credentials, devices, paths):
  behind a gate that is off by default?
- Caches keyed only by a resource or provider name and shared across users: can one user read or
  poison another user's entry?
- Secrets: redacted everywhere they are serialized, logged or returned? In git history? In image
  layers via build args? Plain values in deploy manifests?
- Injection into anything interpreted: SQL/NoSQL operators, regex from user input, newlines or
  quotes into line-based protocols, shell, templates, JSON parsed without validation reaching the
  database.
- SSRF, path traversal, unsafe deserialization, weak or homemade crypto.
- Audit fields the caller can forge (client IP headers accepted from anyone).
- TLS verification disabled, tokens over plain HTTP, non-constant-time token comparison, body
  parsed before the token is checked.
- Error responses and alerts that leak internals or can be flooded by unauthenticated input.

> Shape: "`GET /invoices/:id` loads by id alone. Any signed-in user can read another company's
> invoice by changing the id. Reproduced with two seeded companies. Scope the query by the
> caller's `company_id` and return 404 on a mismatch."

## 7. Input validation and error handling

- Paging and size inputs bounded? Non-numeric values that become `NaN` and disable the limit?
- Parse failures return 400, not 500.
- Sort and filter fields whitelisted.
- Silent fallback: a partially supplied value silently replaced by a server default. Raise a
  clear invalid-parameter error instead.
- `x || default` masking a legitimate falsy value.
- Errors swallowed, logged-and-continued, or returned raw to the client.
- Dead branches: a "not found" path the service can never produce because it throws instead.
- Generic exceptions where a specific type would let callers act.

## 8. Performance and scale

- Can each hot query use an index? Partial indexes are used only when the query implies the
  partial filter. Prove it with a query plan for every query on a large or growing collection,
  including in code that looks careful. Reading the schema is not proof.
- Does the cost grow with an append-only collection or log?
- N+1 patterns: one round trip per item inside a request.
- Unbounded bulk endpoints, sequential remote calls per item, no timeouts on outbound calls.
- Do the tests run at a data size where the problem could ever show? If not, say so: passing tests
  prove nothing about this.

> Shape: "Login filters on `email`, but the only index on it is partial (`deleted_at: null`) and
> the query does not include that condition. Plan: full scan, 400,000 rows examined per login.
> With the condition added: index scan, 1 row examined."

## 9. Contracts and API shape

- A new field, return shape or event that other work will depend on: is its shape settled now?
  Changing it after a consumer exists is a breaking change. Duplicates, identity and scope (per
  item, per run) all decided?
- Removed or renamed fields, changed defaults, status codes, and events or schemas consumed by
  other services: who breaks?
- Explicit declarations over inference (a class attribute instead of introspection).
- Identifiers normalized the same way at every entry point (case, whitespace), on write and on
  query.
- Status codes consistent across endpoints (207 only for real partial success).
- Does the change alter documented behavior? Then the docs change in the same PR.

> Shape: "`failed_items` is a flat list of names, with duplicates and no item id. Decide the shape
> before the dashboard PR starts reading it; changing it afterwards breaks that consumer."

## 10. Operations and deployment

- Does the deploy manifest set every config value the code requires? Does the code fail loudly at
  startup when one is missing?
- Graceful shutdown: signal handling (a process running as PID 1 ignores signals it has no
  handler for), draining in-flight work, closing the database last.
- Health probes that test the real dependencies; a startup probe long enough for migrations.
- Container image: supported runtime version, no dev dependencies, non-root user, no secrets in
  layers, lockfile honored.
- CI: does anything run typecheck and tests before deploy?
- Migrations: idempotent, targeting the right collection or table, consistent failure behavior,
  never edited after they ran.
- Observability: can an operator tell from logs and metrics alone that this path broke?
  Correlation IDs, log volume, personal data logged at INFO, alerts that unauthenticated input can
  trigger.
- Resources: leaked handles and connections, unbounded in-memory caches, memory growth in
  long-running processes.

## 11. Tests

- Is the riskiest code covered, or only the easy code? Cite coverage per file.
- Do mocks return values the real implementation can never produce? Then the test proves fiction.
- Do tests model the real semantics rather than a mocked happy path?
- Does a test assert the buggy behavior as correct?
- Is there a regression test that fails on the old code? For a performance fix, one that fails on
  a bad query plan.
- New tests in the existing file for that area, not a parallel structure.
- Tests for dead code are noise.

## 12. Strictness and tooling

- Will the project's own strict checks pass (strict typing requiring annotations on new
  definitions, linters, pre-commit)? Run them; do not guess.
- Type escapes (`any`, casts, ignores) hiding a real mismatch.
- Unused mappings, variables, imports left after a refactor.

## 13. Hygiene and drift

- Dead code: branches for removed backends or features, scripts that run on import, functions
  nothing calls but still ship in the build.
- Documentation drift: defaults, commands, examples and field names in README, API collections and
  docs that no longer match the code.
- Supply chain: new dependencies (maintenance, license compatibility, install scripts), type
  packages mismatched with their runtime packages, deprecated stubs.
- Project rules: required headers, commit conventions, whatever the project's own instructions
  require. A rule phrased as an instruction for writing new code (headers on new files, a style
  for new components) is checked only against code the change under review adds. In project
  mode it produces no findings on existing code; at most, a note in Scope saying how much of the
  repository follows it.
- **Comments and style against the project's rules.** First write down each comment and style
  rule from the rules files (SKILL.md section 2), with the file it came from and which one wins
  where they conflict. Check each rule against the code in scope: grep where a rule is
  mechanical (a banned marker, a language, a required header), read where it is not (whether a
  comment explains why). Where a configured linter or formatter covers a rule, run it and use
  its result as the evidence. If no script, hook or CI step runs it, say so, and report the
  formatting state once for the repository, not once per file. If no rules file says anything
  about comments or style, file nothing here except the items below.
- When the rules leave a question undecided (which comment language applies, whether the code
  counts as public), ask it with the other open questions (SKILL.md section 2). If no one can
  answer, file only what holds under every reading (two languages in one file is a finding
  whichever language is right) and list the undecided rule under Assumptions.
- Whatever the rules files say, these are findings:
  - A comment that contradicts the code it describes. Show the claim and the line that
    disproves it. Workarounds built on the false claim belong in the same finding.
  - A reference that cannot be followed: a document, section, path or note that does not exist
    in the repository or anywhere a reader of the code can open. Check the path before filing.
  - Code disabled by commenting it out, with no date and no reason next to it.
- Report these per rule, not per comment: one finding with the rule, its source file, a count,
  and three to five `file:line` examples (every instance when there are fewer). In change mode, check only comments the change adds or
  edits; older ones go under Pre-existing as one grouped item.

## 14. What is in good shape

Name the parts that are well engineered and why, with evidence: idempotent, writes the audit
record before deleting, bounded concurrency, fails loudly on bad config, redacts secrets, tests
against a real database. This section tells the team which code is the standard to bring the rest
up to. It is never filler, and it is written only after that code passed the same checks as the
rest.

## 15. Conditional dimensions

Apply only when the project has this kind of code.

- **Data migrations:** backfills on large tables, locks and downtime, reversibility, ordering
  against the code deploy (expand, migrate, contract).
- **Privacy and data lifecycle:** what personal data is collected, retention and deletion, export,
  personal data sent to analytics or third-party SDKs.
- **Frontend:** race conditions between requests and UI state, loading and error states, XSS
  through raw HTML insertion, bundle size of new dependencies.
- **Accessibility:** keyboard navigation, labels, contrast, focus management.
- **Internationalization and RTL:** hardcoded strings, date and number formats, layout direction.
- **Feature flags and configuration:** the behavior when a flag or key is missing, dead flags.
- **LLM and agent code:** prompt injection from untrusted input, tool permissions, model output
  trusted as data or as instructions, cost per request.

## Tone (change mode)

Direct and specific, never vague. Thank the author for real work and say what is right before what
is wrong. Point at the exact line and the exact existing code to use. Suggest discussing a large
redesign with the author before anyone reimplements it. Close with a verdict that says exactly
what stands between this and merge.
