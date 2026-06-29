# AGENTS.md — Shared rules for all agents

This file is the single source of truth for how agents work in this repository.
Every harness (Claude Code, Codex, Qwen Code, DeepSeek, OpenCode, etc.) reads it.
Keep it lean. If a rule changes, change it here — not in per-tool config.

## Sources of truth (never duplicate state)

- **Backlog & task status → GitHub Issues.** Authoritative. Do not maintain a second
  task list anywhere.
- **Product intent → `specs/<version>/spec.md` and `plan.md`.** Code must conform to these.
- **UI/UX intent (design track) → `specs/<version>/design.md`.** The prototype and any
  shipped UI must conform to it.
- **Execution / board tool (e.g. Multica) → mirrors GitHub Issues and runs the work.**
  It does NOT own the backlog. If the board and a GitHub Issue disagree, the **Issue wins**.

## Roles

- **Commander (plan + review):** the strong, low-frequency models (Codex / Claude).
  Used for writing & clarifying specs, design briefs, planning, decomposition, and PR
  review. Use sparingly.
- **Executor (implementation):** the high-frequency, metered models (Qwen / DeepSeek).
  Used for writing code, prototypes, tests, and docs against a single task.
- **Reviewer ≠ author.** A PR MUST be reviewed by a different model than the one that wrote it
  (Codex-written → Claude reviews; Claude-written → Codex reviews). Never self-review.

## Before touching code, every task must load

1. The relevant `specs/<version>/spec.md` and `plan.md` (plus `design.md` on the design track).
2. Its GitHub Issue (scope + acceptance criteria).

Read only what the task needs. Do not pull the whole repo into context.

## Task discipline

- One task per session/branch. Keep tasks small (≈0.5–2h; smaller is better).
- Branch from `main`, named `issue-<n>-<slug>`.
- Use TDD where practical: write/extend tests first, then make them pass.
- Before claiming a task done, **all** of these must pass locally: lint, typecheck, tests, build.
  If you cannot make them pass, **stop and report a blocker on the Issue** — do not mark it done.

## Design & prototype track (optional — Scenario 2/3)

Some projects need a design/prototype phase before full code. It reuses this same skeleton;
only the deliverables and one extra gate change.

- **Design intent → `specs/<version>/design.md`:** user flows, information architecture,
  visual direction, component inventory, responsive breakpoints, accessibility target, and a
  per-screen acceptance checklist. Authoritative for *what the UI must be*.
- **Visual source of truth = the prototype app OR a linked Figma file — never both.** In an
  agent pipeline the prototype is usually code (HTML / components under `prototype/`), because
  models emit code, not Figma. If a human uses Figma, link it from `design.md`; do not
  duplicate its state into the repo.
- **Design tokens** (colour / spacing / type) live in one place (`design/tokens/` or the
  prototype's token module). No magic values scattered across components.
- **Every prototype PR must produce a preview URL that opens on a phone** (see
  `.github/workflows/preview.yml`). Reviewers look at the live preview, not screenshots.
- Roles are unchanged: Commander writes `design.md`; an executor produces the prototype;
  reviewer ≠ author still holds.

## Pull requests

Open one PR per task. The description MUST contain:

- Linked Issue (`Closes #<n>`)
- What was implemented
- Test results
- Anything not done / known gaps
- Risks or assumptions

Never push directly to `main`. `main` requires a PR with green CI and human approval.

## Gates (cheapest first)

1. **Spec gate (human):** spec/plan is reviewed *before* any code or design. Cheapest place to
   catch errors — take it seriously.
2. **Design gate (human — design track only):** review `design.md` + the live preview *before*
   writing production code. Same logic as the spec gate: catch UX errors when they are cheapest
   to fix.
3. **CI gate (automated):** lint / typecheck / test / build must be green.
4. **Review gate:** independent agent review (reviewer ≠ author), then human approval on the PR.

## Honesty

- Do not claim success you have not verified. If tests fail or you are unsure, say so on the
  Issue and stop. A reported blocker is good; a silent wrong "done" is the worst outcome.
- Do not invent facts, file contents, or API behavior. If you need ground truth (real data,
  a real library API), fetch or ask for it — never fabricate.

## Scratch files

- `.superpowers/sdd/` is git-ignored working-tree scratch (task briefs, reports, progress
  ledger). Never commit it. **Never run `git clean -fdx`** — it deletes the progress ledger.
