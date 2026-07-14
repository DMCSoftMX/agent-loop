# Spec: <<title>>

> **Refs:** #<<issue>>  ·  **Status:** draft → approved (on merge)
> The WHAT and WHY, plus a **lightweight HOW**. For heavy or cross-cutting work, escalate the
> design to a full `plan.md` via the `plan` label — otherwise keep it inline below.

## Problem / motivation

<<What need or problem exists today, and why it matters. 1–3 sentences.>>

## Goal

<<The outcome in one sentence. What is true when this is done.>>

## Scope

**In:**
- <<what this work covers>>

**Out (non-goals):**
- <<what this work explicitly does NOT cover>>

## User-visible behavior

<<How the change is observed by a user / caller / consumer. Before → after.>>

## Acceptance criteria

<!-- Specific and VERIFIABLE. This is what review checks the PR against; anything left
     uncovered is BLOCKING. Prefer observable outcomes over implementation steps. -->

- [ ] <<criterion 1>>
- [ ] <<criterion 2>>
- [ ] CI green (lint · typecheck · test · build)
- [ ] Follows the conventions in CLAUDE.md

## Approach & tasks

<!-- Lightweight HOW: the intended approach in a few lines + a short task checklist. Enough for
     the implementer not to guess. If this section starts needing real architecture, contracts
     or a data model, STOP and escalate to a full plan.md (label `plan`). -->

<<one or two sentences on the intended approach; name the main files/areas>>

- [ ] <<task 1>>
- [ ] <<task 2>>

## Edge cases & risks

- <<edge case / failure mode / runtime behavior CI can't verify>>

## Open questions

- <<anything that needs a human decision before/while implementing>>
