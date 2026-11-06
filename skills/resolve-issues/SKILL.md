---
name: resolve-issues
description: Use when the owner says "resolve issues" in a repo. Verify and close what is done, plan the rest (asking only real unknowns), build it with subagents, verify with ONE CI run on a pull request, merge into the target, close the issues and watch its CI/CD to green.
version: 4
pcp_format: 2
---

# Resolve issues (verify, plan, run multi-agent work, one CI run, close)

Works for any repo. "Resolve issues" alone means this whole procedure, end to end: no further instructions are needed. The coordinator (you) stays free for the owner: subagents explore, implement, merge and fix; you plan, launch, relay, decide what goes back to the owner and close issues.

## HARD RULE: the full suite runs exactly ONCE, as the pull request's CI
The one full run (full unit, build, e2e) is the CI of the pull request from the session branch into the target branch, opened after the LAST workstream has merged into the session branch. Nobody runs the full unit suite, the build or e2e locally: not workers, not the integration agent, not the coordinator, not "to be safe", not between waves. Locally the most anyone runs is the fast pre-commit checks (lint, typecheck, format) plus the unit tests next to a change, and after resolving a merge conflict, the unit tests of the conflicted files. Work the owner adds mid-run becomes more workstreams merged first; the PR waits for it rather than CI running twice. A red CI is fixed and pushed to the same PR (that is still the one run, finishing). Fixing it is the one exception to "nothing runs locally": before pushing a CI fix, verify it locally — the failing tests and failing e2e projects first, then the full unit suite (and the full e2e suite when feasible) — so the push turns CI green in one go rather than another CI round.

## 1. Gather and verify (read-only)
- List open issues WITH comments: addenda in comments often add whole items. Read any memories or docs an issue references.
- Read the repo's agent and contributor docs (CLAUDE.md, CONTRIBUTING, ARCHITECTURE, DESIGN). Their boundaries bind every design.
- Note the session branch, the target branch (the repo's docs decide, e.g. PCP: `develop` by PR), the release rules ("never bump versions unless asked") and the check commands.
- Verify every open issue against the code (at the target branch's head) with the explore agents of step 2: resolved (cite the code that does it), partial (what is left) or open. A fix committed to the target branch counts as resolved.
- Close each already-resolved issue by hand with a comment naming the commit and code that resolve it (auto-close does not fire on these repos). A partial one stays open and its remainder becomes the work.

## 2. Explore
- At most 3 read-only explore agents in parallel, each with a distinct area (e.g. gateway/API surface; runtime/browser/execution; storage/permissions/UI/services).
- Ask for a per-issue verdict (resolved/partial/open) with file:line references, existing helpers to reuse, the choke points every call passes through, where a fix would go, and the constraints that block designs (no background key, no child processes, stateless transport, a CLAUDE.md boundary the fix would cross...).
- Tell the owner in one line what is running.

## 3. Summarise, then ask only real unknowns (AskUserQuestion)
- FIRST: show the owner a table of every open issue still relevant: number, title, what is left, and who it waits on (code / owner's manual step / outside party), plus which issues you just closed as resolved and why.
- Ask ONLY where the issue genuinely leaves something open: product shape with two or more sound designs, a change to a security boundary or a rule in the repo's docs (e.g. a CLAUDE.md "never"), accounts/keys/money only the owner has, or work outside the repo (forks, other services). Everything an issue settles goes straight into the plan without a question.
- Multiple choice, recommended option first and labelled "(Recommended)", each option with a one-line justification and trade-off; explain the hard constraint in the question itself. At most 4 per batch.
- If the owner skips a question (answers with a message instead), ask it again in the next batch. Never fill in the recommended option as an assumed default.
- Before asking about accounts or external setup, look at what PCP can read first and ask with what you found. Before asking for a manual step, check whether PCP can do it now (SSH servers, APIs, browser); batch read-only checks into one command.
- Read free-text answers carefully: they often widen scope. Don't offer to cut scope by size; "resolve issues" means all of them.

## 4. Plan document
- Context; the decision log (owner's answers, and the designs taken straight from the issues).
- For each issue: what it is, and the architecture of the fix as a pattern (modules, data flow, reused helpers, tests). No code.
- Execution: workstreams in waves decided ONLY by dependency, never by size or priority, with hotspot files named (merged one at a time).
- Verification: the one PR CI run (see the hard rule), then the target branch's CI/CD.
- Out of scope (follow-up issues).

Design rules the owner values:
- Generalize: no special cases for one site or vendor. Defaults by class of action, with advanced settings tucked away.
- Cross-cutting concerns (rate limits, pacing, safety checks) live at the single choke point every entry path passes, so nothing bypasses them.
- Prefer folding features into existing tools over adding new ones.
- Reporting that prevents duplicate work: run ids, a check/poll tool, failure text that lists completed side effects, and "verify it didn't partly succeed before retrying".
- Shareable content carries a version number, and a format change must stay backward compatible.
- Untrusted input goes through review tiers. An agent's help is advisory, and the owner approves.
- Propose tool access per endpoint with secret-returning tools kept on "ask".

## 5. Field bug reports: investigate first
- A dedicated investigator reproduces each claim live, with logs and code, and gives a per-item verdict: verified defect / misuse or unclear description / client behaviour / not reproduced, each with evidence.
- Safety: read-only on real accounts. Click only what changes nothing; never send, accept, post or submit. Only read-only commands on hosts. Stop at any permission link and hand it to the owner.
- Fix ONLY verified causes. Unverified items and side findings go back to the owner as questions.
- Issues about outside submissions are often stale: a browser agent reads the live docs first; never submit, connect, upload or accept terms.

## 6. Execute
- Branches: the SESSION branch (the one the harness designates, e.g. claude/...) is the integration branch, restarted from the target's head when its last PR has merged. The TARGET branch is what the repo's docs say (PCP: `develop`; plekje: `dev`).
- Each workstream: one general-purpose worker in its own worktree, on branch ws/<round>-<letter>, cut from the session branch's current head. The brief names that sha ("run `git merge-base --is-ancestor <sha> HEAD`; if not, reset to it before any work").
- Size the parallelism to the machine (`nproc`, memory): about one worker per core at a time; launch the rest as slots free, in dependency order.
- Worker brief: the issue facts, the decided design, the binding repo rules, file:line pointers, required tests, contracts shared with a parallel worker, the commit trailers, WIP commits only (never `git stash`, shared across worktrees), no push, and the line "Do NOT run the full unit suite, the build or e2e; CI runs them once at the end." Workers update the specs their change affects, commit, and report the sha, a summary, the shared files they touched, migrations, and their doubts.
- Integration agent: ONE long-lived general-purpose agent owns the session branch. The coordinator hands it each finished workstream (branch + worker report); it reviews the diff against the decided design, merges with --no-ff, resolves conflicts keeping both sides' intent, runs the fast checks plus the unit tests of conflicted files, commits, pushes the session branch, removes merged worktrees, and reports anything that changes a design (which the coordinator takes to the owner). Shared repo fixes a worker discovers are merged first and handed to the others as exact commits.
- After the last merge, the integration agent: merges the latest target branch in, opens the PR into the target (description per the repo's template, issues listed), and drives its CI to green (root-cause each failure, fix within each change's intent, push; anything design-changing goes to the coordinator). Merge on green with a mergeable PR.
- Never bump versions; say in the summary when a MINOR bump is justified.

## 7. Close and confirm
- Once the PR is merged into the target branch: close each fixed issue by hand with a comment saying what changed and the merge commit (a partly fixed one stays open with a comment on what is left).
- Watch the target branch's CI/CD run for that merge to green; if it fails, the integration agent fixes and pushes (to the target per the repo's rules) until a build succeeds.
- Short status lines to the owner when waves start and finish; relay investigator verdicts and permission links; ask before overriding any decision.
- After a restart: check branches, worktrees, PRs and agent outputs before continuing; never redo merged work.
- Final summary: what each workstream did, the verdicts, the issues closed (already resolved / fixed now), what stays open and why, CI/CD status, and the bump suggestion.
