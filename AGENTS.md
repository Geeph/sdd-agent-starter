# AGENTS.md — Shared rules for all agents

This file is the single source of truth for how agents work in this repository.
Every harness reads it — directly (Codex, OpenCode), via a thin pointer file
(`CLAUDE.md`, `QWEN.md`, `GEMINI.md`, `.github/copilot-instructions.md`, `.cursorrules`), or
fed by the orchestrator for models without a context-file convention (e.g. DeepSeek).
Keep it lean. If a rule changes, change it here — not in per-tool config.

## Sources of truth (never duplicate state)

- **Backlog & task status → GitHub Issues.** Authoritative. Do not maintain a second task list
  anywhere (`plan.md` / task lists are planning snapshots, not status).
- **Product intent → `specs/<version>/spec.md` and `plan.md`.** Code must conform to these.
- **UI/UX intent (design track) → `specs/<version>/design.md`** plus its visual source (a
  linked Figma file or a code prototype). Shipped UI must conform to it.
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

Writing `spec.md` / `design.md` (including linking a Figma file) is **planning, not code** — the
Commander does it up front, so the design phase does not need a plan/Issue to exist yet.
Everything that is **code** — including a code prototype — obeys the rule above: its Issue +
plan come first.

Read only what the task needs. Do not pull the whole repo into context.

## Task discipline

- One task per session/branch. Keep tasks small (≈0.5–2h; smaller is better).
- Branch from `main`, named `issue-<n>-<slug>`.
- Use TDD where practical: write/extend tests first, then make them pass.
- Before claiming a task done, **all** of these must pass locally: lint, typecheck, tests, build.
  If you cannot make them pass, **stop and report a blocker on the Issue** — do not mark it done.

## Design track (optional — Scenario 2/3)

Some projects need a design phase before full code. It reuses this skeleton; only the
deliverables and one extra gate change.

- **Design intent → `specs/<version>/design.md`:** user flows, information architecture,
  visual direction, component inventory, responsive breakpoints, accessibility target, and a
  per-screen acceptance checklist. Authoritative for *what the UI must be*. Written by the
  Commander — a planning artifact, not code.
- **Visual source of truth = a linked Figma file OR a code prototype — never both:**
  - **Figma** (human designer): link it from `design.md`. The "preview" is the Figma share
    link itself — open it on a phone. No deploy or CI needed for the design gate.
  - **Code prototype** (agent-generated UI): build it under `prototype/`. Because it is code,
    it is a normal tracked task — its own Issue + light plan first. Its preview is a deployed
    URL. The built-in GitHub Pages workflow deploys after the prototype PR merges to `main`;
    use Vercel/Cloudflare only when the design gate must happen before that merge.
- **Design tokens** (colour / spacing / type) live in one place (`design/tokens/` or the
  prototype's token module). No magic values scattered across components.
- Roles unchanged: Commander writes `design.md`; an executor builds any code prototype;
  reviewer ≠ author still holds.

## Pull requests

Open one PR per task, using the PR template (`.github/pull_request_template.md`). It MUST contain:

- Linked Issue (`Closes #<n>`) — validated by the required `pr-hygiene` check
- What was implemented
- Test results
- Anything not done / known gaps
- Risks or assumptions
- **Author model** and **Reviewer model** (reviewer ≠ author)

Never push directly to `main`. `main` requires a PR with green CI and human approval.

## Gates (cheapest first)

1. **Spec gate (human):** review `spec.md` before design starts, and review the relevant
   `plan.md` before code starts. Cheapest place to catch errors — take it seriously.
2. **Design gate (human — design track only):** review `design.md` + the Figma link (or the code
   prototype's live preview) *before* writing production code. Same logic as the spec gate.
3. **CI gate (automated):** lint / typecheck / test / build must be green. CI skips a stack that
   is absent; once present, the root app must pass all four. A code `prototype/` only needs to
   build (it is throwaway for the design gate).
4. **Review gate:** independent agent review (reviewer ≠ author), then human approval on the PR.

## Honesty

- Do not claim success you have not verified. If tests fail or you are unsure, say so on the
  Issue and stop. A reported blocker is good; a silent wrong "done" is the worst outcome.
- Do not invent facts, file contents, or API behavior. If you need ground truth (real data,
  a real library API), fetch or ask for it — never fabricate.

## Scratch files

- `.superpowers/sdd/` is git-ignored working-tree scratch (task briefs, reports, progress
  ledger). Never commit it. **Never run `git clean -fdx`** — it deletes the progress ledger.
