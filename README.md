# agentic-workflows

Reusable workflows for AI coding agents: skills and agents that carry a piece of work from a backlog item to a local commit. On the way, they settle any open design choice, build the change, and run it through review gates. The user answers the questions that matter, reviews the result, and pushes. Claude Code is the first agent they're built for; the aim is workflows any coding agent can use.

This repository shows how the system is designed. It isn't a package to install.

## Who it's for

- **The system** is built for one user working with an AI coding agent in their own repositories.
- **This public copy** is for engineers who build workflows for AI coding agents, and want to see how review loops, a quality gate and safety rules can fit together.

## The docs

| Page | What it covers |
|---|---|
| [Architecture](docs/architecture.md) | The parts, how they're grouped, and how they hand work to each other |
| [Work-item flow](docs/work-item-flow.md) | One item from the backlog to a commit, and several items at once |
| [Design principles](docs/principles.md) | The ideas behind the skills, and why each one is there |
| [Safety principles](docs/safety.md) | The rules agents work under, and why |
| [Notice](NOTICE.md) | All rights reserved |

## The skills

A skill is a written procedure the agent loads when a task matches its description. Each `SKILL.md` below is the procedure the skill follows.

| Skill | Role | What it does |
|---|---|---|
| [work-item](skills/work-item/SKILL.md) | Orchestration | Runs backlog items end to end, from start to commit and PR text, and stops cleanly when a step can't finish |
| [product-spec](skills/product-spec/SKILL.md) | Planning | Turns an idea into a short spec and an honest verdict, from build to don't build |
| [system-design](skills/system-design/SKILL.md) | Planning | Designs or reviews a system from the user's own architecture decisions, and asks on every one-way door |
| [work-breakdown](skills/work-breakdown/SKILL.md) | Planning | Turns a design or spec into ordered backlog items, each small enough for one change |
| [debug](skills/debug/SKILL.md) | Diagnosis | Proves a bug's root cause before anyone fixes it, then fixes it with a regression test |
| [code-quality-gate](skills/code-quality-gate/SKILL.md) | The gate | Builds, fixes warnings, formats, runs affected tests and reviews the change, restarting until one clean pass |
| [recursive-review](skills/recursive-review/SKILL.md) | Review | Reviews and fixes in rounds, until a round asks for no must-fix or should-fix change |
| [deep-review](skills/deep-review/SKILL.md) | Review | Multi-pass defect review of code changes, where every finding needs a concrete failure scenario |
| [adversarial-analysis](skills/adversarial-analysis/SKILL.md) | Review | Two-sided analysis: an independent adversary works blind, and the two are reconciled claim by claim |
| [triage-review-comment](skills/triage-review-comment/SKILL.md) | Review | Judges code-review comments against the actual code, with evidence, an action and a draft reply |
| [backlog](skills/backlog/SKILL.md) | Records | Files and manages tasks and bugs in the project's local backlog, each with a stable ID |
| [pr-description](skills/pr-description/SKILL.md) | Records | Writes a change's pull request text from its commit, with testing claims taken only from the gate's report |
| [project-setup](skills/project-setup/SKILL.md) | Setup | Sets up a repository's formatter, linter, tests and CI, so the gate has something real to run |
| [retro](skills/retro/SKILL.md) | Learning | Looks back over recent work for repeated mistakes, and proposes changes for the user to decide on |

## The agents

| Agent | What it does |
|---|---|
| [code-author](agents/code-author.md) | Builds a change under the code-craft rules: reuses before it writes, keeps nesting and complexity low, and hands the change back for the lead to gate. It can't run the gate or a review loop itself. |
| [review-adversary](agents/review-adversary.md) | The independent analyst the review skills launch, as an adversary, a challenger, an area reviewer or a fresh reviewer. It can't launch agents or skills, so a review can never recurse through it. |

## Layout

```
README.md
NOTICE.md
docs/                  design pages
agents/                the two agent definitions
skills/<name>/SKILL.md one file per skill
```

## What isn't here

- **Each skill's reference files, scripts and evals stay private.** A link inside a `SKILL.md` to `references/…` or `scripts/…` points to a file this repository doesn't include.
- **The hooks that enforce the safety rules stay private too,** along with the user's own instructions and every work record: backlog items, reports and plans.

## License

All rights reserved. See [NOTICE.md](NOTICE.md).
