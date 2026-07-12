# House Rules Check

Every repo has "house rules" — the binding rules for how work is done there: conventions, architecture decisions, module docs, process requirements. A PR can be bug-free and still wrong because it ignores those rules. This check makes reading the house rules a required review step, not a thing that happens only when the rules are easy to find.

## Step 1: Discover the House Rules (before reading the diff)

Collect the repo's rule sources. Look for ALL of these — in big repos they are scattered:

| Source | Where to look | What it governs |
| --- | --- | --- |
| `CLAUDE.md` / `CLAUDE.local.md` | Repo root **plus every directory that is an ancestor of a changed file** (a directory's CLAUDE.md only applies to files at or below it) | Coding conventions, forbidden patterns, required workflows |
| `CONTRIBUTING.md`, `docs/` guides | Root, `docs/`, `.github/` | Process: commit format, PR requirements, test expectations |
| ADRs | `docs/adr/`, `docs/decisions/`, `adr/` | Architecture decisions — the "why" behind structure; a PR that quietly reverses an accepted ADR needs an explicit conversation, not a merge |
| Module docs | `BLUEPRINT.md`, `REQUIREMENTS.md`, `README.md` in the changed modules | Module purpose, boundaries, and behavior contracts |
| Machine-enforced config | `.eslintrc`/`eslint.config.*` boundary rules, ArchUnit/Konsist tests, `.editorconfig`, CODEOWNERS, PR/issue templates | Rules the team cared enough to automate — review only what CI can't check |
| Precedent | `git log` of the touched files; comments on previous PRs that touched the same files | How this team actually resolved the same question before |

Quick discovery commands:

```bash
# Rule files near the changed paths
git diff --name-only origin/main...HEAD | xargs -n1 dirname | sort -u \
  | while read d; do ls "$d"/CLAUDE.md "$d"/BLUEPRINT.md "$d"/REQUIREMENTS.md 2>/dev/null; done
ls CLAUDE.md CONTRIBUTING.md .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null; ls docs/adr docs/decisions 2>/dev/null
```

## Step 2: Check the PR Against It

For each rule source found, ask three questions:

1. **Letter** — does any changed line break an explicit rule?
2. **Architecture** — does the change fit the documented structure (module boundaries, layering, approved libraries), or does it route around it?
3. **Process** — did the PR do what the house rules say a change of this kind must do (tests for new endpoints, docs updated, migration guide, changelog, feature flag)?

Also check **intent alignment**: read the linked issue/ticket and any plan. Does the diff do what was asked — no more, no less? Flag scope creep and silent deviations as questions ("the ticket says X, this also does Y — intentional?"), since deviations may be deliberate.

**Docs-sync is part of the check**: if the PR changes a module's behavior, that module's `BLUEPRINT.md`/`REQUIREMENTS.md` must change in the same PR. Stale module docs are a house-rules violation, not a nit.

## Step 3: Hold Findings to the Evidence Bar

House-rules findings are the easiest place to produce noise. Rules for reporting them:

- **Quote the exact rule and the exact line that breaks it.** Name the source file (e.g. "`billing/CLAUDE.md`: 'Monetary amounts MUST use Money' — `RefundRequest.amount: Double`"). No "spirit of the doc" inferences, no style preferences dressed up as rules.
- **Verify the doc actually says it.** Before flagging, re-read the rule — a paraphrase from memory is not evidence. CLAUDE.md is authoring guidance for writing code; not every instruction in it is review-applicable.
- **Respect explicit silences.** A rule that is deliberately suppressed in code (lint-ignore comment, documented exception, ADR superseded-by note) is not a violation.
- **House rules conflict with the code they govern?** If the codebase widely ignores a documented rule, raise it once as a process question ("rule X seems dead — update the doc or enforce it?") rather than flagging every occurrence in the PR.
- **Severity**: an explicit-rule violation is 🔴 `[blocking]` (the team wrote it down for a reason); missing docs-sync is 🟡 `[important]`; intent deviations are questions until answered.

## Checklist

- [ ] Rule sources discovered: root + ancestor CLAUDE.md, CONTRIBUTING, ADRs, changed modules' BLUEPRINT/REQUIREMENTS
- [ ] No changed line breaks an explicit written rule (quoted rule + quoted line for any finding)
- [ ] Change fits documented architecture; no ADR silently reversed
- [ ] Process obligations met: required tests, docs-sync in the same PR, migrations, flags
- [ ] Diff matches the linked issue/plan; scope creep and deviations raised as questions
- [ ] No finding based on memory of a rule — every citation re-read from the source
