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
  truth)**, CI as the automated gate, optional preview deploy for prototypes.

All agent rules live in [`AGENTS.md`](./AGENTS.md). Read that first.

## Layout

```
specs/
  _template/                # copy into specs/<version>/ to start a version
    spec.md                 #   what & why
    design.md               #   UI/UX intent (design track only)
    plan.md                 #   how it gets built → tasks
    tasks.md                #   staging list before GitHub Issues
  v1/ …                     # one folder per version
prototype/                  # runnable prototype app (design track); deployed to preview
design/tokens/              # design tokens (colour/spacing/type) — optional
.github/workflows/
  ci.yml                    # lint/typecheck/test/build (Node/TS + Python); green on empty repo
  preview.yml               # build prototype/ and publish a phone-viewable preview
AGENTS.md                   # single source of truth for agent behavior
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

Use this for back-end, library, CLI, or any work with no UI to get right first.

### Flow B — with Design (spec → design → prototype → code)

```
spec.md ─▶ [SPEC GATE] ─▶ design.md + prototype/
   ─▶ preview URL (open on your phone) ─▶ [DESIGN GATE: review design + live preview]
   ─▶ (then identical to Flow A) plan.md ─▶ Issues ─▶ implement ─▶ PR ─▶ CI ─▶ review ─▶ merge
```

1. Commander writes `specs/v1/spec.md` → **you review** (spec gate).
2. Commander writes `specs/v1/design.md` (flows, IA, visual direction, tokens, responsive
   breakpoints, a11y target, per-screen acceptance). An executor builds the prototype under
   `prototype/`.
3. Each prototype PR deploys a **preview URL** (`preview.yml`). You open it on your phone and
   review `design.md` + the live preview — the **design gate** — before any production code.
4. Once design is approved, continue exactly like Flow A: `plan.md` → Issues → implement → PR →
   CI → review → merge. The approved prototype is the reference.

Use this whenever there's UI to settle before committing to full code.

## One-time setup checklist

- [ ] Protect `main`: require a PR + passing CI before merge (Settings → Rules / ruleset).
- [ ] Confirm CI is green on the empty scaffold.
- [ ] Point your orchestrator at this repo; set **GitHub Issues** as the backlog source of truth.
- [ ] Keep any `CLAUDE.md` lean — shared rules belong in `AGENTS.md`.
- [ ] **Design track only —** enable previews: either turn on GitHub Pages
      (Settings → Pages → Source: *GitHub Actions*) for a single published preview, or connect
      Vercel / Cloudflare Pages for per-PR URLs and add the token via `gh secret set`.
- [ ] **Design track only —** create the `design` / `prototype` / `code` Issue labels.
