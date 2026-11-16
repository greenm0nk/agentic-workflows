# Design principles

These are the ideas the skills are built on, each with the reason it's there. The [safety principles](safety.md) are on their own page.

## A blind adversary, reconciled claim by claim

**Why:** a single analyst anchors on the first plausible story, and a second one told to attack the question breaks that anchor.

- Both analysts get the same neutral brief. It holds none of the lead's hunches, because what the lead leans toward is exactly what the adversary mustn't see.
- The lead writes down its own claims before it opens the adversary's, so neither shapes the other.
- Every claim gets a row: agreed, one side only, or in conflict.
- A conflict is settled by a test first. Argument is only for what no test can settle, and lasts at most two rounds.
- Comment triage, product specs, higher-risk code reviews, and review loops over designs and docs all use this protocol.

## Review in rounds until nothing must change

**Why:** fixing what one review found changes the subject, and the fixes need reviewing too.

- Each round is a full, fresh review of the current state, by a new reviewer. From the second round on, its brief also says what changed.
- Recommendations are classed: must-fix, should-fix, nice-to-have, open item, tracked or rejected. Only must-fix and should-fix keep the loop going.
- A decision ledger carries across rounds. Reviewers don't see it, and the loop checks each new recommendation against it.
- A point raised again with no new evidence doesn't count.
- **A clash** is a recommendation that would undo an earlier accepted change. It triggers a search for an approach that satisfies both, and a challenger tries to break the best one. A clash is always documented. A strong solution is applied, and otherwise the user decides.
- Nothing is edited until every agent in the round has reported, so no one judges a moving target.
- Reaching the round cap, which the stakes set, without converging goes to the user. So does a concern that comes back after being fixed twice.

## The gate restarts until one clean pass

**Why:** a fix in one step can break an earlier one, so only a pass that changes nothing proves the whole.

- The steps run in order: build, errors and warnings, format and lint, tests, then a review loop.
- Any step that changes code sends the gate back to the build.
- No step is skipped or shortened for size or time. The gate stops and says why instead.
- Warnings are fixed, never silenced. The only exception is a verified false positive, suppressed as narrowly as possible and listed.
- A failing test gets a root-cause investigation before anything changes, and a test is never weakened to get through.
- If repeated passes don't come back clean, or one fix keeps breaking another step, the user decides.

## The lead reads less code on purpose

**Why:** a lead that wrote the code would be reviewing its own work, and one that didn't can audit it.

- The lead hands the build to code-author, which writes the change and hands it back.
- code-author can't run the gate or a review loop. The lead runs them on its result.
- A code review round runs in a fresh agent after a round that applied changes, or when the code is the session's own work, so the loop never reviews its own work alone.

## The scope rule

**Why:** real problems found outside a task's scope would otherwise creep into the change or get lost.

- A loop fixes what its task owns, and nothing else.
- Findings outside that scope go to the local backlog as items with stable IDs, which the review that found them cites.
- IDs are permanent: never reused or renumbered. Items are closed, never deleted.
- A review comment gets a scope too: fix now, fix in passing, follow-up, track or ask. A valid comment about code the change doesn't touch usually becomes a backlog item, not a change.
- The same rule covers a bug's siblings found elsewhere.

## One item, one branch, one commit, one PR description

**Why:** what limits a change is verification, by the gate and by a human reviewer reading it.

- Work is cut into items, each small enough for one change.
- The commit comes after the gate, and waits for external review. Fixes from that review go into the same commit.
- The PR text reuses the commit's words, so nothing is written twice. Its testing lines come only from the gate's report.

## Tested to be kept

**Why:** a change to the workflow runs on every task after it, so an untested one is a guess repeated many times.

- A change to the workflow's own skills, agents, rules or hooks stays only once its tests or evals pass on its final text.
- If they can't run, or don't pass, the change is reverted, never left in place with a note that it's untested.
- A change from a retro starts with an eval case that fails today, and is reviewed before its eval runs: a review loop for text, the gate for code.

## The taste profile

**Why:** design choices stay consistent when they come from one place the user owns, not from each agent's taste.

- The user's own design decisions live in one taste profile. system-design applies it, project-setup takes its defaults from it, and code-author follows it for structural choices.
- No entry means ask. A design question the profile doesn't settle goes to the user.
- Entries are quoted and cited, never sharpened, extended or merged.
- A one-way door, a choice that's hard to undo, is always asked, even when the profile has a default.
- A new entry goes in only once the user approves its wording.

## Stop cleanly

**Why:** half-done work that looks finished is worse than work that's plainly unfinished.

- When a step can't finish, the run stops there. Nothing is committed, the item stays open, and the work stays on its branch.
- The stop says which step, why, what's on the branch, and what would let it continue.
- A reduced gate isn't a gate. Short of time, or with the user away, the run stops instead.
- Picking up needs no extra state: the item and the repository hold it.

## Evidence over authority, and calibrated confidence

**Why:** acting on a confident claim with nothing behind it does more harm than an honest "unknown".

- Neither analysis wins by default, and the user's framing is a claim to test too.
- Each claim is marked observed or inferred. Observed means it rests on something neither model wrote: a command's output, a test result, or a source quoted word for word.
- Two analyses by the same model are one witness. An agreed claim that drives a verdict still needs an observed basis.
- A concession counts only when it cites the observation that decided it.
- When the evidence doesn't decide, the answer says "unknown" or "unresolved" rather than splitting the difference.
- In code review: no scenario, no finding. Confidence is confirmed, likely with its assumption stated, or a question.
- In a bug hunt, a hypothesis stays inferred until an experiment confirms it, and nothing is fixed until the cause is proven.
