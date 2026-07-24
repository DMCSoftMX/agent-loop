# agent-loop — central engine

The **versioned engine** for the spec-driven agent loop. Its logic, templates and prompts are
maintained **once, here**. Each project repo carries only a thin stub that *calls* this engine —
no copied workflows. Change something here + tag → Renovate opens a bump PR in every project.

> **Status: all 8 phases wired** as reusable workflows, plus a `preflight` setup validator. A
> project consumes them by copying one thin stub ([`stubs/loop.yml`](stubs/loop.yml)).

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
│   ├── ci.yml            stack-aware validate (reads setup.env)
│   └── preflight.yml     one-shot setup check (workflow_dispatch): secret · token · App · config
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
    uses: DMCSoftMX/agent-loop/.github/workflows/specify.yml@v0.5.0
    secrets: inherit
  # … plan · implement · claude · review · pr-gate · spec-guard · ci · preflight (same shape)
```

That's the whole footprint. Plus the per-repo data that rightly stays local: `CLAUDE.md`,
`setup.env`, and the `.specs/<issue#>.ref` pins.

**Since v0.5.0 the specs themselves are NOT local.** They live in
[`DMCSoftMX/specs`](https://github.com/DMCSoftMX/specs) (ADR-0001) — a code repo versions code,
not documents. Each repo keeps only a pointer per issue. See *Cross-repo auth* below: this is the
one thing v0.5.0 asks of you that v0.4.x didn't.

**After wiring the stub, run preflight once** (Actions → *Agent loop* → *Run workflow*). It pings
the App-token exchange and checks the secret, `setup.env`, labels and default branch, then fails
with a clear checklist — so a misconfigured repo surfaces the problem here instead of in the first
real `specify`. It only fires on `workflow_dispatch`; normal events skip it. (One caveat it *can't*
catch from the inside: if the stub lacks its `permissions:` block or engine access isn't
`organization`, preflight itself won't start — a `startup_failure` on preflight **is** that
diagnosis.)

## Runtime model (the key tricks)

- **No engine token, ever:** the runner never checks out this engine repo, so `agent-loop` stays
  **private** with no per-repo PAT. The phases check out the caller (for `CLAUDE.md`, `setup.env`)
  and, since v0.5.0, the specs repo at `.specs-repo/` — never this one.
- **Cross-repo auth (v0.5.0, new requirement):** the caller's default `GITHUB_TOKEN` is scoped to
  the caller, so it cannot read or write the specs repo. Each phase mints a short-lived App
  installation token covering *both* repos from `LOOP_APP_ID` + `LOOP_APP_PRIVATE_KEY`. Those must
  be set **per repo** (org secrets aren't available to private repos on this plan). `specify`,
  `plan`, `implement` and `spec-guard` fail loudly without them; `review` and `claude` degrade to
  running without spec context. `preflight` verifies the whole chain — including that the minted
  token can actually *write* to the specs repo — before you run a real phase.
- **Templates moved out:** spec/plan/tasks templates now live in the specs repo under
  `templates/`, so editing one is a PR there instead of an engine tag. The tradeoff: they are no
  longer pinned to `@vX`.
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

| Central — engine (here) | Central — specs repo | Per-repo (each project) |
|---|---|---|
| Reusable workflows (logic) | The specs themselves | Thin stub (`loop.yml`) |
| Phase prompts | spec/plan/tasks templates | `CLAUDE.md`, `setup.env` |
| Gate logic (pr-gate, spec-guard) | Spec PRs (the intent gate) | `.specs/<issue#>.ref` pins |
| — | — | Labels, branch protection, secrets |

PR/issue templates can't be "reused" (GitHub reads them from the repo) — publish them as
**org defaults** in `DMCSoftMX/.github`, or sync with Cruft/multi-gitter.
