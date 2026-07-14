# agent-loop — central engine (prototype)

The **versioned engine** for the spec-driven agent loop. Its logic, templates and prompts are
maintained **once, here**. Each project repo carries only a thin stub that *calls* this engine —
no copied workflows. Change something here + tag → Renovate opens a bump PR in every project.

> **Status: prototype.** Only the `specify` phase is wired (1 of 7). `plan`, `implement`,
> `@claude`, `review`, `ci`, `pr-gate`, `spec-guard` follow the **identical** pattern.
> This directory is the content that would become its own repo, `DMCSoftMX/agent-loop`.

## Structure

```
agent-loop/                              ← future DMCSoftMX/agent-loop repo (semver-tagged)
├── .github/workflows/
│   └── specify.yml          on: workflow_call   ← the reusable SPECIFY phase
├── templates/
│   └── spec.md              the spec template, fetched at runtime
└── stubs/                   what each PROJECT repo copies (once)
    ├── loop.yml             the thin caller stub (.github/workflows/loop.yml)
    └── renovate.json        auto-bumps the engine version across projects
```

## How a project consumes it

The project repo has one thin workflow, `.github/workflows/loop.yml` (see [`stubs/loop.yml`](stubs/loop.yml)):

```yaml
on: { issues: { types: [labeled] } }
jobs:
  specify:
    if: github.event.label.name == 'specify'
    uses: DMCSoftMX/agent-loop/.github/workflows/specify.yml@v0.1.0
    secrets: inherit
```

That's the whole footprint. Plus the per-repo data that rightly stays local: `CLAUDE.md`,
`setup.env`, `specs/`.

## Runtime model (the key trick)

When the reusable workflow runs it does **two** checkouts:

1. **The caller repo** (default checkout) → reads `CLAUDE.md`, writes the spec into `specs/`.
2. **This engine**, at the *exact version being used* (`github.job_workflow_sha`) → gets
   `templates/` + prompts. No version duplication: the templates always match the `@vX` in the stub.

So the project needs zero config in the stub — `setup.env`/`CLAUDE.md`/`specs/` are read from the
caller at runtime.

## Versioning & propagation

- Engine is released with **semver tags** (`v0.3.0`, `v0.4.0`, …).
- Projects pin `@vX` in their stub. **Renovate** ([`stubs/renovate.json`](stubs/renovate.json))
  detects a new tag and opens a **bump PR** in each project → you merge it (human gate preserved).
- This resolves the moving-tag-vs-pinned dilemma: propagation is automatic **but reviewable**,
  not applied blindly.

## What stays central vs per-repo

| Central (here, versioned) | Per-repo (in each project) |
|---|---|
| Reusable workflows (logic) | Thin stub (`loop.yml`) |
| `templates/`, prompts | `CLAUDE.md`, `setup.env` |
| Gate logic (pr-gate, spec-guard) | `specs/` (generated data) |
| — | Labels + branch protection (settings) |

PR/issue templates can't be "reused" (GitHub reads them from the repo) — publish them as
**org defaults** in `DMCSoftMX/.github`, or sync with Cruft/multi-gitter.
