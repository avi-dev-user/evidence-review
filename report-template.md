# Project report template

The shape of a project-mode report. Keep the section order. Replace everything in angle
brackets. Write in the language the project's rules require. The generated report starts with
the copyright header the project requires, as an HTML comment.

```markdown
# Code review: <project>

| | |
|---|---|
| Date | <YYYY-MM-DD> |
| Updated | <YYYY-MM-DD>: <what changed, with finding IDs>  (omit on first version) |
| Revision | `<short sha>` (branch `<branch>`) |
| Links | <"Point to <host> at `<sha>`" or "Local paths; the revision is not pushed"> |
| Scope | <read in full / skimmed / skipped, explicitly; library internals read> |
| Assumptions | <defaults used because no one could answer, and reachability assumptions; omit if none> |
| Requested by | <name; omit the row if not given> |

## Summary

<Two or three sentences on the overall shape. Where quality differs between parts, say which
parts and why.>

The most serious problems:

1. **<Plain-language consequence, not the mechanism.>** <One to three sentences.> ([<ID>](#<id>))
2. ...

<Build health in numbers: typecheck, build, N tests passing, audit result, coverage overall and
for the riskiest files. Name untested critical paths.>

### Findings at a glance

Status is **Open** unless the table says otherwise. **Postponed** means accepted for now; the
section gives the reason. **Fixed** means resolved in code; the section says how. **Closed** means
no further work is planned; the section says why.

| ID | Severity | Status | Finding | Reproduced |
|---|---|---|---|---|
| [C1](#c1) | Critical | Open | <one line> | yes / partly / artifact / no / not tried |
| [L1-Ln](#low) | Low | see each item | Hygiene, dead code, duplication, dependencies, docs | see each item |

### How this review was done

- <What was read.>
- <Exact commands run and on which runtime version.>
- <What was reproduced and how: throwaway scripts against which real or stubbed dependencies.
  State that the scripts are not committed.>
- <Other checks: git history scanned for secrets, query plans, timings.>
- <What could not be checked from here.>

Severity scale: **Critical** means exploitable now, with serious impact. **High** means data
loss, cross-tenant access or a broken core feature. **Medium** means a real bug or risk that
needs specific conditions. **Low** covers hygiene, maintainability and docs.

---

<a id="<concern-id>"></a>
## <Requester's named concern, if any: its own section here, above the severity sections>

---

Each finding below follows the same shape. Leave out a severity heading that has no findings.

## Critical

<a id="c1"></a>
### C1. <Title that states the defect>

Status: **Open**

Where: [`<file>:<lines>`](<pinned link>), ...

<What the code does, then what goes wrong, with the concrete scenario. Say "Reproduced:" and the
exact observable result where it was reproduced.>

**Recommendation**

- <Concrete change. Point at existing code to reuse by file and line.>
- <Defense in depth, if relevant.>
- <What fixing a related finding does not close.>

---

## High

## Medium

---

<a id="low"></a>
## Low

- **L1. <Short title>.** Status: Open. Evidence: <label>. <One to four lines with a link and the
  fix.>

Group Low items by theme. If there are more than about 10, keep the most useful ones and
summarize the rest as a count with their themes.

---

## What is in good shape

- <Specific, with evidence.>

<One line: which part is the standard to bring the rest up to.>

## Suggested order of work

1. **[<ID>](#<id>) + [<ID>](#<id>):** <what to do, in one line>.
2. ...
N. Everything else as files are touched.

Done so far: <IDs with status, or "nothing yet">.
```

## Writing a fixed finding (update mode)

Change the status line to `Status: **Fixed** (<date>). See "How it was fixed" at the end of this
section.` Keep the original text. Append:

```markdown
**How it was fixed:**

- <The rule the code now follows, in one sentence.>
- <Each mechanism, with the file and the commit.>
- <What was deliberately not done, and why.>
- **Deferred:** <part left for later, and what mitigates it meanwhile.>
- Tests: <which tests, and that they fail against the unfixed code.>
- <Measured result, before and after, where relevant.>
- **After deploying:** <what to look for in production to confirm.>
```

For a postponed finding: `Status: **Postponed** (<date>). <The compensating control.>` and end
the section with the trigger to revisit and the cheapest first step when it is picked up.
