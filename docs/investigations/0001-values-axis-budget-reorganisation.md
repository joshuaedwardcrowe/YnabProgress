# 0001. Can SpendfulnessCli perform a values-axis budget re-organisation?

- **Status:** In Review
- **Spike:** #200
- **Time-box:** 2 days
- **Date:** 2026-08-01

## Verdict

**New complexity.** A values-axis re-organisation is *merge-shaped* — many
existing categories collapse into fewer, value-named ones — and a merge
decomposes into four operations of which the YNAB API can perform one and
a half:

| # | Step | Possible? |
|---|---|---|
| 1 | Create the target category / group | ✅ `createCategory`, `createCategoryGroup` |
| 2 | Re-point transactions from source to target | ⚠️ bulk update works, **but a split transaction's category cannot be changed in place** — see below |
| 3 | Move money already assigned to the source | ❌ `money_movements` is read-only |
| 4 | Retire the emptied source category | ❌ no `DELETE`; `hidden` is not writable |

This is not a coverage gap that closing more YnabSharp tickets will fix.
Steps 3 and 4 are ceilings in the YNAB API itself, and the split-transaction
restriction in step 2 is explicit in the spec.

The capability is therefore not "perform the migration." It is **perform the
automatable portion, then hand the user a checklist for the rest.** That is
still worth building — it is the difference between an afternoon of careful
clicking and a week of it — but it changes what the milestone is promising,
so it needs to be scoped that way rather than discovered halfway through.

## Recommendation

Build it as two things, sequenced:

1. **A mapping-driven re-point command.** Takes a hand-authored old → new
   mapping, creates any missing target categories and groups, and bulk
   re-points every non-split transaction. Depends on YnabSharp
   [#147](https://github.com/joshuaedwardcrowe/YnabSharp/issues/147)
   (create), [#100](https://github.com/joshuaedwardcrowe/YnabSharp/issues/100)
   and [#101](https://github.com/joshuaedwardcrowe/YnabSharp/issues/101)
   (group create/update). None are built yet — this is blocked on them.

   The mapping is the command's input, and looks roughly like this — old
   category on the left, target group and category on the right, many-to-one
   by design:

   ```csv
   old_category_group,old_category,new_category_group,new_category
   🏡 Home,Mortgage,Sanctuary (Am I Physically Safe),The Macro Fortress
   🏡 Home,Council Tax,Sanctuary (Am I Physically Safe),The Macro Fortress
   🧺 Cupboard,Homeware,Sanctuary (Am I Physically Safe),The Micro Perimeter
   💼 Working - BrightHR,Lunch Out,Side Quest (Professional Journey),Workday Nutrition
   ```

   Four rows collapsing to three targets is the merge ratio in miniature,
   and it is why steps 3 and 4 below cannot be skipped.

2. **A retirement checklist output.** For everything the API can't do:
   each source category that still holds assigned money, and each one
   that now needs hiding by hand. Generated, not written by the user.

   **No such output type exists in the CLI today.** Every command currently
   returns a table view model (see
   [concepts/0001-console-pipeline.md](../concepts/0001-console-pipeline.md)); a checklist is a
   different shape of outcome, and building one is a capability in its own
   right rather than a detail of this migration. Tracked separately on the
   Ideas board — this recommendation depends on it, and shouldn't silently
   assume it.

Splits and transfers both need deciding before either lands — see open
questions. Neither is a hard block: the web app will delete a reconciled
transaction if you insist, so the only unknown is whether the API is as
permissive. One `DELETE` against a scratch plan answers it, and it's worth
doing before slicing anything into sub-issues — if the API refuses, most of
the register is untouchable and recommendation 1 covers materially less
than it appears to.

## What was established

**The migration is merge-shaped, not rename-shaped.** A rename would be
`updateCategory` and nothing else. Consolidation is what makes it hard, and
it holds regardless of which target taxonomy is finally chosen — the shape
follows from the merge ratio, not from any particular set of names.

**Split transactions cannot be re-categorised *in place*.** The spec is
explicit: "If an existing transaction is a split, the `category_id` cannot
be changed." This matters more here than elsewhere, because this repo
already treats splittable transactions as a first-class concern (#165, #164).

**Delete-and-recreate is a legitimate workaround for splits, and a bad one
for transfers.** More survives the round-trip than the restriction
suggests: `cleared` accepts `reconciled`, and `import_id` is settable, so
reconciliation state and bank-import de-duplication both carry over, as do
amount, date, account, payee, memo, approved and flag. What does not carry
over is the transaction `id` (and every subtransaction id), so anything
holding a reference breaks.

Transfers are the exception, and the reason this needs deciding rather than
just doing. `transfer_transaction_id` is read-only — it is absent from
`SaveTransaction` — and a transfer's opposite leg is created implicitly via
a transfer payee. Deleting one leg and recreating it is therefore not a
faithful round-trip; it is a different operation that happens to leave a
similar-looking row.

**Three permanent API ceilings.** No `DELETE` for categories or groups (the
only two `delete` operations in the whole spec are on Transactions and
Scheduled Transactions); `hidden` is required on `CategoryBase` but absent
from `SaveCategory`, so it is readable and never writable; all four
`money_movements` operations are `get*`.

> *Permanent home:* YnabSharp's `docs/ynab-api-coverage.md`, under "What the
> API cannot do" — added in
> [YnabSharp#148](https://github.com/joshuaedwardcrowe/YnabSharp/pull/148).
> This file is not the only copy.

**The mapping cannot be inferred.** Nothing in the data says `🧺 Cupboard`
belongs under `Sanctuary (Am I Physically Safe)` rather than
`Community (I Am Loved)`. It is a values judgement, so it has to be authored
by hand and fed to the tool as input.
[my-financial-map](https://github.com/joshuaedwardcrowe/my-financial-map)
is the prose draft of it.

**The taxonomy is total.** Every old category maps to a value; there is no
legitimate unclassified residue. The point of a re-organisation is that the
new structure is *all* that remains, so an unmapped category is an error —
the tool should fail, list what it couldn't place, and refuse to proceed
rather than quietly leaving a category behind.

What looked at first like residue — Banking, `Get a Car`, CroweCaptured,
Home Maintenance, most of Celebrating — is mostly the merge itself: several
old categories collapsing into one new one, which reads as "disappearing"
only if you expect a one-to-one mapping. Those five still need placing, but
that is a gap in the map rather than evidence against totality
([my-financial-map#1](https://github.com/joshuaedwardcrowe/my-financial-map/issues/1)).

**Previewing is possible, but the scratch plan is made by hand.** YNAB
allows more than one plan, and the CLI can target any of them by id. But
there is no create-plan operation in the API (`getPlans`, `getPlanById`,
`getPlanSettingsById` only), so the throwaway copy has to be created in the
YNAB web app first. Cheap, and worth doing — just not automatable end to end.

## Evidence

Against the vendored spec, `docs/ynab-openapi-spec.yaml` at version
**1.86.0** in YnabSharp (last checked 2026-07-26):

```
# category operations — eight, not the seven the coverage doc listed
grep -n "operationId:" docs/ynab-openapi-spec.yaml | grep -i categor

# every delete verb in the spec — two, both on transactions
grep -n "^\s*delete:" docs/ynab-openapi-spec.yaml

# money movements — all four are get*
grep -n "operationId:" docs/ynab-openapi-spec.yaml | grep -i money

# hidden: present on the read schemas, absent from SaveCategory
grep -n "hidden" docs/ynab-openapi-spec.yaml
sed -n '/^    SaveCategory:/,/^    SaveMonthCategory:/p' docs/ynab-openapi-spec.yaml
```

The split-transaction restriction is in the `category_id` description on
`SaveTransactionWithOptionalFields`.

The under-reported counts this turned up were corrected in
[YnabSharp#148](https://github.com/joshuaedwardcrowe/YnabSharp/pull/148),
and the missing create endpoint raised as
[YnabSharp#147](https://github.com/joshuaedwardcrowe/YnabSharp/issues/147).

## Open questions

- **Do we delete-and-recreate splits, or leave and report them?** The
  round-trip is faithful apart from the `id` (above), so this is a judgement
  about whether losing transaction identity is acceptable, not a technical
  blocker. Recreating is the only way to get splits onto the new axis at all.
- **Does the *API* refuse to delete a reconciled transaction?** The YNAB web
  app allows it, with a prominent warning — so the data model permits it and
  this is not a hard block. What is unknown is whether the API mirrors the
  UI's permissiveness or refuses outright, since the spec says nothing
  either way. One `DELETE` against a scratch plan settles it. Worth doing
  before breakdown, because it decides the question above for most of the
  register.
- **What happens to transfers?** Delete-and-recreate is unsafe for them
  (above), so they need either an in-place path, an explicit skip, or a
  manual step. Unlike splits, there is no workaround to choose between.
- **How much assigned money actually needs moving?** Step 3 is manual, so
  its cost is proportional to how many source categories hold a non-zero
  balance at migration time. Migrating just after a month rolls over may
  make it nearly free. Unmeasured.

## Out of scope

The taxonomy itself — which values exist and what belongs under them — is a
personal decision, not a technical one, and was not evaluated here.

`SPENDFULNESS.md`'s §3 Category-Group-to-Behaviour matrix looks overtaken by
this change of axis, but sense-checking it against YNAB's own material is
its own piece of work and is tracked separately.

Whether the live plan should ever be written to directly, versus always via
a previewed scratch plan, is a design decision for the delivery ticket.
