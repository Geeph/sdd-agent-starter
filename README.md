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

All agent rules live in [`AGENTS.md`](./AGENTS.md). Read that first. Each harness gets a thin
pointer file (`CLAUDE.md`, `QWEN.md`, `GEMINI.md`, `.github/copilot-instructions.md`,
`.cursorrules`) so it loads the same rules; Codex and OpenCode read `AGENTS.md` natively.

## Layout

```
specs/
  _template/                # copy into specs/<version>/ to start a version
    spec.md                 #   what & why
    design.md               #   UI/UX intent (design track): a Figma link OR a code prototype
    plan.md                 #   how it gets built -> Issues (status lives in the Issues)
  v1/ …                     # one folder per version
prototype/                  # OPTIONAL code prototype (only if you don't design in Figma)
design/tokens/              # design tokens (colour/spacing/type) — optional
.github/
  pull_request_template.md  # required PR description (linked Issue, sections, author/reviewer)
  ISSUE_TEMPLATE/task.yml   # task issue form (scope + acceptance + track)
  copilot-instructions.md   # -> AGENTS.md
  workflows/
    ci.yml                  # lint/typecheck/test/build (Node/TS + Python); green on an empty repo
    pr-hygiene.yml          # validates required PR sections, Issue link, and reviewer separation
    preview.yml             # OPTIONAL code-prototype preview (Figma needs no deploy)
AGENTS.md                   # single source of truth for agent behavior
CLAUDE.md / QWEN.md / GEMINI.md / .cursorrules   # thin pointers -> AGENTS.md
.gitignore
README.md
```

## Two flows

Both flows share the same spine — `main` is protected, and every change is a small task →
GitHub Issue → branch (`issue-<n>-<slug>`) → PR → CI → independent review (reviewer ≠ author)
→ human approval → merge. **GitHub Issues are always the backlog source of truth.** The only
difference is whether a **design phase** sits in the middle.

### Flow A — without Design (spec → code)

```
spec.md ─▶ [SPEC GATE: you review] ─▶ plan.md ─▶ GitHub Issues
   ─▶ executor implements on issue-<n>-<slug> ─▶ PR
   ─▶ [CI] ─▶ [REVIEW: reviewer ≠ author] ─▶ [you approve] ─▶ merge
```

1. Commander writes `specs/v1/spec.md` → **you review the spec** (cheapest gate).
2. Commander writes `plan.md`, decomposes into small tasks → GitHub Issues.
3. Orchestrator dispatches each Issue to an executor; it reads the spec + Issue, works on a
   branch, opens a PR.
4. CI runs (lint / typecheck / test / build). An independent agent reviews. You approve.
5. Merge → close Issue → next task.

Use this for back-end, library, CLI, or any work with no UI to settle first.

### Flow B — with Design (spec → design → code)

`design.md` captures the UI intent; its **visual source** is a **Figma link** (default) or a
**code prototype** under `prototype/` — never both. The design gate reviews whichever it is:
for Figma you just open the share link on your phone; for a code prototype, its live preview.
No production code is written until the design is approved.

```
Figma:     spec ─▶ SPEC GATE ─▶ design.md + Figma ─▶ DESIGN GATE
Code UI:   spec ─▶ SPEC GATE ─▶ design.md ─▶ prototype plan + Issue ─▶ prototype PR
                    ─▶ merge ─▶ Pages URL ─▶ DESIGN GATE

After either gate: production plan ─▶ Issues ─▶ implement ─▶ PR ─▶ CI ─▶ review ─▶ merge
```

1. Commander writes `specs/v1/spec.md` → **you review** (spec gate).
2. Commander writes `specs/v1/design.md` and links the **Figma** file. For agent-generated UI
   instead, first create a light prototype plan + Issue, then an executor builds it under
   `prototype/` and opens a normal PR.
3. **Design gate:** open the Figma link on your phone. For a code prototype, merge its reviewed
   PR so the built-in Pages workflow can publish the shared URL, then review that URL. If the
   gate must precede the prototype merge, configure a per-PR provider such as Vercel/Cloudflare.
   In both cases the design gate happens *before* production code.
4. Once approved, continue exactly like Flow A: `plan.md` → Issues → implement → PR → CI →
   review → merge. The approved design is the reference.

Use this whenever there's UI to settle before committing to full code.

> Design intent (`design.md`, a Figma link) is **planning, not code** — the Commander writes it
> up front, so the design phase doesn't deadlock against the "load plan + Issue before code"
> rule. Only the implementation (and any code prototype) is code, and it follows that rule.

## One-time setup checklist

- [ ] Protect `main`: require a PR + passing CI before merge (Settings → Rules / ruleset).
- [ ] Confirm CI is green on the empty scaffold.
- [ ] Point your orchestrator at this repo; set **GitHub Issues** as the backlog source of truth.
- [ ] Keep the harness pointer files (`CLAUDE.md`, …) thin — shared rules live in `AGENTS.md`.
- [ ] **Design track —** link Figma in `design.md` (no infra needed). When using a *code*
      prototype, enable GitHub Pages (Source: *GitHub Actions*) or connect Vercel / Cloudflare
      and add the token via `gh secret set`.
- [ ] Make `PR links an Issue` a **required** check in the `main` ruleset.
