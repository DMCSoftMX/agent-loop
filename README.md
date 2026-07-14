# agent-loop — central engine

The **versioned engine** for the spec-driven agent loop. Its logic, templates and prompts are
maintained **once, here**. Each project repo carries only a thin stub that *calls* this engine —
no copied workflows. Change something here + tag → Renovate opens a bump PR in every project.

> **Status: all 8 phases wired** as reusable workflows. A project consumes them by copying one
> thin stub ([`stubs/loop.yml`](stubs/loop.yml)).

## Structure

```
agent-loop/                              ← DMCSoftMX/agent-loop (semver-tagged)
├── .github/workflows/                   ← the reusable engine (on: workflow_call)
│   ├── specify.yml       phase 1 · writes spec.md
│   ├── plan.yml          optional escalation · deep plan.md + tasks.md
│   ├── implement.yml     agent implements against the spec
│   ├── claude.yml        interactive @claude
│   ├── review.yml        PR review vs the spec · emits REVIEW-VERDICT
│   ├── pr-gate.yml       binding definition-of-done gate
│   ├── spec-guard.yml    anti-drift: code change must reconcile its spec
│   └── ci.yml            stack-aware validate (reads setup.env)
│   (spec/plan/tasks templates are INLINED in the specify/plan prompts — no separate files,
│    no engine checkout, so this repo can stay private with zero per-repo tokens.)
└── stubs/                what each PROJECT repo copies (once)
    ├── loop.yml          the thin router stub (.github/workflows/loop.yml)
    └── renovate.json     auto-bumps the engine version across projects
```

## How a project consumes it

Copy [`stubs/loop.yml`](stubs/loop.yml) to `.github/workflows/loop.yml`. That single file routes
every event (labels, `@claude`, PRs, push) to the reusable workflows here, pinned to `@vX`:

```yaml
jobs:
  specify:
    if: github.event_name == 'issues' && github.event.label.name == 'specify'
    uses: DMCSoftMX/agent-loop/.github/workflows/specify.yml@v0.2.0
    secrets: inherit
  # … plan · implement · claude · review · pr-gate · spec-guard · ci (same shape)
```

That's the whole footprint. Plus the per-repo data that rightly stays local: `CLAUDE.md`,
`setup.env`, `specs/`.

## Runtime model (the key tricks)

- **One checkout, no engine token:** the reusable checks out only the **caller repo** (for
  `CLAUDE.md`, `setup.env`, `specs/`). The spec/plan/tasks **templates are inlined in the phase
  prompts**, so the runner never checks out this engine repo — which means `agent-loop` stays
  **private** with no per-repo PAT, and the template is automatically the one from the pinned
  `@vX` (it travels with the workflow file).
- **The stub grants the token ceiling:** a called reusable can't exceed the caller's permissions,
  so `stubs/loop.yml` sets top-level `permissions:` (contents/PR/issues/id-token write). Without
  it every write phase dies at **startup** on a read-only default token.
- **Stack from `setup.env` at runtime:** `implement`, `claude` and `ci` `source setup.env` from
  the caller, set up the toolchain per `STACK`, and use `INSTALL_CMD` / `ALLOWED_VERIFY_TOOLS` /
  the lint·typecheck·test·build commands. So the stub needs **zero** stack config.

## Versioning & propagation

- Engine released with **semver tags** (`v0.2.0`, `v0.3.0`, …).
- Projects pin `@vX` in their stub. **Renovate** ([`stubs/renovate.json`](stubs/renovate.json))
  opens a **bump PR** in each project on a new tag → you merge it (human gate preserved).

## Branch protection ⚠️ (check-name change)

As reusable workflows, the required-check **context is `<stub-job> / <engine-job>`**. Set branch
protection on `develop` to require:

- `ci / validate`
- `pr-gate / pr-gate`
- `spec-guard / spec-guard`

(and on `main`: `ci / validate` only — the gates skip on release PRs).

## What stays central vs per-repo

| Central (here, versioned) | Per-repo (in each project) |
|---|---|
| Reusable workflows (logic) | Thin stub (`loop.yml`) |
| Prompts + inlined spec/plan/tasks templates | `CLAUDE.md`, `setup.env` |
| Gate logic (pr-gate, spec-guard) | `specs/` (generated data) |
| — | Labels + branch protection (settings) |

PR/issue templates can't be "reused" (GitHub reads them from the repo) — publish them as
**org defaults** in `DMCSoftMX/.github`, or sync with Cruft/multi-gitter.
