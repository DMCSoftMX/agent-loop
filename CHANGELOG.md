# Changelog

Every released tag of the engine, newest first. A project repo consumes a tag by pinning it in its
stub (`uses: DMCSoftMX/agent-loop/.github/workflows/<phase>.yml@vX.Y.Z`), so **a version is only
real once it is tagged AND its `stubs/loop.yml` pins itself** — see [RELEASING.md](RELEASING.md).

## v1.1.3 — `implement` publishes the branch itself — 2026-09-05

`implement` no longer depends on the agent pushing. A deterministic step now publishes the branch
with `GITHUB_TOKEN` after the agent returns — the way `specify` already publishes the spec comment
instead of trusting the agent to post it.

`v1.1.2` fixed the prompt that told the agent *not* to push, but a prompt is a request. Every
failure in this release cycle had the same shape: the loop asked the agent for an artifact and
believed the report instead of checking. This closes the last one in `implement`.

The prompt still tells the agent to push, on purpose — the action's own instructions say the same,
and contradicting them is exactly what broke `v1.1.0`. In the happy path the new step is a no-op
(*"Everything up-to-date"*); it earns its place on the runs where it is not.

Two guards it carries:

- **It never publishes an empty branch.** If the agent committed nothing, `HEAD` is still the base
  commit — pushing it would satisfy the ref assertion, open a PR with no diff, and send the four
  gates chasing nothing. The step counts commits ahead of `github.sha` and stays out of the way,
  leaving the assertion to fail loudly.
- **It never force-pushes.** A divergence means something unexpected happened; that should fail, not
  be clobbered.

This does not touch ADR-0003. What must come from a human is the **pull request** — one opened with
this token fires no `pull_request` workflows, so no gate would run. A branch carries no such
constraint; it is just the artifact the human's click acts on.

Adopt it by bumping the pin; nothing else changes.

## v1.1.2 — `implement` tells the agent to push — 2026-09-05

`implement`'s prompt forbade the one thing the phase needs:

> *"Do NOT try to open the PR yourself and do NOT run `gh` or `git push` — you do not have those
> tools ... Committing the branch is your last action."*

That is false. `claude-code-action` grants `Bash(.../scripts/git-push.sh:*)` and instructs the agent
to use it, so the engine's prompt and the action's own contradicted each other — and models broke
the tie differently. `sonnet` pushed anyway; `claude-opus-5` obeyed the explicit prohibition and
stopped after the local commit, so no ref ever reached the remote. Before `v1.1.1` that run still
reported green.

Isolated on `agent-loop-smoke#27` — same repo, same issue, same spec hash, same engine, same action
version, six minutes apart: `--model claude-opus-5` produced no branch twice, `--model sonnet`
pushed one with the correct diff and pin. The model was the proximate difference; the prompt was the
bug, latent since it was written and hidden by a model that ignored it.

Item 5 now tells the agent to publish the branch with the granted script, and keeps the real
constraint: it must not open the PR itself, because a PR opened with this token fires no gates
(ADR-0003) — the human's click is the mechanism.

Adopt it by bumping the pin; nothing else changes.

## v1.1.1 — `implement` asserts the ref, not the name — 2026-09-05

`implement`'s branch assertion trusted `steps.claude.outputs.branch_name` — the name the action
*intends* to use, which it reports whether or not the agent ever pushed. When the agent edited
files, reported success and pushed nothing (first seen on `agent-loop-smoke#27`), the step went
**green**, wrote a "Create the PR ➔" link pointing at a ref that did not exist, and the agent's own
comment claimed the commit had landed. The step's comment promised the opposite: *"No branch → the
agent committed nothing → fail loud instead of reporting a green run with no artifact."* It was
asserting a string was non-empty.

Every candidate name — from the action's output or from the `matching-refs` fallback — is now
confirmed against the remote (`gh api repos/<repo>/git/ref/heads/<branch>`) before the step will
surface it. A reported-but-unpushed branch logs a warning and falls through to the fallback query;
if nothing was pushed at all, the step fails and says so.

Present since `v0.7.2`, when the assertion replaced the auto-opened PR (ADR-0003). Nothing else
changes — adopt it by bumping the pin.

## v1.1.0 — Opus 5 for the authoring phases — 2026-09-05

The three phases that *write* — `specify`, `plan` and `implement` — now default to
**`claude-opus-5`** instead of `sonnet`. `review`, `claude` and `preflight` are unchanged and stay
on `sonnet`.

Nothing about the interface moves: the optional `model` input each phase already exposed is
untouched, so a repo that wants the old behavior keeps it from its own stub:

```yaml
  implement:
    uses: DMCSoftMX/agent-loop/.github/workflows/implement.yml@v1.1.0
    with:
      model: sonnet
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

The new default is a **pinned model id**, not the floating `opus` alias — the engine asks for the
same model on every run until a release says otherwise.

⚠️ `preflight` still pings on `sonnet`, so a green preflight does **not** prove the account can
reach `claude-opus-5`. If the model is unavailable on the plan, `specify` is where it surfaces.

Adopt it by bumping the pin; nothing else changes.

## v1.0.0 — stable interface — 2026-09-05

**No functional change.** The reusable workflows are byte-identical to `v0.8.0`; only the version
strings move. What this tag adds is a promise: the surface listed under
[Stability](README.md#stability) will not change without a major bump.

It is cut now because the last thing that could have forced an early major — renaming the loop's
labels, branch prefix and trigger phrase — was decided against on 2026-09-05: the names stay.

Adopt it by bumping the pin; a repo on `v0.8.0` gains nothing but the guarantee.

## v0.8.0 — release hygiene — 2026-09-05

- **MIT `LICENSE`.** The repo is public and had none, which legally meant nobody could reuse it.
- **This changelog** and **[RELEASING.md](RELEASING.md)**, the checklist whose absence caused the
  `v0.7.2` mistake below.
- **`tag-guard`**: pushing a tag whose `stubs/loop.yml` does not pin that same tag now fails loudly,
  so the `v0.7.2` trap cannot recur.
- **`preflight` reports pin freshness**: it compares the caller's pins against the newest engine tag
  and says so, instead of leaving a repo silently a version behind.
- **`stubs/dependabot.yml` replaces `stubs/renovate.json`** (see agent-loop#19): Renovate was never
  installed on any account, so the file it shipped never opened a single bump PR. Dependabot needs
  no app and tracks reusable-workflow refs.

No change to the phases or the gates: a repo moves to `v0.8.0` by bumping its pin, nothing else.

## v0.7.3 — 2026-08-01

Tag hygiene only. **Its reusable workflows are byte-identical to `v0.7.2`**; what it fixes is the
`stubs/loop.yml` shipped *inside* the tag, which `v0.7.2` left pinning `v0.7.1`.

## v0.7.2 — 2026-08-01

- **`implement` asserts the branch instead of opening the PR** (ADR-0003). The human clicks the
  pre-filled "Create the PR ➔" link in the run summary — that click is the mechanism: a PR opened
  with the loop's own token does not trigger `pull_request`, so no gate would run on it.
- ⚠️ The tag ships a `stubs/loop.yml` still pinned at the broken `v0.7.1`. Copy the stub from
  `v0.7.3` or later.

## v0.7.1 — 2026-08-01 — **BROKEN, never pin it**

`implement` tried to open the PR itself and always died with a 401: `claude-code-action` revokes its
App token before returning (`DELETE /installation/token`). Reverted in `v0.7.2`.

## v0.7.0 — 2026-07-25 — honest gates

- `review` really reviews: it must emit `REVIEW-VERDICT: PASS|BLOCK`, and a `BLOCK` — or a run that
  produced no verdict at all — turns the check red.
- Phases surface `permission_denials_count` instead of passing silently.
- `preflight` reports the **enforcement level**: whether the gates actually block a merge or are
  advisory (they are advisory on a private Free repo, where the human is the gate).
- Dropped the invalid `administration` permission key, which broke stub parsing.

## v0.6.0 — 2026-07-24 — specs as issue comments (ADR-0002)

The spec stops being a file in a dedicated repo and becomes the issue's canonical comment. The pin
`.specs/<n>.ref` moves from a git sha to a **sha256 of the comment body**, so drift detection
survives. Consequences: no GitHub App, no cross-repo token, no specs repo, and `spec-guard` becomes
secret-free — which is why it works on fork PRs.

## v0.5.0 – v0.5.4 — 2026-07-23 — specs in a dedicated repo (ADR-0001, retired)

The spec lived in `<account>/specs`, referenced from the code repo. It worked and was validated end
to end, but the first cross-account adoption exposed the whole tax it implied — an App to mint a
cross-repo token, public repos, one specs repo per account, secret sprawl — which is what ADR-0002
removed.

## v0.1.0 – v0.4.1 — 2026-07-14 — pre-SDD

The loop before specs existed: triage, interactive `@claude`, review and CI, wired as reusable
workflows.
