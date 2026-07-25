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
│   ├── specify.yml       phase 1 · posts the spec as the issue's canonical comment
│   ├── plan.yml          optional escalation · posts a deep plan comment
│   ├── implement.yml     agent implements against the spec · writes the .specs/<n>.ref pin
│   ├── claude.yml        interactive @claude
│   ├── review.yml        PR review vs the spec · posts it · REVIEW-VERDICT BLOCK ⇒ red check
│   ├── pr-gate.yml       binding definition-of-done gate
│   ├── spec-guard.yml    anti-drift: re-hashes the spec comment, must match the pin
│   ├── ci.yml            stack-aware validate (reads setup.env)
│   └── preflight.yml     one-shot setup check (workflow_dispatch): secret · Claude App · config · gate enforcement level
│   (spec/plan templates are INLINED in the specify/plan prompts — no separate files, no engine
│    checkout, so this repo can stay private with zero per-repo tokens.)
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
    uses: DMCSoftMX/agent-loop/.github/workflows/specify.yml@v0.7.0
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
  # … plan · implement · claude · review · pr-gate · spec-guard · ci · preflight (same shape)
```

That's the whole footprint. Plus the per-repo data that rightly stays local: `CLAUDE.md`,
`setup.env`, and the `.specs/<issue#>.ref` pins.

**Since v0.6.0 the spec is a comment on the issue** (ADR-0002) — a canonical comment marked
`<!-- agent-spec:<n> -->` that `specify` posts and you edit inline to approve. Everything happens in
**this repo** with the default `GITHUB_TOKEN`: no dedicated specs repo, no GitHub App, no cross-repo
token, no cross-account tax. The only secret the loop needs is `CLAUDE_CODE_OAUTH_TOKEN`, passed
**explicitly** (not `secrets: inherit`, which does not cross the account/org boundary).

**After wiring the stub, run preflight once** (Actions → *Agent loop* → *Run workflow*). It pings
the Claude App-token exchange and checks the secret, `setup.env`, labels and default branch, then
fails with a clear checklist — so a misconfigured repo surfaces the problem here instead of in the
first real `specify`. It only fires on `workflow_dispatch`; normal events skip it. (One caveat it
*can't* catch from the inside: if the stub lacks its `permissions:` block, preflight itself won't
start — a `startup_failure` on preflight **is** that diagnosis.)

## The SDD flow (v0.7.0)

1. **`specify`** (label `specify`) → the agent drafts the spec; a deterministic step upserts it as
   the issue's canonical comment. Re-running edits that same comment in place.
2. **Approve** → read the comment, edit it inline if needed. Adding `claude-implement` is the
   go-ahead (or `no-spec` to implement without a spec).
3. **`plan`** (optional, label `plan`) → posts a deep-design comment that supersedes the spec's
   light approach.
4. **`claude-implement`** → the agent reads the spec comment, implements, and commits the code plus
   `.specs/<n>.ref`, then opens a PR. **Fail-closed:** with no spec comment and no `no-spec` label,
   it stops rather than silently building from the issue body.
5. **Gates** on the PR → `review` · `pr-gate` · `spec-guard` · `ci`. **`spec-guard`** re-fetches the
   pinned comment and re-hashes it: if someone edited the spec after the code was written, the hash
   no longer matches and the PR goes red until it is reconciled. **`review`** posts its review
   against the spec and ends with `REVIEW-VERDICT: PASS|BLOCK`; a `BLOCK` (or a run that produced no
   verdict at all) turns the check red — a green `review` now means it actually reviewed, not that it
   stayed silent.

> **Enforcement is only as strong as your plan.** These gates *report* on every PR, but they only
> *block a merge* where the branch is protected and admins are included. On a **private Free** repo
> the protection API is unavailable, so the gates are **advisory** — a green loop is not an enforced
> loop, and the human is the merge gate. `preflight` prints the enforcement level for `develop` and
> `main` so you know which world you're in; see *Branch protection* below.

## Runtime model (the key tricks)

- **No engine token, ever:** the runner never checks out this engine repo, so `agent-loop` stays
  **private** with no per-repo PAT. The phases check out only the caller (for `CLAUDE.md`,
  `setup.env`, and the PR's `.specs/<n>.ref`).
- **The spec is an issue comment; the pin is a content hash.** `.specs/<n>.ref` records
  `source=issue-comment`, the `comment_id`, and `spec_sha256` — the hash of the comment body the
  code was written against. `implement` and `spec-guard` use the *identical* hash pipeline
  (`gh api …/comments/<id> --jq .body | tr -d '\r' | sha256sum`), so the pin reconciles exactly.
- **Only `GITHUB_TOKEN`:** every phase acts on its own repo's issue/PR, so no App and no cross-repo
  token. A welcome side effect: `spec-guard` is secret-free again and **works on fork PRs**.
- **Templates are inlined** in the specify/plan prompts, so they are pinned with `@vX` and need no
  separate repo or checkout.
- **The stub grants the token ceiling:** a called reusable can't exceed the caller's permissions,
  so `stubs/loop.yml` sets top-level `permissions:` (contents/PR/issues/id-token write). Without
  it every write phase dies at **startup** on a read-only default token.
- **Stack from `setup.env` at runtime:** `implement`, `claude` and `ci` `source setup.env` from
  the caller, set up the toolchain per `STACK`, and use `INSTALL_CMD` / `ALLOWED_VERIFY_TOOLS` /
  the lint·typecheck·test·build commands. So the stub needs **zero** stack config.

## Versioning & propagation

- Engine released with **semver tags** (`v0.2.0`, `v0.3.0`, …). The `@vX` pin **is** the migration
  mechanism: `v0.5.x` = specs in a dedicated repo (ADR-0001), `v0.6.x` = specs as issue comments
  (ADR-0002), `v0.7.x` = `review` posts + emits an enforceable verdict, agent phases surface
  `permission_denials_count`, and `preflight` reports the gate enforcement level. No runtime
  fallback — a repo migrates by bumping its pin.
- Projects pin `@vX` in their stub. **Renovate** ([`stubs/renovate.json`](stubs/renovate.json))
  opens a **bump PR** in each project on a new tag → you merge it (human gate preserved).

## Branch protection ⚠️ (check-name change)

As reusable workflows, the required-check **context is `<stub-job> / <engine-job>`**. Set branch
protection on `develop` to require:

- `ci / validate`
- `pr-gate / pr-gate`
- `spec-guard / spec-guard`
- `review / review` — *optional*: `review` can now turn red on `REVIEW-VERDICT: BLOCK`, so you *may*
  require it. Expect the occasional `--admin` override when the agent blocks a false positive.

(and on `main`: `ci / validate` only — the gates skip on release PRs).

**But protection only bites where the plan allows it.** On a **private Free** repo the
branch-protection API returns `403 Upgrade…`, so none of the above can be *required* — the checks run
and report but nothing stops the merge. And requiring a **review approval** on a solo repo's `main`
can only be satisfied by an `--admin` bypass (you can't approve your own PR), which just trains
everyone to bypass. So be honest about it: **on this plan the human is the gate.** Run `preflight`
(it now prints, per branch, whether the branch is *protected* or *advisory only* — it reads the
branch's `protected` flag with the default token; the finer *admins-bypass* detail needs admin the
loop token doesn't have, so confirm "Include administrators" yourself in Settings → Branches).

## What stays central vs per-repo

| Central — engine (here) | Per-repo (each project) |
|---|---|
| Reusable workflows (logic) | Thin stub (`loop.yml`) |
| Phase prompts + inlined templates | `CLAUDE.md`, `setup.env` |
| Gate logic (pr-gate, spec-guard) | `.specs/<issue#>.ref` pins |
| — | The spec comments live on the issues |
| — | Labels, branch protection, the one secret |

PR/issue templates can't be "reused" (GitHub reads them from the repo) — publish them as
**org defaults** in `DMCSoftMX/.github`, or sync with Cruft/multi-gitter.
