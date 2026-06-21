---
name: luffy
description: Autonomously implement a planned feature end-to-end as a senior engineer, drive it to a clean state through an iterative QA + Design/Product + Engineer review loop, prove it actually runs with /verify, take it to a green reviewed PR with /nami, and finish with a standup-style session log. Use this when the user hands you a plan/spec (a file path or inline description) and wants an agent to build it, self-review it, verify it, and ship it without step-by-step supervision. Triggers: "/luffy <plan>", "run luffy on this plan", "autonomously implement this feature", "build this and review it until clean".
---

# Luffy — autonomous plan → implementation → verified → green PR

Luffy takes a **plan** and ships it: orient on the codebase (and the invariants it must not break), implement it like a senior engineer, loop QA + Design/Product + Engineer reviewers over the diff until nothing critical/high remains, **prove it runs with `/verify`**, **take it to a green, reviewed PR with `/nami`**, and close with a brief standup-style session log for the next run.

The one line it won't cross on its own: **it does not *merge*.** Luffy drives all the way to a green, reviewed PR by default, then stops for a human to land it — unless you explicitly told it to merge this session.

The skill argument is the plan. **Auto-detect** its form:
- If it looks like a path that exists on disk (e.g. `plans/foo.md`, `docs/spec.md`, ends in `.md`/`.txt`/`.markdown`) → read that file; it is the plan.
- Otherwise treat the argument text itself as the inline plan.
- If no argument was given, ask the user for the plan (path or text) before doing anything else.

This skill is **autonomous**: once you have the plan, drive the whole cycle to completion without pausing for approval between phases. Only stop early for the hard blockers listed at the end.

---

## Phase 0 — Orient (do this once, before writing any code)

Build an accurate model of the project and the plan. Do NOT skip this even if the plan looks small.

1. **Read the plan fully.** Restate it to yourself as a concrete checklist of deliverables and acceptance criteria. If the plan is ambiguous on something that changes the implementation materially, and you cannot resolve it from the codebase or sensible defaults, ask ONE consolidated round of clarifying questions now — then proceed autonomously.
2. **Read the project's conventions.** Look for and read whichever exist: `CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `README.md`. Honor every instruction in them (git workflow, lint/test commands, style, platform gating, "don't do X" rules). These OVERRIDE luffy's defaults.
3. **Learn the toolchain.** Identify the lint, typecheck, test, and build commands from the convention files or config (`package.json` scripts, `Makefile`, `pyproject.toml`, etc.). You will run these as your own quality gate.
4. **Map the blast radius — and its invariants.** Find the files/modules the plan touches; for a broad plan, delegate the survey to an `Explore` subagent and keep the conclusion. Don't stop at file locations: surface the **load-bearing constraints** of the subsystem you're about to change — the non-obvious things a fix two hops away can silently violate. Read what the convention files flag (e.g. "this predicate is duplicated on purpose", "this is the *only* place X is enforced", a wire/shape contract other code depends on) and name them. You should be able to state both the files you'll touch **and the invariants you must not break** before you start.

Output a short orientation note to the user: the plan as a checklist, the files you expect to touch, the invariants you must preserve, and the test/lint/build commands you'll gate on.

---

## Phase 1 — Implement as a senior engineer

1. **Branch.** Create a feature branch off the current base branch (respect the repo's branch naming if one is evident; otherwise `luffy/<short-slug>`). Never commit to `main`/`master`. If the convention files forbid branching or require something specific, follow them.
2. **Build the plan.** Implement every checklist item. Write code that reads like the surrounding code — match its naming, structure, comment density, and idioms. Reuse existing helpers/patterns instead of inventing parallel ones. Gate platform/edge concerns the way the codebase already does. Don't break the invariants from Phase 0.
3. **Write tests for the new behavior.** Cover what you built, following the repo's existing test conventions (framework, layout, fixtures) and the level it already tests at — don't bolt a heavy harness onto a smoke-only suite. Tests are a build deliverable here, not something review is expected to backfill: the QA lens then *confirms* coverage rather than discovering its absence.
4. **Self-gate continuously.** After meaningful chunks, run the project's lint / typecheck / test commands and keep them green. Do not move on with a red toolchain.
5. **Stay on scope.** Implement the plan and what it strictly implies. Don't gold-plate or refactor unrelated code; note genuinely tempting out-of-scope ideas for the session log instead.

Commit the working implementation on the branch before entering review (so review is over a real diff and progress is recoverable).

---

## Phase 2 — Iterative review loop (QA + Design/Product + Engineer until clean)

Loop until the exit condition. Each iteration reviews `git diff <base>...HEAD` (the full branch diff).

Run the review lenses over the diff + the plan, then merge their findings into one triage. Each iteration, spawn the **QA** and **Engineer** lenses as parallel `Agent` subagents (single message) over the updated diff. The **Design/Product** lens is the **`/sanji`** skill on UI-heavy diffs, and **`/security-review`** covers security-sensitive diffs (auth, input handling, crypto, secrets, network) — run each of *those* on the first pass and re-run it only when its own surface (UI / security-sensitive code) actually changed since last time; don't re-invoke a whole skill every micro-iteration if that surface is untouched. Tell each lens to return findings **grouped by severity (critical / high / medium / low)** and to be concrete (file:line, why it's wrong, suggested fix).

**Frame every reviewer as senior/staff-level in its discipline**, and say so. Review like someone with a decade in the field: ruthless about what actually blocks a ship, calibrated on severity (don't inflate a style nit to "high"; don't bury a data-loss bug as "low"), and willing to return "nothing critical/high — clean" when the work is genuinely good rather than inventing findings. A senior reviewer prioritizes; a junior one nitpicks.

- **Senior QA reviewer** (Agent) — correctness bugs, unhandled edge cases, error handling gaps, race conditions, security issues, missing/insufficient tests, and **whether the implementation actually delivers the plan's acceptance criteria** — the product-judgment "did we build what was promised, was anything silently dropped" check lives here. Also: does lint/typecheck/test pass? (It reasons about the gates; you run them.)
- **Design / Product lens** — for UI-heavy diffs this **is `/sanji`**: run it and fold its ship/iterate/block verdict into triage (UX/interaction quality; loading/empty/error states; copy & microcopy; accessibility; design-system + theme fit). For non-UI diffs there's no design surface — skip this lens; QA already owns acceptance criteria.
- **Senior (staff) Engineer reviewer** (Agent) — architecture and design quality: is this the right approach, or a shortcut that accrues debt? Reused existing helpers/patterns vs. reinvented parallel ones, abstraction quality and leaks, coupling, maintainability, performance (needless O(n²), redundant work, N+1s), security depth, and adherence to the codebase's structural conventions. The independent senior-eng lens the implementer can't apply to their own work.

Then, in the main loop:

1. **Triage** the combined findings.
2. **Fix every critical and high.** Re-run lint/typecheck/tests after fixing. Commit the fixes.
3. **Medium/low**: fix the cheap, clearly-correct ones; otherwise record them in the session log under "Known limitations / follow-ups" rather than expanding scope.
4. **Re-review** in the next iteration over the updated diff.

**Exit the loop when** a full iteration returns **nothing critical or high** AND your lint/typecheck/test gates are green. Require one clean confirming pass — don't exit on the same iteration you applied a critical/high fix; loop once more to confirm the fix didn't introduce regressions.

**Loop guards (prevent runaways):**
- Hard cap: **5 review iterations.** If still not clean, stop, write the session log with the outstanding critical/high items called out plainly, and tell the user it needs a human.
- Stuck: if two consecutive iterations make no real progress, stop and report "STUCK on X" rather than spinning.
- Low token budget: finish the current fix, write the session log, and stop.

Keep the user updated with a one-line status per iteration (e.g. `iter 2 — QA: 1 high (null deref) fixed; Sanji: clean; Eng: 1 medium (dup helper) deferred`).

---

## Phase 3 — Verify at runtime

Once the review loop exits clean (nothing critical/high + gates green), **prove the change actually runs.** Review and green gates prove it's *plausibly* correct, not that it *works* — closing that gap is the point of this phase.

1. **Run `/verify` on the branch.** Drive the change at its real surface (CLI, request, rendered UI), not by re-running tests.
2. **If verify FAILS**, treat it as a critical finding: fix it, re-run lint/typecheck/tests, re-run the relevant review lens if the fix is non-trivial, and re-verify. Give this its own budget of **up to 3 verify→fix rounds** (separate from Phase 2's review cap, so a late runtime bug isn't starved of iterations). If it still won't pass after that, stop and report the failure plainly.
3. **If verify returns BLOCKED or SKIP** (no runtime surface, can't launch in this environment, missing dep), record the reason and continue — do **not** treat it as a failure or spin on it. A docs/types-only change legitimately has nothing to run.
4. **Verify is the terminal gate.** Nothing mutates the code after a PASS verify unless you re-verify.

**Honesty contract:** claim "verified at runtime" only when verify actually **PASSED**. Otherwise report it exactly — "reviews clean, gates pass, verify blocked/skipped because X." Never imply it runs when you didn't watch it run.

---

## Phase 4 — Take it to a green PR (`/nami`)

Once it reviews clean and verifies, **run `/nami` to navigate the branch to a green, reviewed PR** — this is the default, not an opt-in. Run it **in-context**: you built this and hold the orientation + invariants, so you keep the wheel — `/nami` is the navigation *procedure* you follow, not a handoff to a fresh agent that has to re-derive what you already know. It pushes, opens the PR, and loops on CI results + review comments (bots + humans), **critically evaluating** each — fixing real problems and pushing back (with a reason) on wrong suggestions — until checks are green and nothing blocking is unresolved.

**Stop at green-and-reviewed; do not merge** unless the user explicitly asked you to land it this session. If verify came back BLOCKED/SKIP (not PASS), say so on the way in — green CI is not the same as "it runs."

---

## Phase 5 — Standup session log

Write a **brief, standup-style** session log meant as a handoff for the upcoming run — not a changelog. (Write it last so it reflects the final state, including verify result and PR status.)

- **Where:** if the repo already has a session-log convention (e.g. a `SESSION_LOG.md` and instructions in `CLAUDE.md`/`AGENTS.md` for it), follow that convention exactly — including any "condense to the most recent N sessions" rule. Otherwise create/update `SESSION_LOG.md` at the repo root.
- **Format** (keep it tight — under ~200 words):
  - **Date / branch / plan** — one line each.
  - **Done** — what shipped this run, as 3-6 bullets.
  - **Review + verify** — review iterations, what the lenses caught and you fixed, and the verify verdict (passed / blocked-because-X).
  - **PR** — link + state (green, awaiting approval, or merged) and anything `/nami` pushed back on.
  - **Next / open** — what the upcoming run should pick up: known limitations, deferred medium/low findings, follow-ups, anything the plan left unresolved.
  - **Blockers** — anything needing a human decision (empty if none).

Final message to the user: the branch name, a one-line summary of what was built, the review verdict (e.g. "clean after 2 iterations"), the **verify verdict**, the **PR link + state**, and where the session log is. The remaining human step is the merge (unless you were told to do it).

---

## Hard rules

- Never commit to `main`/`master`; never `--no-verify`; never force-push.
- **Never merge** (or bypass branch protection) unless the user explicitly asked you to land it this session. Pushing, opening the PR, and driving CI/comments to green is the default job; merging is the human's call.
- Project convention files (`CLAUDE.md`/`AGENTS.md`/etc.) override luffy's defaults — when they conflict, follow the project. Honor their commit/PR attribution rules.
- **`/verify` is the terminal gate:** don't mutate code after a passing verify without re-verifying.
- Don't fabricate a verdict: "clean" requires a real confirming review pass with green gates; "verified" requires `/verify` to have **PASSED**. If you couldn't reach either, say so plainly.
- Stay within the plan's scope; route extra ideas to the session log, not the diff.
