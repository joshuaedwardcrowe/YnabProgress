# Continuous Input

Status: Superseded by KitCli's `workflow-run-state-machine.md` concept doc
Date: 2026-08-02 (superseded)

## Premise

This ADR originally recorded how a `CliWorkflowRun` stays active across
multiple user inputs — paging, multi-step pipelines — by looping back
to `Running` instead of resetting per command, via a
`ReachedReusableOutcome` state.

## Superseded

That mechanism is `KitCli.Workflow` machinery, not a SpendfulnessCli
decision — nothing in it is Spendfulness-specific. It's now documented
in KitCli itself, more accurately and in more depth than this ADR ever
was (this ADR predates `MovePastAsk`/paging support, and used the old
`CliCommandOutcomeKind`/"property" terminology KitCli has since renamed
to `OutcomeKind`/"artefact"):

- [0010-workflow-run-state-machine.md](https://github.com/KitCli/KitCli/blob/main/docs/concepts/0010-workflow-run-state-machine.md) — the state machine and its transition table.
- [0006-outcomes.md](https://github.com/KitCli/KitCli/blob/main/docs/concepts/0006-outcomes.md) — reusable vs. final outcome kinds.
- [0008-artefacts.md](https://github.com/KitCli/KitCli/blob/main/docs/concepts/0008-artefacts.md) — how a reusable outcome becomes queryable by a later command.

This file is kept only as a pointer. See [KitCli#69](https://github.com/KitCli/KitCli/issues/69) for the discussion that retired it.
