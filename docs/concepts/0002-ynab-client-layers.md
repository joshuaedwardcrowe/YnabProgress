# 0002. YNAB client layers

How this repo reaches the YNAB API, and where the seam sits.

**The client itself is no longer in this repo.** It used to be — a
top-level `Ynab/` project — until it was extracted and replaced by the
[YnabSharp](https://github.com/joshuaedwardcrowe/YnabSharp) NuGet package
(commit `298a939`, "Swapped Out Ynab Library"). The layers below are
YnabSharp's namespaces, described only as far as SpendfulnessCli touches
them. For how they work internally, read YnabSharp's own `docs/concepts/`.

## What SpendfulnessCli consumes

| Namespace | What this repo uses it for |
| :--- | :--- |
| `YnabSharp.Http` | `YnabHttpClientBuilder` — builds an authenticated client from an API key. |
| `YnabSharp.Clients` | Direct, per-resource communication with the YNAB API. |
| `YnabSharp.Connected` | `ConnectedBudget` — a budget whose accounts, categories and transactions are reachable as methods, so callers never manage clients themselves. |
| `YnabSharp.Factories` | `ITransactionFactory` — how a transaction is constructed, including the split cases. |
| `YnabSharp` | The domain types (`CategoryGroup`, `Transaction`, …) that aggregators consume. |
| `YnabSharp.Extensions` | Query extensions such as `FilterToSpending()` and `FilterToOutflow()`. |

## SpendfulnessBudgetClient — the seam

`Spendfulness.Database.Sqlite.SpendfulnessBudgetClient` is the only place
the rest of the CLI acquires a budget. It resolves the active user from the
local SQLite store, reads that user's YNAB API key, and returns a
`ConnectedBudget`:

```csharp
var budget = await spendfulnessBudgetClient.GetDefaultBudget();
var transactions = await budget.GetTransactions();
```

Command handlers depend on `SpendfulnessBudgetClient`, never on
`YnabHttpClientBuilder` or a raw client. That keeps credential lookup and
user resolution in one place, and means a handler never has to know an API
key exists. The decision is recorded in
[ADR 0009](../adr/0009-spendfulness-budget-client.md); the surface it wraps
is [ADR 0005](../adr/0005-connected-api-concept.md).

It throws `SpendfulnessDbException` with
`SpendfulnessDbExceptionCode.CannotConfigureBudget` when there is no active
user, or that user has no API key — so "not set up yet" is a typed failure
rather than a null reference further down.
