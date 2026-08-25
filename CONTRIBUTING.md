# Contributing to SpendfulnessCli

SpendfulnessCli is a personal tool, not a published library — the
process below is deliberately lighter than [KitCli](https://github.com/KitCli/KitCli)'s
or [YnabSharp](https://github.com/joshuaedwardcrowe/YnabSharp)'s, but
keeps the same core habit: non-obvious decisions get a paper trail
instead of relying on memory.

## Before you write code

- **Bugs and small fixes** — just open a PR. No issue required.
- **New commands, or anything that changes how a command reports or
  aggregates data** — open an issue first. Get the shape agreed before
  investing in the implementation.
- **Architectural decisions** — see [ADRs](#adrs) below.

## Branching & PRs

- Branch off `main`, one branch per issue (e.g.
  `163-splittable-transactions-personalisation`). No long-running
  branches.
- One logical change per PR.
- **Keep PRs small: max 20 files, 10-15 preferred.** If a change is going
  to blow past that, plan the split into multiple PRs upfront, not after
  the fact.
- **PR titles use [Conventional Commits](https://www.conventionalcommits.org/):**
  `<type>(scope): <description>` — `type` is one of `feat` `fix` `docs`
  `chore` `refactor` `test` `ci`; `scope` (optional) matches a command
  area (`chat`, `export`, `organisation`, `personalisation`,
  `reporting`, `reusable`). `description` is lowercase, imperative, no
  trailing period. For a breaking change, add `!` right before the
  colon. Example: `feat(reporting): add loan-to-value command`.
- **If a linked issue exists, mirror its labels and milestone onto the
  PR.** GitHub doesn't do this automatically. Keeping both in sync means
  milestone/label filtering and progress tracking work across the PR
  list too, not just issues.
- There's no enforced CI or required review on this repo (single
  maintainer, no branch protection) — build and test locally before
  merging anyway.
- Docs-only changes (like this file) can be committed straight to
  `main`.

## Testing

- **Build test doubles reusably from the start, not as a private nested
  class you promote later.** Put the double in the relevant test
  project's `TestHelpers/` (or equivalent) folder the first time, not
  the second.
- **Serialize real DTOs instead of hand-writing JSON/CSV string
  literals** for canned response bodies — construct the actual type and
  serialize it, so the fixture can't drift from the real shape.
- **Name test doubles `Test*`**, not `Stub*`/`Fake*`/`Mock*`.

## ADRs

An ADR ([`docs/adr/`](docs/adr/)) captures a decision — its premise, the
problem, and the solution — not how something works today (that's
[`docs/concepts/`](docs/concepts/)). Number sequentially
(`000x-title-in-kebab-case.md`), starting from the seed template at
[`docs/adr/0000-template.md`](docs/adr/0000-template.md).
[`docs/adr/future-adr.md`](docs/adr/future-adr.md) tracks decisions that
are known to need one but haven't been written yet.

**Write one when you're** introducing a new cross-cutting pattern,
changing a project boundary, or making a breaking change to a command's
shape. **Skip it for** bug fixes and internal refactors.

## Concepts

[`docs/concepts/`](docs/concepts/) explains how the system's pieces fit
together today — one sequentially-numbered file per concept
([`0001-console-pipeline.md`](docs/concepts/0001-console-pipeline.md),
[`0002-ynab-client-layers.md`](docs/concepts/0002-ynab-client-layers.md)), seeded from
[`0000-template.md`](docs/concepts/0000-template.md). Keep them current:
if a change makes one inaccurate, update it in the same PR.

## Investigations

An investigation ([`docs/investigations/`](docs/investigations/)) is what
a technical spike produces — the finding, not the code, since there's no
pair here to carry it forward otherwise. Number sequentially
(`000x-question.md`) from
[`0000-template.md`](docs/investigations/0000-template.md); skip it if a
reader wouldn't otherwise re-derive the finding from scratch.

Lead with the verdict — **new complexity** or **no new complexity** — not
the evidence. On no new complexity the spike closes and a fresh ticket
carries the work; on new complexity the issue that prompted the spike
stays open as the parent, with the build hanging off it as sub-issues in
delivery order. Either way the spike issue closes and is never reused.
The verdict asks whether the work is as small as the issue assumed, not
whether the design questions got answered. An investigation records what
was found, not what was decided — that's an ADR, which it may justify but
doesn't replace. Ships through a PR with a Status like any other work;
durable facts about a dependency also belong in that dependency's own
docs, not only here.

## Issues

Labels: `User Value` (direct value to the CLI's user) · `Developer
Value` (internal/plumbing) · `New Command` · `Tech Debt` · `Tech Debt -
Preferential` · `Bug` · `Enhancement` · `Documentation` · `Integration`
· `Spike`.

`User Value` is a value classification, not an Agile user-story
format — don't force issues into "As a user, I want..." shape. Larger
`User Value` issues get broken into sub-issues through backlog
refinement (see [Projects](#projects) below), a few at a time — not all
at once upfront.

**Issue titles** follow a two-stage convention:

- **Idea-stage** (unvalidated, pre-WAG) — plain-language problem
  statements, e.g. "No way to identify X" / "No command to do Y". This
  is deliberate: an idea is a pitch for an unmet need, not yet a scoped
  unit of work.
- **Delivery-stage sub-issues** (carved out by a planning spike, ready
  to build) — Conventional Commits style, matching PR titles:
  `type(scope): description`, e.g. `feat(reporting): add loan-to-value
  command`. By this point the work is scoped, so the title should read
  like the commit that will close it.

## Projects

Work bigger than a single issue goes through a pipeline biased toward
re-planning over predicting — estimates are inputs to prioritization,
not commitments to defend:

1. **WAG** — a fast, rough gut-feel estimate (in months), logged on
   [SpendfulnessCli's own Ideas board](https://github.com/users/joshuaedwardcrowe/projects/13)'s
   `WAG (months)` field, purely to judge whether an idea is worth
   pursuing at all. Non-binding — expected to be wrong.
2. **SWAG** — the same estimate, re-checked against everything else
   competing for the slot, logged in the same board's `SWAG (months)`
   field. **Setting `Priority` (`High`/`Medium`/`Low`) is mandatory at
   this point** — Status can't move to `SWAG'd / Prioritized` until
   it's set, forcing an explicit call on how the idea stacks up against
   what's already prioritized. "Prioritizing" then means
   sorting/grouping the board by `Priority` or `SWAG` — there's no
   separate roadmap artifact to keep in sync. Still non-binding: a
   relative sizing input, not a plan.
3. **New GitHub Project** — once an idea is greenlit, it graduates off
   the Ideas board into its own project (e.g.
   [Spendfulness](https://github.com/users/joshuaedwardcrowe/projects/9),
   [YNAB Analysis & Automation](https://github.com/users/joshuaedwardcrowe/projects/8)).
4. **Inception spike** — plans the *next* milestone in real detail;
   everything beyond that is a rough forecast, re-planned properly once
   you actually get there (rolling-wave planning, not a full plan for
   the whole estimate up front). Refresh the Ideas board's `Validated
   Estimate (months)` field as it's learned, not just once.
5. **Backlog refinement, just-in-time** — rather than one big spike
   producing the full chronological order for an entire milestone, only
   the next handful of tickets need to be fully ordered and estimated
   at any moment. The rest of the milestone stays a loosely-ordered
   backlog, refined incrementally as work proceeds. A milestone-scale
   re-planning pass is still useful when picking up a milestone cold —
   treat its output as a starting point, not a fixed contract.

   A **spike** (a specific, scoped investigation — "should we support
   X," "what does Y actually look like") resolves to one of two
   outcomes: **new complexity found**, or **no new complexity**. On no
   new complexity, close the spike and open a fresh, cleanly-titled
   delivery-stage ticket for the actual build — don't retitle or reuse
   the spike issue in place. That new ticket gets sized in a normal
   backlog-refinement pass, not as part of the spike itself.
6. **Fixed-length iterations + end-of-iteration review** — work in
   short, regular iterations rather than open-ended milestone spans.
   At the end of each one: check what actually got done vs. planned,
   re-prioritize the backlog based on what was learned, and feed the
   iteration's actual pace back into WAG/SWAG calibration. This
   inspect-and-adapt step is what keeps the rest of the pipeline
   honest — without it, WAG/SWAG/the inception spike are just a plan
   nobody revisits.
7. **Tickets with Estimates** — the leaf/actionable tickets pulled into
   an iteration get the `Estimate` field (Fibonacci story points, not
   time) on the project board — the parent story tracks the outcome,
   not the effort to reach it. Don't second-guess an estimate just
   because a ticket is taking a while — re-estimate only on genuine
   scope change; see [SoloCAIRN's Sizing
   note](https://github.com/joshuaedwardcrowe/SoloCAIRN/blob/main/docs/03-lifecycle.md)
   for the full reasoning.

### Status gates

Once a ticket has an Estimate (step 7 above), the delivery boards'
(YNAB Analysis & Automation, Spendfulness) `Status` field tracks it
through these Agile/Scrum-style gates. Move the card at each transition
— don't let it sit stale while the real work moves past it:

| Status | Move here when |
|---|---|
| `Backlog` | Triaged (has all three labels) but not yet pulled into an iteration. |
| `To Do` | Pulled into the current iteration and given an `Estimate`. |
| `In Development` | Implementation starts — first commit/branch pushed. |
| `In Review` | PR opened, labels/milestone mirrored from the issue. |
| `In QA` | All review feedback addressed and the change has been run locally — this repo has no enforced CI, so manual verification is the gate, not a green check. |
| `Ready for Release` | Merged, but deliberately held back from `Done` — e.g. waiting on a batch of related work. Optional: most tickets skip straight to `Done` on merge. |
| `Done` | Merged and the issue is closed. This is the default landing spot from `In QA` — don't hold a ticket at `Ready for Release` without a specific reason to. |

This repo follows [SoloCAIRN](https://github.com/joshuaedwardcrowe/SoloCAIRN)
for a ticket's Build-stage lifecycle, with one extension specific to
this repo, not something SoloCAIRN itself prescribes: **the GitHub
Issue itself is the story artifact** — no separate markdown file or
dedicated location. It's already written down, reviewable via
comments, and tracked through GitHub's own history.

Current project boards:

- [YNAB Analysis & Automation](https://github.com/users/joshuaedwardcrowe/projects/8) — analyse and CRUD commands
- [Spendfulness](https://github.com/users/joshuaedwardcrowe/projects/9) — the spendfulness-measurement theme
- [Ideas](https://github.com/users/joshuaedwardcrowe/projects/13) — this repo's own WAG/SWAG staging board described above. Ideas that don't pertain to any one repo (e.g. cross-cutting or not-yet-homed ideas) go on the [shared personal-account Ideas board](https://github.com/users/joshuaedwardcrowe/projects/10) instead.
- Creating a Proof of Concept, Adding Settings, Tech Debt Monitoring — narrower/unexplored scopes

## Milestones

Milestones group issues by feature area within this repo (e.g.
`Measuring Spendfulness`, `Modifying YNAB Data`) — narrower and
repo-scoped, unlike a Project board which can span themes or
work-types. Only the immediate milestone is planned in real detail (via
the **inception spike**, step 4 above); later milestones stay a rough
forecast, refined properly when picked up.

**Naming.** When a milestone is tied to catching up to (or tracking) an
external spec or API version, name it after that version (e.g. `YNAB
API v1.86.0`), not a goal-style description (e.g. `Full YNAB API
Coverage`) — a version-anchored name pins the milestone to a concrete,
checkable target and supports a version history over time. Feature-area
milestones (like the ones above) don't need this — it only applies when
there's an actual external version to anchor to.

## Questions

Open an issue if something in this document is unclear or actively
getting in the way — this document is subject to the same process as
everything else.
