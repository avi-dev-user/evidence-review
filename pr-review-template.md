# Change review template

The shape of a change-mode review (diff, branch, PR or MR). It is written so it could be posted
as a review comment as-is, but it is only posted when the user explicitly says so.

Open with one line on scope, revision and what was verified, and whether the code was executed
or reviewed statically (and why). Then:

```markdown
<One or two sentences: what this change does, and whether the design is right. If the change
belongs in a different layer, say so here first and stop the line-level review short: there is no
point polishing code that should not land in this form.>

**Confirmed working**

- **<Claim>.** <How you verified it: readers traced with file:line, library behavior read, test
  run, diff compared.>
- Ran <commands> on this branch: <results in numbers>.

**Blocking**

1. **<Title>.** <What goes wrong, the concrete scenario, and the fix. Point at the existing helper
   or pattern by file and line.>

   ```<lang>
   <a short sketch of the fix, only when it removes ambiguity>
   ```

**Non-blocking**

- <Real but fine as a follow-up. Say why it is not blocking: rare, strictly better than before,
  mitigated elsewhere.>

**Nits**

- <Style, small simplifications, unused leftovers.>

**Pre-existing** (not introduced here, not blocking)

- <Issues found in the surrounding code, with location.>

**Questions**

- <Intent or use case questions whose answer could change the design.>

**Known limitations to note in the description**

- <Accepted gaps the author should state rather than fix now.>

<Merge order and dependencies on other PRs, if any.>

Verdict: <exactly what stands between this and merge, or "approve".>
```

Leave out any section that would be empty. Never invent a finding to fill one.
