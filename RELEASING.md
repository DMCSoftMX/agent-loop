# Releasing the engine

A tag is the only thing project repos consume. Everything below exists because two releases went out
wrong without it: **`v0.7.2` shipped a stub pinned at the broken `v0.7.1`**, and `v0.7.3` was tagged
but never published, so GitHub kept announcing `v0.7.0` as *Latest* for a month.

## Checklist

1. **`main` is green and everything you want is merged.** The engine has no CI of its own; the
   validation is step 5, on the smoke repo.
2. **Bump the stub template to the version you are about to cut** — `stubs/loop.yml` (nine `uses:`
   lines) and the `@vX.Y.Z` mentions in `README.md`. *This is the step that was skipped in `v0.7.2`.*
   A repo onboarded by copying the stub from the tag gets exactly what this file says.
3. **Update [`CHANGELOG.md`](CHANGELOG.md)**: rename the `(unreleased)` heading to the version and
   date it. Say what a consumer has to do to adopt it — usually "bump the pin, nothing else".
4. **Tag and push**:
   ```sh
   git tag vX.Y.Z && git push origin vX.Y.Z
   ```
   `tag-guard` runs on the push and fails if `stubs/loop.yml` does not pin `vX.Y.Z`. If it goes red:
   `git push --delete origin vX.Y.Z`, fix step 2, tag again. Deleting a tag nobody has adopted yet is
   cheap; a wrong tag that repos have already pinned is not.
5. **Validate end to end on `DMCSoftMX/agent-loop-smoke`**: bump its pin to the new tag, **merge that
   bump**, and run from `develop` — not from the bump's branch. `claude-code-action` refuses to run
   when the workflow file differs from the one on the default branch ("Skipping action due to
   workflow validation"), so a dispatch against the branch skips the Claude ping and preflight ends
   red for a reason that has nothing to do with the release. Then run one full lap — issue → `specify` → edit the spec comment → `claude-implement` → click "Create the
   PR ➔" → the four gates green → merge. If the release changes a phase, do the lap; if it only
   touches docs or stubs, `preflight` is enough.
6. **Publish the release** so *Latest* tells the truth:
   ```sh
   gh release create vX.Y.Z --title "vX.Y.Z — <headline>" --notes-file <(sed -n '/^## vX.Y.Z/,/^## /p' CHANGELOG.md)
   ```
   Mark a known-broken release as a pre-release (`--prerelease`) so it stops being offered.
7. **Let the consumers catch up.** Each repo carries `.github/dependabot.yml` scoped to this engine,
   so the bump arrives as one grouped PR per repo within the week — you merge it. Bump by hand only
   when you need it today.

## Version numbers

- **Patch** — a fix inside a phase that changes nothing for the caller.
- **Minor** — new behavior, new optional inputs, anything a repo gets simply by bumping the pin.
- **Major** — anything that makes an existing stub wrong: renaming a label, a job, the branch
  prefix, the trigger phrase, or the `.specs/<n>.ref` format. There is no runtime fallback by
  design: **the pin is the migration mechanism**, so a repo moves when it bumps, never before.
