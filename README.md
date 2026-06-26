# sdd-agent-starter

Template for spec-driven, multi-agent software projects.
Click **"Use this template"** to start a new project — the conventions below come pre-wired.

## Stack this assumes

- **Commander (plan + review):** Codex / Claude (low-frequency).
- **Executors (implementation):** Qwen / DeepSeek (high-frequency, metered).
- **Methodology:** Spec-Driven Development (e.g. Superpowers' `brainstorming` +
  `writing-plans` for the front half).
- **Orchestration:** a board/runtime (e.g. Multica) that mirrors GitHub Issues and
  dispatches work to executors.
- **State:** GitHub — repo for artifacts, **Issues for the backlog (single source of
  truth)**, CI as the automated gate.

All agent rules live in [`AGENTS.md`](./AGENTS.md). Read that first.

## Layout

```
specs/                      # spec.md / plan.md / tasks.md, per version (e.g. specs/v1/)
.github/workflows/ci.yml    # minimal CI (Node/TS + Python); green on an empty repo
AGENTS.md                   # single source of truth for agent behavior
.gitignore
README.md
```

## Flow (per project)

1. Commander writes `specs/v1/spec.md` (SDD) → **you review the spec** (cheapest gate).
2. Commander writes `plan.md`, then decomposes into small tasks → GitHub Issues.
3. Orchestrator dispatches each Issue to an executor; the executor reads the spec + Issue,
   works on a branch, and opens a PR.
4. CI runs (lint / typecheck / test / build). An independent agent reviews (reviewer ≠ author).
   You approve the PR.
5. Merge → close Issue → next task.

Scenario 2/3 (design, prototype, full code) reuse this exact skeleton — only the *content*
of the tasks changes, plus a preview deploy so prototypes are viewable from your phone.

## One-time setup checklist

- [ ] Protect `main`: require a PR + passing CI before merge (Settings → Rules).
- [ ] Confirm CI is green on the empty scaffold.
- [ ] Point your orchestrator at this repo; set **GitHub Issues** as the backlog source of truth.
- [ ] Keep any `CLAUDE.md` lean — shared rules belong in `AGENTS.md`.
