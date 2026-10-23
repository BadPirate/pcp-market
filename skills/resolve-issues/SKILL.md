---
name: resolve-issues
description: Use when the owner asks to "resolve all open issues" (or a set of them) in a repo. Plan first with multiple-choice questions, then run the work with subagents and one combined test pass before anything reaches a shared branch.
version: 2
pcp_format: 2
---

# Resolve issues (plan, then run multi-agent work over a repo's open issues)

Works for any repo. The owner wants a reviewed plan first, multiple-choice questions where the scope is unclear, and execution by subagents with one combined pre-check before anything reaches a shared branch.

## HARD RULE: the full suite runs exactly ONCE
The full suite (full unit run, build, e2e) runs ONE time per run of this skill: after the LAST workstream has merged into the session branch, immediately before merging into the target branch. Never between waves, never after each merge, never "to be safe", never by a workstream worker, never by the coordinator while waves remain. Between waves the most anyone runs is the fast pre-commit checks (lint, typecheck, format, plus the unit tests next to a change), and the coordinator only runs them after resolving a merge conflict. Work the owner adds mid-run becomes more workstreams merged first, and the single full run moves later; it is not run twice. If you notice you are about to run the full suite and workstreams are still open, stop: that run is wasted and will be repeated.

## 1. Gather (read-only)
- List open issues WITH comments: addenda in comments often add whole items. Read any memories or docs an issue references.
- Read the repo's agent and contributor docs (CLAUDE.md, CONTRIBUTING, ARCHITECTURE, DESIGN). Their boundaries bind every design.
- Note the designated dev branch, the release rules ("never bump versions unless asked") and the check commands.

## 2. Explore
- At most 3 read-only explore agents in parallel, each with a distinct area (e.g. gateway/API surface; runtime/browser/execution; storage/permissions/UI patterns).
- Ask for facts with file:line references, existing helpers to reuse, the choke points every call passes through, and the constraints that block designs (no background key, no child processes, stateless transport...).
- Tell the user in one line what is running.

## 3. Ask before designing (AskUserQuestion)
- FIRST, before any question: show the owner a summary of every open issue that is still relevant (not closed, not already done): number, title, what is left, and who it waits on (code / owner's manual step / outside party), with what the live checks found. Only then start the question batches.
- For every issue whose scope is unclear or that has two or more sound architectures: multiple choice, recommended option first and labelled "(Recommended)", each option with a one-line justification and trade-off. At most 4 questions per batch; run several batches.
- Explain the hard constraints in the question itself (e.g. "PCP can't decrypt outside a request").
- If the owner skips a question (answers with a message instead), ASK IT AGAIN in the next batch. Never fill in the recommended option as an assumed default.
- Before asking about accounts or external setup (Stripe, hosting, DNS...), look at what PCP can read first (search for the service) and ask with what you found, plus a recommendation.
- Before asking for a manual step, check whether PCP can do it now (SSH servers, Coolify APIs, browser). Ops issues often close with live checks plus one approval each. Batch read-only SSH checks into one command, because every command asks.
- Read free-text answers carefully: they often widen scope (new UI tabs, trust levels, sync rules). Ask follow-ups about what they leave open.
- Don't offer to cut scope by size ("only the foundations", "skip the big one"). "Resolve all open issues" means all of them; how big an issue is never decides whether it is in or which wave it lands in. Ask only about what is genuinely undecided (product shape, open questions in the issue, accounts/keys only the owner has).
- Before finishing the plan, ask whether anything about the scope is unclear. If the owner rejects a plan with notes, fold the notes in, ask about new ambiguities, and re-present.

## 4. Plan document
- Context; then the decisions made (decision log).
- For each issue: what the issue is, and the architecture of the fix as a pattern (modules, data flow, reused helpers, tests). No code.
- Execution: workstreams in waves decided ONLY by dependency (a workstream goes in the earliest wave whose prerequisites have merged), never by size or priority, with hotspot files named (merge those one at a time).
- Verification: name the ONE full-suite run, after the final merge and just before the target merge (see the hard rule). The plan never schedules a full run between waves.
- What is out of scope (follow-up issues).

Design rules the owner values:
- Generalize: no special cases for one site or vendor. Defaults by class of action, with advanced settings tucked away.
- Cross-cutting concerns (rate limits, pacing, safety checks) live at the single choke point every entry path passes (direct call, scripts, wrappers, owner-approved calls), so nothing bypasses them.
- Prefer folding features into existing tools over adding new ones (e.g. one search plus one describe across tools, skills and memories).
- Reporting that prevents duplicate work: run ids, a check/poll tool, failure text that lists completed side effects, and a clear "verify it didn't partly succeed before retrying".
- Shareable content carries a version number, and a format change must stay backward compatible.
- Untrusted input (repos, schemas, documents) goes through review tiers. An agent's help is advisory, and the owner approves.
- Propose tool access per endpoint with secret-returning tools (env values, private keys, tokens) kept on "ask".

## 5. Field bug reports: investigate first
- A dedicated investigator reproduces each claim live, with logs (e.g. through a PCP server's command tool) and code, and gives a per-item verdict: verified defect / misuse or unclear description / client behaviour / not reproduced, each with evidence.
- Safety: read-only on real accounts. Click only what changes nothing; never send, accept, post or submit. Run only read-only commands on hosts.
- Stop at any permission link and hand it to the owner.
- Fix ONLY verified causes. Bring unverified items, and side findings with fix options, back to the owner as questions. No exploratory guess fixes without approval.
- Issues about outside submissions (OpenAI, Meta, Google) are often stale. Have a browser agent read the live docs first. Never submit, connect, upload or accept terms; stop at each login with hand_over. Hand-over links expire, so re-issue them when the owner asks.

## 6. Execute
- Branches: the SESSION branch (the one the harness designates, e.g. claude/...) is the integration branch: workstreams merge into it. The TARGET branch is what the repo's docs say finished work lands on (e.g. main). Don't ask which; this is settled. EXCEPTION: if the repo's docs say otherwise, follow them (e.g. plekje's CLAUDE.md: work lands on `dev` directly, and anything needing e2e that can't run here goes as a PR into `dev`, watched to green, then merged).
- Each workstream gets one general-purpose agent with worktree isolation, on branch ws/<letter>, cut from the session branch's current state so it sees earlier merges.
- Worktree isolation can cut a worker's worktree from an OLDER commit than the session branch. Every brief names the session-branch sha it needs ("first run `git merge-base --is-ancestor <sha> HEAD`; if it isn't there, `git reset --hard <session branch>` before any work"), and the coordinator checks each worktree's base after launch.
- Shared repo fixes a worker discovers (lint config, hooks) get merged first and handed to the others as exact commits to apply, so the same fix isn't written N different ways.
- Worker brief = the issue facts, the decided design, the binding repo rules, file:line pointers, required tests, any contract shared with a parallel worker (e.g. "add optional {runId, signal} to callServerTool options; the other worker passes them"), the commit trailers, and the explicit line "Do NOT run the full unit suite, the build or e2e; that runs once at the end."
- Workers stay lean: the pre-commit checks (lint, typecheck, format) plus only the unit tests next to their change. No full suite, no build, no e2e. They update the specs their change affects, commit, don't push, and report the sha, a summary and their doubts.
- Coordinator: review each diff (skim the key files), merge with --no-ff into the session branch, and resolve conflicts. Run independent workstreams in parallel; dependent ones in later waves. Between waves the coordinator runs NO suite, NO build and NO e2e, and spawns no test worker: at most the fast pre-commit checks after a conflict resolution. Merge fallout is fixed once, in the single final run, not every wave.
- Remove worker worktrees after merging: lint at the repo root picks up their files (and builds).
- The single full run (see the hard rule): only once every workstream, including any the owner added mid-run, has merged into the session branch, one integration worker runs the FULL suite there (lint, typecheck, format, unit, build, all e2e) and fixes what fails within each change's intent; anything that would change a design goes back to the owner. Prepare its environment (databases, browsers, containers) while the waves run, but start the run itself only after the last merge. Then merge the latest target branch in, merge into the target branch, and push both.
- Never bump versions; say in the summary when a MINOR bump is justified.

## 7. Keep the owner in the loop
- Short status lines when waves start and finish. Relay investigator verdicts and any permission links. Ask before overriding any decision.
- After a restart: check branches, worktrees and agent outputs before continuing; never redo merged work.
- Final summary: what each workstream did, the verdicts, which issues will close on release, and the bump suggestion.
