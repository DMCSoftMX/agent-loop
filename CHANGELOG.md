# Changelog

Every released tag of the engine, newest first. A project repo consumes a tag by pinning it in its
stub (`uses: DMCSoftMX/agent-loop/.github/workflows/<phase>.yml@vX.Y.Z`), so **a version is only
real once it is tagged AND its `stubs/loop.yml` pins itself** — see [RELEASING.md](RELEASING.md).

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
