# Skills

This repository contains specialized **agent skills** for the OpenCode agent system. Each skill is a self-contained package of instructions, workflows, and exploration commands that enable the agent to perform specific, complex tasks autonomously.

Skills are stored as directories under the repository root, each containing a `SKILL.md` file that defines the skill's behavior.

---

## Available Skills

### `university-project-review`

Performs a comprehensive, two-phase review of a university student's full-stack web application project. It simulates a lecturer evaluating a software engineering course submission by orienting to the codebase and assessing its quality across **14 predefined categories**.

**Invocation:** Explicitly triggered when the user types `/university-project-review`.

**How it works:**
- **Phase 1 — Project Orientation:** Explores directory structure, package.json files, README, env templates, and monorepo configs to map the project and detect the tech stack.
- **Phase 2 — Category Reviews:** Runs targeted bash and grep explorations to audit source code against a checklist in each category, assesses file sizes and layering against rules of thumb, and writes evidence-based findings.

**14 Categories Reviewed:**

1. Component Library
2. Styling
3. Loading Data
4. Environment Variables
5. REST API Design
6. Database
7. Backend Design Patterns
8. Auth
9. Testing
10. Logging & Monitoring
11. Error Handling
12. Security
13. Database Transactions *(Added)*
14. Page Decomposition *(Added)*

**Output:** A `REVIEW.md` file written to the project root containing:
- A summary table with status per category (✅ Good | ⚠️ Concerns | ❌ Issues | N/A)
- Evidence-based findings citing specific file paths and code snippets
- Actionable recommendations for the student

**File:** [`university-project-review/SKILL.md`](university-project-review/SKILL.md)

---

### `deep-review`

Performs a systematic, evidence-based review of a pull request or merge request. It is **host-agnostic**, working on both GitHub (`gh`) and GitLab (`glab`), and reviews changes against the repository's own house rules (CLAUDE.md, ADRs, module docs) rather than generic best practice alone.

**Invocation:** Triggered when reviewing PRs/MRs or code changes — code review, architecture reviews, security audits, or bug finding.

**How it works (four phases):**
- **Phase 0 — Platform:** Detects the host from the git remote and loads the matching command map before running any host command.
- **Phase 1 — Context:** Reads the PR description, linked issue, CI status, and the repo's house rules (`CLAUDE.md`, ADRs, module `BLUEPRINT.md`/`REQUIREMENTS.md`).
- **Phase 2 — Load References (mandatory):** Inspects the diff and reads every matching reference guide (React, TypeScript, Kotlin, security, performance, architecture, etc.).
- **Phase 3 — Review:** Assesses architecture and design first, then line-by-line for logic, security, performance, and maintainability.
- **Phase 4 — Report:** Emits a strict-format report with a verdict, severity-grouped findings, a module coupling map (Mermaid), and the references used, then posts it as a single PR/MR comment.

**Coverage:** React 19, TypeScript, Kotlin (Android and backend), modular-monolith module boundaries, and house-rules alignment.

**Supporting files:**
- `reference/` — the standards guides that carry the actual review criteria
- `assets/` — the PR review template and quick checklist output contracts
- `scripts/pr-analyzer.py` — optional triage helper for large diffs

**Output:** A structured review comment posted to the PR/MR (verdict, findings by severity, coupling map, references used).

**File:** [`deep-review/SKILL.md`](deep-review/SKILL.md)

---

## Adding a New Skill

To add a new skill:

1. Create a new directory: `mkdir <skill-name>/`
2. Add a `SKILL.md` file inside with a YAML frontmatter block:
   ```yaml
   ---
   name: <skill-name>
   description: <Concise description of what the skill does>
   ---
   ```
3. Add detailed instructions, exploration commands, and output templates.
4. Update this `README.md` to include the new skill.

---

## Project Structure

```
.
├── README.md                           # This file
├── deep-review/
│   ├── SKILL.md                        # Skill definition and workflows
│   ├── assets/                         # Output templates (PR review template, checklist)
│   ├── reference/                      # Standards guides (React, TypeScript, Kotlin, security, …)
│   └── scripts/                        # Helper scripts (pr-analyzer.py)
└── university-project-review/
    └── SKILL.md                        # Skill definition and workflows
```

---

## Contributing

When modifying a skill, ensure you also update the corresponding `SKILL.md` metadata (description, category counts) and this `README.md` if new skills are added.
