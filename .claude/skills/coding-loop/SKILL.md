---
name: coding-loop
description: Use when the user invokes /coding-loop <build prompt> or says any of these keywords: "coding loop", "code loop", "run the loop on ...", "loop this build", "build loop", "loop build", "put this through the loop", "pipeline this", "run the pipeline", "plan build review", or hands over a self-contained coding task to build end-to-end with review gates. ALWAYS use it right after a plan is approved in plan mode (ExitPlanMode), whatever the size of the plan — never implement an approved plan directly. Orchestrates the bounded plan→build→review pipeline — Opus orchestrates and gives final approval, Sonnet builds (Haiku for trivial units, Opus for hard ones), Haiku does recon and log triage, Fable adversarially reviews, a fresh-context Opus agent final-reviews — capped at 3 rounds, ending in a verified handback. Works in any repo. NOT for recurring/interval tasks (that's /loop). Never commits to shared branches, pushes, or opens PRs without explicit per-PR authorization.
---

> **Canonical skill — one source, synced into every repo.** The source of truth is
> `johnlocascio/claude-skills/coding-loop/SKILL.md`; every repo carries an identical copy at
> `.claude/skills/coding-loop/SKILL.md` so cloud sessions (which only see the clone) can run it.
> Never edit a repo copy — edit the source, then run `sync-coding-loop.sh` from claude-skills.

# /coding-loop — bounded plan → build → review pipeline

## Input

`/coding-loop <build prompt>` — a description of what to build. If the prompt is missing, ask for it; do not invent a task. For multi-PR or multi-day work, don't run this directly — wrap the units in a goal file and use `/goal` where the repo provides it (it runs this pipeline per unit with a persistent State log).

## Roles (fixed — this division of labor is the point)

| Role                   | Model                                               | Mechanics                                                                                                                                                                                                                                          |
| ---------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Orchestrator + planner | Opus 5.5                                            | Plans, writes briefs, adjudicates findings, verifies, reports, and gives final approval. Never builds. Runs as the main session — if the session is on another model, say so in the handback rather than silently substituting.                    |
| Recon                  | Haiku 5.5                                           | `Agent` / `subagent_type: Explore`, `model: "haiku"` — parallel read-only codebase sweeps during planning: locate patterns, files, tests. Escalate a sweep to `sonnet` only when it must judge, not just find.                                     |
| Builder                | Sonnet 5.5 (Haiku 5.5 if trivial, Opus 5.5 if hard) | `Agent` / `subagent_type: claude` — `model: "sonnet"` by default, `model: "haiku"` for units graded trivial, `model: "opus"` for units graded hard. Grade assigned in Phase 1 decomposition; one builder per work unit.                            |
| Adversarial reviewer   | Fable 5.1                                           | `Agent` / `subagent_type: claude`, `model: "fable"` — hunts correctness bugs, edge cases, security/RLS/multi-tenancy holes, spec infidelity, hallucinated APIs. Reviews the diff only; never the build narrative.                                  |
| Final reviewer         | Opus 5.5, fresh context                             | `Agent` / `subagent_type: Explore`, `model: "opus"` — cold read of spec + diff only, no build narrative, so it can't anchor on the builder's rationale. Fresh context is the property that matters here as much as the model.                      |
| Log triage             | Haiku 5.5                                           | `Agent` / `subagent_type: claude`, `model: "haiku"` — reads long gate, test, audit or browser-console output saved to a file and returns only the failures with file:line. Keeps raw logs out of the orchestrator's context. Reports; never fixes. |

**Model assignment (set 2026-10-07, by user instruction — adds Haiku 5.5 to the 2026-09-28 split).** Sonnet 5.5 is the workhorse builder. Opus 5.5 builds hard units and drives the loop — orchestrates, adjudicates, approves, gives the cold final read. Fable 5.1 is the adversarial reviewer, so the defect hunt is always a different model from the builder. **Haiku 5.5 does the cheap, high-volume work that needs speed, not judgment:** recon sweeps, log triage, and building units graded trivial. **Haiku never reviews, refutes, adjudicates, or runs the browser test** — every judgment in the loop stays on Sonnet or above. On a hard unit the cold final reviewer is the same model as the builder; that is acceptable because its context is clean and Fable has already supplied the cross-model pass.

**Use the aliases, not pinned model IDs.** `haiku` / `sonnet` / `opus` / `fable` resolve to the newest generation of each (today: Haiku 5.5, Sonnet 5.5, Opus 5.5, Fable 5.1) and self-update. Pinned IDs decay silently — this file once carried a stale claim that `opus` meant Opus 4.8 for months after it had moved on.

**What this trades away.** The adversarial pass used to run on Codex, a different vendor's model, deliberately: a reviewer outside the Claude family catches failure modes Claude-reviewing-Claude misses. Builder, adversarial reviewer and final reviewer are now all Claude, so that diversity is gone and the loop's remaining orthogonality comes from **a different Claude model on the adversarial pass**, **fresh context**, and **role separation**. If a build ever ships a defect all three passes missed, add a Codex pass back for that unit rather than adding a fourth Claude reviewer — more of the same family mostly agrees with itself.

**Model fallback.** Roles degrade, never halt: if a spawn is rejected for usage or limits, respawn on the session's own model rather than stopping the loop. Every substitution gets one line in the handback report so the user knows which model actually built or adjudicated.

## Entering from plan mode

When the loop starts right after a plan was approved in plan mode, that plan **is** the Phase 1 plan — the user already approved it. Do not re-plan from scratch or ask for approval again.

- Phase 0: the ambiguity check is already settled by the approval. Still do the repo-fit check, write the success criteria if the plan lacks them, and keep the high-blast-radius pause unless the approved plan already showed that design.
- Phase 1: save the approved plan as `BRIEF.md` (and start `LOOP.md`), then add only what it is missing — success criteria, per-unit grade (standard/hard), workspace setup. Still run the spec self-review; an approved plan can still prescribe a wrong API.
- This applies to **every** approved plan, however small — never implement an approved plan directly. A small plan is simply one standard-graded unit; it still gets EXERCISE and both reviews.

## Phase 0 — gates (before any planning)

1. **Repo-fit check (one beat).** Does this feature belong in THIS repo — its product, schema, personas? A contractor feature once landed in a lending platform because three review layers never asked. Mismatch → stop and ask.
2. **Ambiguity check.** Multiple reasonable interpretations → state them and ask which. Don't guess and build.
3. **Success criteria.** Write down what "done" verifiably looks like: tests that pass, commands + expected output, UI states, required breakpoints. For UI work, any mockups shown before building are interactive HTML (see **Mockups** below) and cover every breakpoint below, iPad included — a phone-and-desktop-only mockup is incomplete. Every later stage judges against this list, so vague criteria poison the whole loop.
4. **Doc routing.** Read the canonical docs the repo's CLAUDE.md / AGENTS.md maps to the touched area (schema doc for schema work, matcher doc for matcher work, etc.).
   **Repo principles.** If the repo ships a `repo-principles` skill, invoke it now. It names the repo's north-star doc (mission + principles). Plan against it, put that doc's absolute path in every builder and reviewer brief, and treat a conflict between the request and those principles as an ambiguity to raise, not a detail to absorb. Repos without one skip this.
5. **High-blast-radius pause.** Schema/migrations, auth/RLS, billing, or anything destructive: run the repo's schema-preflight skill where one exists, present the design, and get an explicit go BEFORE writing it. Everything else proceeds without asking — the user invoked the loop to build.

## Phase 1 — plan (main session)

1. Fan out Explore agents (`model: "haiku"`) in parallel for recon (existing patterns, files to touch, tests that cover the area). Keep conclusions, not file dumps.
2. Write the plan: work units, per-unit files + interfaces, test plan, invariants affected (multi-tenancy, RLS, audit, idempotency, type safety — whatever the repo's law names), risks, and the success criteria from Phase 0.
3. Create the loop's working files (see **Working files** below): save the plan as `BRIEF.md` and start `LOOP.md`. Subagents never see this conversation — every builder/reviewer prompt passes the brief's absolute path plus the exact spec section inline. A builder working from a vague prompt produces a round-1 diff that burns the whole round budget.
4. **Spec self-review — one pass over your own brief before anyone builds it.** Hunt for _prescriptions that are wrong_, not for things the brief forgot: an API used incorrectly, a constant that won't match its counterpart, a technique that doesn't do what you think it does. Where a claim is checkable — does this function exist, is it IMMUTABLE, does this attribute behave this way — check it instead of asserting it. This is the cheapest check in the pipeline and it catches a class nothing downstream is looking for: reviewers judge the diff _against_ the brief, so a defect **in** the brief is invisible to them, and a good builder will implement a wrong instruction faithfully and report success.
5. Decompose: independent units run as parallel builders; dependent units run sequentially. Set up the workspace BEFORE spawning builders: single unit → a branch per the repo's naming conventions in the main checkout; parallel units or a dirty working tree → one worktree per unit under `.claude/worktrees/<slug>/`. Never build on `main`.
6. **Grade each unit — trivial, standard or hard — and record the grade in `BRIEF.md` and `LOOP.md`.** The grade picks the builder's model; every unit gets the full review floor either way, so a misgrade costs at most a round.
   - **trivial → build `model: "haiku"`.** Purely mechanical, fully specified in the brief so there is nothing to decide: copy and text changes, config values, renames and moves, find-and-replace with an exact pattern, adding fixture or seed data, a new test that mirrors an existing one line for line. If the builder would need to read surrounding code to decide _how_, it isn't trivial. Never trivial: anything touching auth/RLS, schema, money, multi-tenancy or security, however small.
   - **standard → build `model: "sonnet"`.** The default, and it covers most real work — not just mechanical changes. Anything with a precedent in the repo to follow, however many files it touches, plus ordinary new code whose design is settled in the brief: new endpoints, components, queries, tests, refactors, integrations with a documented API.
   - **hard → build `model: "opus"`.** A unit is hard when the brief can't settle the design in advance and the builder has to reason its way to it: **no precedent in the repo**, a concurrency or ordering rule, a migration whose backward-compatibility window is non-obvious, a cross-cutting change where the parts must agree, or a surface where being subtly wrong is expensive and silent (money, auth/RLS, multi-tenancy, idempotency). "Touches the schema" alone is not hard; "touches the schema in a way nothing in this repo has done before" is.
   - **Tie-break down, then escalate.** When unsure between standard and hard, grade standard; when unsure between trivial and standard, grade standard — Sonnet 5.5 is strong enough that the review floor catches what it misses, and one wasted round is cheaper than routinely paying for the bigger builder. If a round returns blocker-level correctness findings, or the builder reports it had to invent an approach rather than follow one, regrade one step up (trivial → standard on `sonnet`, standard → hard on `opus`) and rebuild that unit. Never downgrade a unit mid-loop.

### Mockups — always interactive HTML

Whenever the loop shows a mockup — a new screen, a layout change, or any UI choice the user should see before it's built — make it an **interactive HTML file**, never a static image, ASCII sketch or prose description.

- **One self-contained file:** inline CSS and JS, no build step; external scripts only from cdnjs.cloudflare.com or cdn.jsdelivr.net. It must open by double-click.
- **Actually interactive:** every state the success criteria name is clickable — tabs, menus, modals, form input with validation errors, empty / loading / error states, hover and focus — driven by plain JS with realistic sample data, not lorem ipsum.
- **Every breakpoint:** a fixed toolbar switches the frame between 375 (phone), 768 (iPad) and desktop widths, and the layout is genuinely responsive inside it. Add a light/dark toggle if the repo supports theming.
- **Looks like the repo:** reuse the repo's design tokens, fonts, spacing and real copy where they exist, so approving the mockup means approving what will ship.
- **Variants side by side:** when there's a real design choice, put the 2–3 options in the same file behind a switcher instead of separate files.
- **Where:** `.claude/loops/<branch>/mockups/<name>.html` next to `BRIEF.md`. Show it with `SendUserFile` (`display: "render"`) so the user can click through it on any device, and open it in the browser pane to check it yourself first: no console errors, every state reachable, every breakpoint holds.
- **Approval gates the build:** wait for the user's pick or changes before building that unit. Record the decision in `LOOP.md` → Decisions, and reference the approved file in `BRIEF.md` as the visual spec — builders build to it and the cold final reviewer checks the shipped UI against it.

## Working files — `BRIEF.md` and `LOOP.md`

Every loop keeps two Markdown files, side by side, in the **main checkout** (not a worktree, which gets deleted, and not the scratchpad, which the user can't find and the next session can't see):

```
<main checkout>/.claude/loops/<branch-or-slug>/BRIEF.md
<main checkout>/.claude/loops/<branch-or-slug>/LOOP.md
```

Find the main checkout with `git rev-parse --path-format=absolute --git-common-dir` (its parent). Keep the folder out of git without touching the repo's `.gitignore`: add `.claude/loops/` to `.git/info/exclude` if it isn't there. Resuming work on a branch → read the existing files first; never start a fresh pair over them.

**`BRIEF.md` — the spec. Stable.** The plan, work units with files + interfaces and grades, test plan, invariants affected, risks, success criteria. It changes only when the user approves a scope or design change — record that change in `LOOP.md` → Decisions. **Builders and reviewers get this file's path; they never get `LOOP.md`** — it is build narrative, and a reviewer who reads it grades the narrative instead of the code. Pass a builder its round's fix-list inline instead.

**`LOOP.md` — the working tracker. Orchestrator-only; updated after every step.** Fixed sections:

```markdown
# <task> — loop tracker

Branch/worktree: … · Base: … · Status: planning | round N | verifying | done | capped-out

## To-dos

- [ ] … (work items beyond the current fix-list; tick as done)

## Security issues

| # | Severity | Issue | Where | Status (open / fixed in round N / accepted by user) |

## Open questions — decisions needed

- [blocking] … → asked the user? when?
- [non-blocking] … → current assumption

## Decisions made

- <date> — <decision> — why — decided by (user / orchestrator)

## Round log

### Round 1

- build: … · exercise: … (per breakpoint) · adversarial: findings in/resolved/carried · cold final: … · next: …

## Deferred

- <out-of-scope item> — offered for the repo's follow-up list? yes/no
```

Rules: a **blocking** question stops the affected unit and goes to the user — don't guess past it. An orchestrator decision that changes what gets built is a scope change: ask, then log it. Anything that would be lost in a compaction belongs in this file before the next step.

## Phases 2–4 — the bounded loop (max 3 rounds, no exceptions)

A **round** = one builder pass, then EXERCISE, then the reviews. Exit the moment both reviews are clean. Cap-out at 3 with blockers still open → HALT and report honestly (see Handback); never loop past the cap, never soften findings to escape it.

**BUILD (model per unit grade — Sonnet by default, Haiku if trivial, Opus if hard).** Round 1 implements the spec; rounds 2–3 work ONLY the adjudicated fix-list — no scope expansion, no drive-by refactors. Builder prompts must include: the brief path + spec section, success criteria, the working directory, "surgical changes only," the explicit-`cd`-on-every-state-mutating-Bash-call rule (shell cwd silently resets to the main checkout), and the return contract: files changed, what was done, scoped gate results. Builders run the cheap gates themselves — typecheck + lint + the tests covering touched code + the repo's formatter — before returning, so reviewers never burn a pass on syntax errors. Local commits only; never push.

**Two build-prompt requirements that exist because reviews structurally cannot cover them:**

- **Falsify every guard test.** For each new or touched test that asserts a guard, invariant, or predicate: break the thing it guards, confirm the test goes RED, restore it, confirm GREEN — and report which ones. Reviewers ask "is this code correct?"; nothing in a review asks "would this test fail if the behavior broke?" A test that cannot fail passes every review and protects nothing. Treat a can't-fail test found in review as evidence this instruction was skipped.
- **Cached gates are not evidence.** Monorepo task runners replay cached results — a green `test` can mean nothing executed, and the log may even come from a different worktree. Force the cache off (`TURBO_FORCE=true` or equivalent) and report the cached-task count as part of the gate result.

**EXERCISE — run it before reviewing it, not after.** As soon as the build returns, the orchestrator drives the change for real: browser for UI at the repo's breakpoints, live request for an endpoint, real database for a migration. This runs BEFORE the code reviews, not as a final gate, for two reasons: execution finds a class of defect that reading cannot (framework-lifecycle behavior, dev-vs-prod divergence, anything requiring a real DOM or a real connection), and reviewing code that is about to change from execution findings wastes the review. When the surface needs auth or data you don't have, a throwaway harness page that imports the real component is legitimate — mark it clearly, and revert it plus any config or middleware change before the commit.

**Browser test — required whenever the diff changes anything a browser renders.** Not optional, not delegated to a subagent, not replaced by reading the JSX. The orchestrator runs it with these mechanics:

1. **Use the built-in browser** (the app's browser pane, `mcp__Claude_Browser__*` tools). Use Claude in Chrome only if the user asks for it.
2. **Start the dev server with `preview_start` by name from `.claude/launch.json`** — never via Bash. If the file has no entry for the app, add one (it's per-machine config; don't commit it unless the repo already tracks it).
3. **At each of the repo's breakpoints** (from its CLAUDE.md / conventions doc; default 375×667 phone, 768×1024 iPad portrait, 1180×820 iPad landscape, desktop), via `resize_window`. iPad is a field device, not an afterthought: landscape usually crosses the desktop breakpoint, so check that layout works by touch (no hover-only controls, 40px targets):
   - `read_page` to confirm the content and structure the success criteria name.
   - Drive every UI state the success criteria list — click, type, `form_input`, submit — including empty, loading, error and validation states, not just the happy path.
   - `read_console_messages` (errors only) and `read_network_requests`: **zero new console errors and no unexpected 4xx/5xx.** A pre-existing error is recorded as pre-existing, not ignored and not fixed.
   - `screenshot` as evidence.
4. **Dark mode** (`resize_window` `colorScheme: "dark"`) if the repo supports theming.
5. **Reset the viewport** to `desktop` when done.
6. **Auth and data:** only test accounts from the repo's seed/fixture/example-config files, only on a local dev host. Never real credentials. Can't get in → the harness page above, not a skipped test.
7. **Findings from the browser test are findings** — they go on the next round's fix-list exactly like review findings and count against the same 3-round cap. Re-run the browser test after any round that touches the exercised surface.
8. **Can't run it** (no web UI in the change, or the surface can't be served locally) → say so explicitly in `LOOP.md` and the handback. Never report a UI change as exercised when it wasn't.

Record per breakpoint in `LOOP.md`'s round log: states driven, console/network result, screenshot taken.

**ADVERSARIAL REVIEW (Fable 5.1).** Spawn `subagent_type: claude`, `model: "fable"` with a brief to review the diff vs the base branch (explicit `cd` into the right checkout, `git diff <base>...HEAD`), hunting correctness bugs, edge cases, security/RLS/multi-tenancy holes, spec infidelity, and hallucinated APIs. Tell it to FIND DEFECTS, not to assess quality — an unprimed reviewer writes a balanced essay, and balance is not what this pass is for. Give it the spec and the diff, never the build narrative: a reviewer that reads the builder's reasoning grades the reasoning instead of the code. Findings come back to the orchestrator — never straight to the builder.

**Watch for the diff-direction artifact.** If the review reports large unrelated deletions — a test harness removed, doc entries vanishing — the branch is behind its base and the reviewer is seeing other merges backwards. Re-diff against the merge base (`git merge-base <base> HEAD`) before acting on it. An alarming finding earns MORE verification, not less; this one has burned a full round more than once.

**COLD FINAL (fresh context, Opus 5.5).** Only once the adversarial pass reports no blockers: spawn the cold reviewer with spec + success criteria + diff, nothing else. Lenses: does the diff meet every success criterion; the repo's principles doc (when `repo-principles` named one); the repo's non-negotiable invariants; the repo's conventions doc; scope discipline; breakpoints if UI changed. It must never see the builder's narrative or the orchestrator's reasoning — being the same model as the builder is acceptable only because its context is clean.

**ADJUDICATE (main session).** After each review, sort findings: real + in-scope → next round's fix-list; false positive → record why; real + out-of-scope → surface to the user and offer an entry wherever the repo keeps deferred work (its CLAUDE.md or docs index says where — e.g. one file per follow-up, or a FOLLOWUPS.md) — fixing it anyway is unauthorized scope creep. Update `LOOP.md` after every step — round log (round #, step, findings in/resolved/carried, next action), plus any to-do, security issue, question or decision the step produced — so a resumed session never redoes a converged round. Security findings go in **Security issues** as well as the fix-list; a security finding is never filed as a false positive without the reason written down.

### Scaling the review (read this before adding reviewers)

The two reviewers above are the default and they are usually enough. When the work genuinely warrants more — a security-sensitive surface, an unauthenticated route, a large greenfield module, or the user has asked for exhaustiveness — scale by **method, not topic**. This distinction is load-bearing and was learned expensively:

- **Topic lenses duplicate.** Splitting a review into "geometry" / "react-correctness" / "spec-fidelity" / "responsive" produces agents that read the same files the same way and converge on the same defects. In one measured run, eight topic lenses returned 35 confirmed findings that deduplicated to ~18 — five of them had independently found a single bug.
- **Method lenses complement.** What finds genuinely different defects is a different _way of looking_: executing the code, refuting a claim, reading spec-against-diff with no build narrative, or a different model family. Prefer at most three or four, each with a distinct method.
- **Convergence is signal, not waste, when unplanned** — several independent lenses landing on the same defect is strong evidence it is real. But do not _engineer_ redundancy to manufacture that signal; that is what the refutation panel is for.

**Refutation panel — keep it.** Route each finding to a small panel prompted to REFUTE by default, and keep only what a majority fails to refute. Across two measured PRs this absorbed roughly a third of all findings before they reached the user. It is the highest-value part of any fan-out.

**Dedupe semantically BEFORE verifying.** Deduping on `file:line` alone is not enough — the same defect gets reported at adjacent lines and survives as separate findings, then each one costs a full verification panel. Collapse by _claim_ first. In the measured run that alone would have halved the agent count with no loss.

**When a finding is contested** — the panel splits, or refutes one phrasing while confirming another — adjudicate it yourself rather than letting the vote bury it. A defect described badly is still a defect.

## Context — when to `/compact`

A multi-round loop fills the orchestrator's context fast: agent reports, diffs, gate logs, browser output. Past a point the orchestrator starts losing early detail — the success criteria, a carried finding — and adjudicates worse. Compaction is safe **only because `BRIEF.md` and `LOOP.md` hold everything that matters**; whatever lives only in the conversation is what compaction loses.

- **Compact at checkpoints, never mid-round.** Good points: right after the plan is saved (end of Phase 1); between rounds, once adjudication is written to `LOOP.md`; before Phase 5. Never between a build returning and its findings being adjudicated and logged — that is where unrecorded findings get lost.
- **When:** check at ~300k tokens of context, and again at ~600k. On machines with the `context-checkpoint` hook (claude-skills/hooks) it prompts this automatically at both points; without it (cloud sessions), the orchestrator checks itself at the same points. At 300k, recommend compacting only if it is a good checkpoint, otherwise carry on; at 600k, if it isn't a good point, get to one and then recommend it. A short single-round loop never reaches either.
- **The orchestrator can't run `/compact` itself.** At such a checkpoint, first make sure `LOOP.md` is current, then tell the user in one line: "Good point to `/compact` — state is in `<LOOP.md path>`," with a suggested focus, e.g. `/compact keep BRIEF.md and state-log paths, current round, open fix-list, success criteria`. Then continue; don't wait on it.
- **After any compaction — manual or automatic** (the harness may summarize on its own) — re-read `BRIEF.md` and `LOOP.md` before the next action. Never rebuild the plan, the fix-list, open questions or decisions from the summary.
- **Keep context lean so compaction is rarer:** take conclusions from subagents, not file dumps; write long gate and scan output to scratchpad files and have a Haiku log-triage agent return only the failing lines; don't paste whole diffs into the conversation when a reviewer agent can read them.

## Phase 5 — verification floor + handback

Before reporting done, ALL of:

- The repo's full gate suite green (e.g. `pnpm typecheck && pnpm lint && pnpm test && pnpm build`) plus formatter check — **with the task cache forced off**, and report the cached-task count so "green" means something ran
- EXERCISE (above) completed on the final diff. It runs during the round, not here; this is the confirmation that it still holds after the last fix, re-run if the last round touched the exercised surface
- Supabase repos: schema changes validated against the dev project (rolled-back DO-block probes or pgTAP), migration as a committed `.sql` file — never MCP `apply_migration`, never Studio paste. **Hold the dev push until review clears the migration** — the ledger is append-only, so a defect found before it applies costs an edit and one found after costs a permanent follow-up migration
- Both reviews clean, or cap-out declared
- `LOOP.md`: every **Security issue** is fixed or explicitly accepted by the user (never by the orchestrator), no **blocking** open question remains, and every to-do is done or moved to **Deferred**
- Every temporary artifact reverted: harness pages, config toggles, middleware bypasses, port rebases, CLI scratch files. `git status` shows only what you intend to commit, and pre-existing dirt you did not create is still there and still uncommitted

**Handback report:** links to `BRIEF.md` and `LOOP.md`, what was built, files changed (as clickable paths), rounds used, per-unit grade + builder model (plus any escalations or model substitutions), findings resolved / carried / deferred, security issues and how each was closed, decisions made, any open non-blocking questions, verification evidence (gate output, plus the browser-test result per breakpoint with screenshots sent via `SendUserFile` for any UI change), and where the work sits (branch/worktree, local commits). On cap-out: the open blockers and the diff state, plainly — a capped-out build is a finding, not a failure to hide.

**Then STOP.** The loop never pushes, opens a PR, marks ready, or merges — each of those needs the user's explicit per-PR authorization, and one approval never carries to the next. Review approvals are not merge authorizations.
