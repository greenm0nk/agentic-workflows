---
name: triage-review-comment
description: Triage one or many code-review comments against the actual code on a branch or commit. For each comment it decides whether it is valid and must be fixed, valid but optional, partially valid, invalid, already addressed, or needs discussion, backed by file-and-line evidence, a proposed action and a draft reply. It is built on the adversarial-analysis protocol - the lead agent assesses each comment itself while an independent adversarial agent assesses it blind, then the two are reconciled. Use it whenever the user shares PR review comments or reviewer feedback (pasted text, a file, or a GitHub PR number or URL), asks whether a review comment is right, whether they should address the feedback, to go through or triage review comments, or for help responding to a reviewer - even if they never say triage. Writes results to a topic folder under reports/ in the current directory.
---

# Triage review comments

Reviewers are sometimes right, sometimes wrong, and often partly right about the wrong thing. The goal is a defensible verdict for each comment, grounded in the code at a pinned revision. That lets the user fix what matters, push back with evidence where the reviewer is wrong, and reply quickly.

`<skill-dir>` means this skill's base directory, and `<skills-root>` its parent directory.

This skill runs the **adversarial-analysis protocol** at `<skills-root>/adversarial-analysis/SKILL.md`. Read it first; if it isn't there, invoke the `adversarial-analysis` skill. Everything below is only what this skill specializes. To judge defect claims, use deep-review's checklists:
- `<skills-root>/deep-review/references/areas/`, for each defect class;
- `<skills-root>/deep-review/references/languages/`, for language traps.

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
- **Scratch:** `<cwd>/triage-review-comment-tmp-<YYYYMMDD-HHMMSS>/`, the temp dir. It holds:
  - the comment dump and the briefs;
  - your verdicts, in `mine.md`;
  - one folder per adversary batch, `adversary-<k>/`;
  - `logs/`, for example `gh api ... --paginate > <scratch>/logs/inline-comments.json 2>&1`.

  Delete it at the end.
- **Report type:** `-triage`, for example `reports/ledger/pr-1234-locking-triage.md`. For a re-run on the same PR or branch, add a dated entry to the report's Revision history.
- **Keep the user informed.** At each phase boundary, post one line of status: comments collected, revision pinned, adversaries launched, reconciling, report written.

## 1. Collect the comments

Read-only sources:

- **Pasted text or a file:** split it into individual comments. Keep the author, the file/line anchor and the exact wording.
- **A GitHub PR (number or URL):**
  - First check that `gh` is available (`gh --version`). If it isn't, say so, and ask the user to paste or export the comments, or to point you to a file. Don't install anything without asking.
  - `gh pr view <n> --json number,title,body,headRefName,headRefOid,baseRefName,url,reviews,comments` gives the review bodies and the conversation.
  - Inline review comments need `gh api repos/{owner}/{repo}/pulls/<n>/comments --paginate`. The fields that matter are `path`, `line`/`original_line`, `commit_id`, `original_commit_id`, `diff_hunk` and `in_reply_to_id`.
  - `gh api` can also post to GitHub, so use it here only to read.
  - Skip resolved threads and bot noise unless the user asks for them, and say how many were skipped.
- **Other tools** (Gerrit, GitLab, review bots): normalize them the same way.

Number the comments R1…Rn. Split a comment that makes several independent claims into R3a, R3b and so on. Save the normalized list to the scratch dir.

## 2. Pin the code

Pick the revision to judge against: what the user names (branch, commit, PR head), otherwise `HEAD`. Record the **full SHA**, and for a PR or branch its merge base, which each comment's scope is judged against.

Then locate each comment's code at that SHA. Line anchors drift, so use the comment's `diff_hunk`, quoted code and `original_commit_id` to find it again: `git show <original_commit_id>:<path>`, `git log -L`, `git grep`. If the code changed after the comment was written, check whether that change already addresses it.

Don't check out branches or touch the working tree. Read with `git show <sha>:<path>`, `git diff` and `git grep <pattern> <sha>`. Any git command that reads paths you didn't name (the diff, the grep, `git show <commit>`) gets `-- ':/' ':(top,exclude,glob)**/reports/**/*.md' ':(top,exclude,glob)backlog/**/*.md'`, which keeps committed suite reports and backlog items out, and every diff gets `--no-ext-diff --no-textconv`. If you have to run a test to settle a comment, ask first, and use a worktree inside the scratch dir. Remove it afterwards with `git worktree remove --force <path>`: `--force` is needed because the test run leaves untracked output in it, and it's your own throwaway checkout.

## 3. Classify each comment

- **Defect claim** (bug, crash, leak, race, deadlock, security hole, performance regression, breaks callers): full protocol.
- **Design or approach** (wrong abstraction, "should use X", architecture concern): full protocol. These often end as "needs discussion". A refactor that reaches past the PR is usually valid but out of scope (Step 5).
- **Test request** ("add a test for…"): check whether a plausible bug would slip past the current tests (`<skills-root>/deep-review/references/areas/tests.md`). Lighter treatment. A test for behavior the PR doesn't add or modify is out of scope.
  - Grade it by that file's severities: a missing test for Medium behavior is "Valid: must fix", and for Low behavior "Valid: optional".
  - If the test would need new test infrastructure (a framework, runner or test target), say so. Inside a `recursive-review` loop, that makes it an open item.
- **Style, nit, naming or docs:** quick check against the project's conventions (agent instructions files, linter configs, nearby code). Don't run linters or formatters; they belong to code-quality-gate. No adversary unless the user asks.
- **Question** ("why is this…?"): answer it from the code and draft the reply.

## 4. Run the protocol, batched

- **Brief (one per batch):**
  - the comments verbatim, with their anchors;
  - the pinned SHA and where the repository is;
  - the PR's title and description.

  Never include your own verdicts.
- **Adversary:**
  - Use `references/adversary-prompt.md`, with subagent type `review-adversary` if it exists (it has no Agent or Skill tool); otherwise use `general-purpose`.
  - Give one subagent each batch of up to about 8 substantive comments. Larger sets get more batches, at most 4 agents in parallel, each with its own folder `<scratch>/adversary-<k>/`.
  - Paste the deep-review checklists and language files relevant to the batch into its prompt, so it never reads files outside the repository.
  - It writes `<scratch>/adversary-<k>/result.md`, replies only `DONE: <path>`, and spawns nothing.
  - Launch it before you start your own assessment.
- **You:**
  - Assess each comment with the deep-review checklist for its defect class. For example, a deadlock claim gets `areas/concurrency.md` plus the language file.
  - Commit your per-comment verdicts to `<scratch>/mine.md` **before** you open any adversary's result.
- **Reconcile** each comment as the protocol describes:
  - agreement: check for a shared blind spot. Both assessments come from one model, so agreement is one witness: a verdict of "must fix" or "invalid" still needs an observed basis, such as the code at the pinned SHA quoted directly, a repro or a tool result;
  - one-sided points: verify them against the evidence;
  - conflicts: settle by test first, with the cheapest repro or check that tells the positions apart. Use a rebuttal round only when no test can settle it (at most 2, with replies written to the adversary's folder). A concession must cite the observation that decided it.
  - A comment about naming, structure, style or API shape can be a matter of priorities, not facts. Judge it as an evaluative or design question (`<skills-root>/adversarial-analysis/references/question-types.md`): give a verdict that depends on the stated priorities, and say which ones.

## 5. Verdicts

| Verdict | Meaning |
|---|---|
| **Valid: must fix** | The claimed defect or risk is real and reachable at the pinned SHA. |
| **Valid: optional** | Correct and reasonable, but a preference or minor improvement; declining it breaks nothing. |
| **Partially valid** | Part of the claim holds. Say exactly which part, and what the right fix is (often not the one the reviewer suggested). |
| **Invalid** | The code already prevents it, or the premise is wrong. Cite the evidence. |
| **Already addressed** | Fixed after the comment (cite the commit) or handled elsewhere (cite file:line). |
| **Needs discussion** | Hinges on intent, requirements or a trade-off the code can't settle. |

Both sides give the same fields for every comment, which keeps reconciliation symmetric:
- the verdict and its confidence;
- the reviewer's claim, restated;
- evidence;
- for a valid claim, the concrete scenario;
- **whether the reviewer's suggested fix is correct** (yes / no / partly, and why);
- what would change the verdict.

Your report then adds a proposed action (a fix sketch, a backlog item, or "no change") and a short, courteous draft reply that rests on the evidence rather than on opinion. For a valid or partly valid claim it also adds the **scope**: the outcome of `<skills-root>/recursive-review/references/task-scope.md`, judged against the merge base (fix now, fix in passing, follow-up, track or ask), with the test that decided it. A claim about code the PR neither adds, reaches, relies on nor makes worse is out of scope; unless it's severe, or a small fix in a function the PR modifies, its action is a backlog item, not a change. Scope doesn't change the verdict: the verdict says whether the reviewer is right, and scope whether this PR acts on it.

**Escalation** follows the protocol: only for divergences that reveal a fundamental issue. Examples:
- a comment shows the PR's approach can't meet its goal;
- comments contradict each other in a way only the user can settle;
- the code being reviewed isn't the revision the comments were written against, and that changes the answers.

A "Needs discussion" verdict on a single design comment is a verdict, not an escalation, unless it blocks the rest.

## 6. Report

Write `reports/<topic>/<slug>.md`:

```markdown
# Review triage: <PR / branch>

- **Date:** YYYY-MM-DD · **Revision:** <full SHA> · **Comments:** <n> (<source>; <k> skipped: resolved or bots)
- **Verdicts:** must-fix <a> · optional <b> · partial <c> · invalid <d> · addressed <e> · discuss <f> · out of scope <t> (tracked: <IDs>)
- **Method:** own assessment + independent adversarial agent(s), <n> rebuttal rounds | single-agent fallback

## Summary
| # | Location | Author | Claim (short) | Verdict | Action |
|---|---|---|---|---|---|

## R1. <short claim> (`path:line`, @author)
> <comment, quoted>

**Verdict:** Valid: must fix (high)
**Evidence:** <file:line / command output>
**Scenario:** <for valid claims>
**Suggested fix correct?** <yes / no / partly, and why>
**Scope:** <fix now | fix in passing | follow-up: <ID> | tracked: <ID, or "to file"> | ask: <why>>, by <the test or row>
**Reconciliation:** <where your verdict and the adversary's differed, and how it was settled; or "agreed">
**Action:** <fix sketch, or "no change">
**Draft reply:** <one to three sentences>

## Escalations and open questions
## Revision history
- YYYY-MM-DD: <initial triage / what changed>
```

Then print a short summary: counts by verdict, the must-fix list, and the report path. When you're running as a round inside `recursive-review`, the triage goes into the loop's report instead of a separate file.

**Applying fixes.** Triage doesn't change code unless the user asks. Inside a `recursive-review` loop, don't apply fixes or start a gate run yourself: the loop applies the changes and its verification step covers the gate. Otherwise, when the user asks you to address the valid comments:
1. Apply the fixes.
2. Invoke `code-quality-gate` with the Skill tool before calling them done.
3. Record each comment's fix in the report. A big or hard-to-undo fix is a follow-up instead (the scope rule's "Follow-ups").

**Out-of-scope comments get backlog items.** Inside a `recursive-review` loop, the loop files them with its batch and puts their IDs into the draft replies. Standalone, file every comment whose scope is "track" in one batch before writing the report, as the scope rule's "Filing" describes (`- caller: triage-review-comment`), one entry per comment, so each draft reply can link its ID ("tracked as WF-014"). Filing is local; posting the reply still needs approval.

**Outward-facing actions need explicit approval.** Posting replies and resolving threads change state outside this machine. Do them only when the user asks, and show exactly what will be posted first. Pushing fixes is never yours to do: the user pushes.
