---
name: code-quality-gate
description: Standard quality gate for code changes in any language. After every set of changes it builds every target that compiles the changed code and fixes all errors and warnings until the build is clean. It applies the project's or the language's native formatter, runs every test the changes affect (not only new ones) and fixes failures until all pass. It then runs recursive review rounds over the changed code and impacted areas. Any gate that changes code restarts the sequence, and it stops only when every gate passes in one clean pass. Per-language and per-project specifics (build systems, target discovery, formatters, affected-test selection) are in its references. Use it after making code changes and before declaring work done; when asked to make sure code builds and tests pass, clean up warnings, or get changes merge-ready; and whenever deep-review, triage-review-comment or recursive-review apply code fixes. Writes a gate report to a topic folder under reports/ in the current directory.
---

# Code quality gate

Code is done when it builds cleanly, is formatted, passes every test it could have broken, and has survived review. This skill runs those checks as gates, in order, until **one complete pass makes no changes**.

`<skill-dir>` means this skill's base directory, and `<skills-root>` its parent directory.

## The gates

| Gate | Passes when |
|---|---|
| G1 Build | every target that compiles the changed code builds |
| G2 Errors and warnings | zero errors, and zero warnings in first-party code those targets build |
| G3 Format and lint | the native or project formatter leaves the changed files unchanged, the configured linters are clean, and the changed lines follow the project's written guidelines |
| G4 Tests | every test affected by the changes passes, or there are no tests |
| G5 Review | a recursive review of the changed code and impacted areas converges |

**Any gate that changes code sends the gate back to G1.** It passes only when a full G1-to-G5 pass changes nothing (G5 has its own rule), and a report records it. **Never skip or shorten a gate** for size, cost or time: stop, and say why.

**Detecting changes.** A gate changed code if you edited a file, ran a write form (a formatter or a fixer), or the change fingerprint differs before and after the gate. Before the first gate, read `references/fingerprint.md` and take the fingerprint exactly as it says, from any directory.

**Trust.** The gate's commands run the repository's own code and follow its configuration, which can point a tool at any path. On code you don't trust, ask the user before starting the gate.

## Working files

<!-- suite-rules:start: identical in all five review skills; adversarial-analysis/scripts/check_suite_consistency.py checks it -->
**Rules shared by the review suite:**
- **Paths and time.** Resolve the current working directory at runtime with `pwd`, and take timestamps with `date +%Y%m%d-%H%M%S` (report dates with `date +%Y-%m-%d`). Never assume a path.
- **Reports** go to `<cwd>/reports/<topic>/<slug>.md`, organized by topic, named after what the report covers: a skill, component, feature or project, never a catch-all such as `workflow`. The slug is lowercase and hyphen-delimited, names the subject, and ends with the skill's report type, such as `-code-review`. If a report on the same subject already exists, add a dated entry to its Revision history instead of creating a new file.
- **Inside a `recursive-review` loop,** use the loop's round folder as your temp dir and leave its deletion to the loop, and put your results into the loop's report instead of a separate one.
- **The suite's own outputs** are Markdown under a `reports/` folder or `<root>/backlog/`, untracked folders whose name contains `-tmp-`, and `deep-review-workspace/`. Leave them out of every scope and search, unless the user names one as the subject or as evidence. Markdown in a skill's own folder or an eval fixture's `files/` tree is project content. Ignore `-tmp-` and `reports` in the names of the folders that contain the repository or the working directory. A project's own code under `reports/` stays in scope. Without git, leave out every `*-tmp-*` folder; the suite's Markdown stays out either way. In a Grep search, exclude them with `!*-tmp-*/` and `!deep-review-workspace/`, plus `!**/reports/**/*.md` and `!<root>/backlog/**/*.md`, `<root>` relative to the working directory (`!backlog/**/*.md` at the root; below it, no glob: drop backlog hits by hand). Where a `reports` folder contains the search root or the working directory, its glob drops project Markdown too, and so does its git pathspec when that folder is inside the repository: leave them out there, and drop report hits by hand. The Glob tool can't exclude, so drop those paths from its results. Never use `!**/*-tmp-*/**`: it also matches the folders above the search root, and drops everything. The globs also hide tracked `-tmp-` folders and project Markdown under `reports/`, so search those separately.
- **Never push.** Commit finished work once its gate passes, on a branch; anything else only when asked. Leave `reports/` out even then, unless they explicitly include it. The first report written inside a git repository adds `/reports/` (from a subdirectory, its path from the root) to the file `git rev-parse --path-format=absolute --git-path info/exclude` names.
- **Shell commands go through the Bash tool.**
- **Never truncate command output** with `head`, `tail`, or a filter that throws lines away before anyone has read them.
  - **Long-running commands** (builds, test suites, installs, validators, paginated fetches, anything run in the background) always redirect their full output to a file in the temp dir, for example `cmd > <temp>/logs/<step>.log 2>&1`. Read that file afterwards. Searching a saved log with Grep is fine.
  - So does any command whose output may be too large for the tool to return in full.
  - **Short commands** run directly; read what the tool returns.
- **Never use `echo`** (or `printf`, `Write-Output`, `Write-Host`) in commands, and don't print status lines from scripts. Write files with the Write tool.
- **Review subagent prompts** carry the rules block from `<skills-root>/adversarial-analysis/references/subagent-rules.md`, pasted in full.
- **After the conversation is compacted,** Read this skill's `SKILL.md` in full, and every reference file the current step follows, before continuing. Compaction keeps only the first 5,000 tokens of each skill, and re-invoking an unchanged skill may add only a short note.
<!-- suite-rules:end -->

This skill's own working files:
- **Temp dir:** `<cwd>/code-quality-gate-tmp-<YYYYMMDD-HHMMSS>/`. It holds:
  - `baseline/`, with the starting state;
  - `logs/`, one file per command run, for example `<temp>/logs/i1-g4-go-test.log`;
  - `iteration-<n>/`, with what each pass changed, saved as patches (`git diff --no-ext-diff --no-textconv`), never as copied source files. Whole-project tools scan the temp dir too.

  Delete it at the end.
- **Report type:** `-gate`, in the format in `references/report-format.md`.
- **Re-invoke this skill with the Skill tool, with the same arguments, at the start of every pass and after any user message.** That continues the same gate run, with its temp dir and pass history; don't restart Step 0.
- **Keep the user informed:** one status line per gate.

## Step 0: Scope and baseline

1. **What changed.** The gate's scope is the change under review:
   - uncommitted work: `git status --porcelain --untracked-files=all` and `git diff HEAD --stat --no-relative`;
   - the branch's commits as well, when the user names a branch or asks about merge readiness, or when the working tree is clean: run `git merge-base HEAD origin/HEAD` as its own command (if that fails, fall back to `origin/main`, `main` or `master`), record the SHA, and add `git diff <sha> --stat --no-relative`;
   - without git, the files the user names or the files you edited.

   Leave out the review suite's own outputs (the shared rules define them). Also leave out build and test output (in-tree build folders, CTest's `Testing/`, coverage files, caches) and agent state (the agent's own worktrees), and name what you left out in the report.

   **Source files in suite temp folders inside the project** break whole-project tools: before G1, deal with them as `references/impact.md` ("Suite temp folders") describes. If the scope is empty (nothing uncommitted, and no commits ahead of the merge base), say there's nothing to gate and stop.
2. **Versions, on a full run.** List version changes (`references/report-format.md`, "Version changes"). If one needs asking and nobody can answer: write the report with status "Paused: awaiting your decision", keep the temp dir, and stop before G1. That isn't a pass. An answer resumes with the rest of Step 0 and the full G1 to G5.
3. **Project root.** Find the root with `git rev-parse --show-toplevel`, or the folder holding the affected code's `go.mod`, `Cargo.toml`, `package.json` or `pyproject.toml`. The whole-project commands act on the current directory: from a subdirectory they cover only that subtree, or fail (`cmake --build build`). When the current directory isn't the root, pass the root to the tool (the ecosystem file names the form, such as `go -C <root>`) rather than `cd`, or list what they didn't cover under "Not verified".
4. **Ecosystem.** Detect the build system, formatter, linters and test runner from the repository's files, then read the matching `references/ecosystems/<language>.md`. Project rules (agent instructions files, CONTRIBUTING, CI configs) override the generic defaults.
5. **Impact.** Follow `references/impact.md`: map each changed file to the targets that compile it, then to everything that depends on those targets, then to the tests that exercise them. Write the map to `baseline/impact.md`. If the full set is too large to build or test in reasonable time, agree a scope with the user rather than silently narrowing it.
6. **Baseline status.** Record what was already failing *before* the change:
   - existing warnings in the affected targets;
   - failing tests.

   When it's cheap enough, build and test the base revision in a worktree, as `references/impact.md` ("Baseline worktree") describes, and remove it before G1.

## G1 Build

Build every target from the impact map: libraries, executables, test binaries, examples, and every platform or configuration variant the project builds in CI that you can build here. A target you can't build locally goes in the report as **not built**, with the reason.

## G2 Errors and warnings

Read the full build logs. Fix every error. Fix every warning in first-party code built by the affected targets.

- **Never silence a warning to pass the gate.** No suppression comments, pragmas or flags (`# noqa`, `#pragma` disables, `-Wno-*`), and no weakened compiler or linter settings (new ignores or excludes). The only exception is a warning verified to be a false positive: then suppress it as narrowly as possible, with an inline justification, and list it in the report. The rule binds the change under gate too: a suppression it adds counts as the warning it hides, unless it meets that exception. Before calling G2 clean, find them as `references/suppressions.md` describes.
- **Leave vendored, generated and third-party code alone.** Report its warnings instead.
- **Pre-existing warnings** (present in the baseline) in files the change didn't touch: if only a few, fix them. File many (roughly more than 20) instead of asking (`references/report-format.md`, "Filing"). If fixing a few needs changes unrelated to the work, ask the user once, and file those they defer. Filed warnings are an exception: G2 is then "pass with <n> pre-existing warnings filed", and the gate's status is "Passed with filed exceptions", or "Passed with user-accepted exceptions" for those the user deferred.
- **After fixes,** rebuild the affected targets and repeat G1 and G2 until the build is clean.

## G3 Format and lint

- **Formatter:** use the project's configured formatter. If there's none, use the language's native formatter (gofmt, rustfmt). If neither exists, skip formatting and note it.
  - Format **the changed files only**.
  - Check first with the project-wide check form. If only the changed files would change, the project-wide write form is fine. The ecosystem file names both forms.
  - Otherwise format just the changed files with the ecosystem file's per-file command (`gofmt -w <files>`, `black <files>`).
- **Linters:** run the linters the project configures and treat their findings in the changed code like warnings in G2. Run them in check mode, and apply fixes deliberately (`ruff check --fix <files>`), so every change is one you chose.
  - List changed files a linter's configuration skips as "not linted" in G3's evidence. For ruff, read `references/ecosystems/python.md` first: it's judged by its exit code, in one exact form.
- **Craft rules and project guidelines** (`references/code-craft.md`; agent instructions files, CONTRIBUTING, style docs read in Step 0): run that file's checker and craft audit on the changed code yourself. A violation counts like a lint warning: fix it, then return to G1. G5's reviewer doesn't judge style.
- **Any formatting or lint fix is a change,** so return to G1.

## G4 Tests

1. **Run every test the impact map selects:**
   - tests of the changed code;
   - tests of everything that depends on it;
   - new and updated tests.

   Use the ecosystem's affected-test selection where it exists. Otherwise run the full suite of each affected package or module. When unsure, run more, not less. When the project configures coverage, collect it in this run (`references/evidence.md`).
2. **Existing tests must keep running.** An existing test of the affected code that the change stops from running or passing, by any means, counts as failing unless the change deliberately removes what it tests. New tests that can't run here are Not verified. `references/suppressions.md` says how to find them.
3. **No tests for the affected code,** and none the change stopped: record "no tests exist for the affected code" and pass G4.
   - Whether new tests get written is G5's business (recursive-review's round classes).
   - Don't add a test framework, runner configuration or test target unless the user asks.
4. **For any failure, investigate the root cause before changing anything:**
   - **Introduced** (it passes at baseline): fix the code. Change the test only if the investigation shows the test's expectation is wrong for the new, intended behavior, and say so in the report.
   - **Pre-existing** (it fails at baseline too, unrelated to the change): ask the user whether to fix it now or defer it. Never skip, delete or weaken it to get through the gate. A deferred failure is filed, and leaves G4 **not passed**: record "not passed: <n> pre-existing failures deferred by the user", never "pass", and the gate's status is "Passed with user-accepted exceptions".
   - **Flaky:** rerun to confirm. Report it, and fix the flakiness if it's in scope. Don't retry until it happens to pass.
5. **Rerun** the failing tests and then the whole affected set until everything passes. Fixes are changes, so return to G1.

## G5 Review

Invoke `recursive-review` with the Skill tool (don't Read its `SKILL.md` first), with `deep-review` as its round reviewer, for every change, even one line or a manifest. Reviewing it yourself doesn't count.
- **Scope:** the gate's scope from Step 0, including every change made by earlier passes, plus the impact map's areas.
- **Arguments:** only `<temp>/g5.md`, the same on every pass. Write it first: this temp dir, the scope and the user's request as the loop's requirements baseline, and `<temp>/evidence.md` (`references/evidence.md`).
- **Verification:** inside that loop, each round's verification step is G1 to G4 of this gate, in verification-only mode (below).
- **Nesting:** the review never starts a nested G5.

**When G5 counts as passed.** If the loop converged, the final state has passed every gate: its last round changed nothing, and its verification ran G1 to G4 after its last change.
- **Open items:** if the loop converged with open items that don't block (Medium or Low questions, say), G5 passes with them listed, and the gate's status says "Passed with open items". Anything the user declined that deep-review's verdict still counts makes it "Passed with user-accepted exceptions": a blocking open item, or a must-fix or should-fix change declined in a clash escalation.
- **Verification results:** the loop's user-accepted verification exceptions make the status "Passed with user-accepted exceptions"; its unverified changes join this gate's Not verified section.
- **Paused:** if the loop paused on blocking open items, the gate's status is "Paused: awaiting your decision", which isn't a pass. Keep the temp dir, and resume as `references/report-format.md` ("Paused") describes.
- **Escalated:** if the loop escalated, the gate hasn't passed.

## Verification-only mode

A review loop that applies fixes invokes this skill to verify them. In that mode, read `references/verification-only.md` before Step 0 and follow it.

## Stop, continue or escalate

- **Pass:** one full G1-to-G5 pass completed with no changes, or G5 counted as passed under its rule. List versions again, as Step 0 says, because a fix can change one. Write the final report, and delete the temp dir.
- **Pass with exceptions** or **Paused:** as Step 0, G2, G4 and G5 describe; the status says which.
- **Continue:** a gate made changes. Start the next pass at G1.
- **Escalate** (write the report first, with status Escalated), with AskUserQuestion, instead of stopping silently, when:
  - 5 full passes haven't produced a clean pass;
  - a fix for one gate keeps breaking another (oscillation);
  - the toolchain or the environment is missing;
  - tests need external services that aren't available;
  - a fix needs a product or design decision.

  Give the history and your recommendation. Non-interactively, stop, and lead with the escalation and a conditional verdict: what each answer would lead to.
