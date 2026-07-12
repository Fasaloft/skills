# Skills

This repository contains specialized **agent skills**. Each skill is a self-contained
directory with a `SKILL.md` file that defines its behavior, plus any supporting references,
assets, and scripts.

---

## Available Skills

| Skill | Description |
| --- | --- |
| [`deep-review`](deep-review/SKILL.md) | Systematic, evidence-based PR/MR review. Host-agnostic (GitHub `gh` / GitLab `glab`); covers React 19, TypeScript, Kotlin, Python, AWS Lambda/serverless, modern C++, Qt/QML, module boundaries, and house-rules alignment. |
| [`university-project-review`](university-project-review/SKILL.md) | Two-phase review of a student's full-stack web app across 14 quality categories; writes findings to `REVIEW.md`. |

---

## Installation

Skills are installed with the [`skills`](https://github.com/vercel-labs/skills) CLI, which
pulls them straight from this GitHub repo — no clone required.

**See what's available in this repo:**

```bash
npx skills add Fasaloft/skills --list
```

**Install a specific skill:**

```bash
npx skills add Fasaloft/skills --skill deep-review
```

**Install everything:**

```bash
npx skills add Fasaloft/skills --all
```

**Target a specific agent** (defaults to the ones it detects):

```bash
npx skills add Fasaloft/skills --skill deep-review -a claude-code
```

**Manage installed skills:**

```bash
npx skills list             # list installed skills
npx skills update           # update all skills
npx skills remove <skill>   # remove a skill
```

---

## Adding a New Skill

1. Create a new directory: `mkdir <skill-name>/`
2. Add a `SKILL.md` file inside with a YAML frontmatter block:
   ```yaml
   ---
   name: <skill-name>
   description: <Concise description of what the skill does>
   ---
   ```
3. Add detailed instructions, exploration commands, and any supporting `reference/`,
   `assets/`, or `scripts/` files.
4. Add the skill to the **Available Skills** table above.

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

When modifying a skill, keep its `SKILL.md` metadata (name, description) accurate and update
the **Available Skills** table in this `README.md` when adding or renaming a skill.
