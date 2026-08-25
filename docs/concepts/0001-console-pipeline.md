# 0001. The console pipeline

What happens between typing a command and seeing a table — the whole path,
in one place.

```
you type an instruction
        │   KitCli parses it into a CliInstruction
        ▼
   CliCommandFactory ──► CliCommand            (a record of typed arguments)
        │
        ▼
   CliCommandHandler                           (orchestrates, calculates nothing)
        │  SpendfulnessBudgetClient ──► ConnectedBudget
        │  Aggregator ──────────────► Aggregate records
        │  CliTableBuilder ─────────► ViewModel
        ▼
   CliCommandOutcome[]
        │   SpendfulnessCliApp.OnRunComplete: report, persist
        ▼
   rendered to the console by KitCli
```

Almost none of the machinery is in this repo. `CliApp`, `ICliWorkflow`,
`CliCommand`, the outcome types and the IO all come from
[KitCli](https://github.com/KitCli/KitCli). What lives here is the
Spendfulness-specific filling: which commands exist, how YNAB data is
reduced, and how it is shaped into a table.

## Wiring it up

`Program.cs` is registration only — settings, services, commands — and then
hands control to KitCli:

```csharp
var cliAppBuilder = new CliAppBuilder().WithCli<SpendfulnessCliApp>();
// ... settings, AddYnab(), AddSpendfulnessSqliteDb(), AddSpendfulness*Commands()
await cliAppBuilder.Run();
```

Each command area registers itself (`AddSpendfulnessReportingCommands()`
and friends), so adding a command means touching one project rather than a
central list.

## The command triple

Every command is three small types in one folder, named after the command:

```
SpendfulnessCli.Commands.Reporting/AverageYearlySpending/
├── AverageYearlySpendingCliCommand.cs         the request
├── AverageYearlySpendingCliCommandFactory.cs  instruction → request
└── AverageYearlySpendingCliCommandHandler.cs  request → outcomes
```

**Command** — a record deriving from `CliCommand`, carrying the arguments
the handler needs and nothing else. One taking no arguments is legitimately
empty:

```csharp
public record AverageYearlySpendingCliCommand : CliCommand;
```

**Factory** — implements `ICliCommandFactory<TCommand>`. Turns the parsed
`CliInstruction` and accumulated `CliCommandArtefact`s into that record.
Raw argument text becomes typed values here, so the handler never parses
anything.

**Handler** — derives from `CliCommandHandler`. It orchestrates the middle
of the pipeline and calculates nothing itself:

```csharp
var budget = await spendfulnessBudgetClient.GetDefaultBudget();
var transactions = await budget.GetTransactions();

var aggregator = new TransactionAverageAcrossYearYnabListAggregator(transactions)
    .BeforeAggregation(y => y.FilterToSpending())
    .BeforeAggregation(y => y.FilterToOutflow());

var viewModel = new TransactionYearAverageCliTableBuilder()
    .WithAggregator(aggregator)
    .Build();

return OutcomeAs(viewModel);
```

The budget always arrives through `SpendfulnessBudgetClient`, never a raw
client — see [YNAB client layers](0002-ynab-client-layers.md).

## Reducing: aggregation knows nothing about tables

This is the one structural decision worth remembering.
`Spendfulness.Aggregation` reduces YNAB objects to plain records;
`SpendfulnessCli.CliTables` turns records into columns and rows. Neither
knows about the other, which is why the same aggregate can feed a table, a
CSV export, or a chat response without change.

**Aggregate** — a record describing one reduced row. No formatting, no
column names:

```csharp
public record CategoryAggregate(Guid CategoryId, string CategoryName);
```

**Aggregator** — two levels, and the split is the point:

| | Where | What it owns |
| :--- | :--- | :--- |
| `CliListAggregator<TAggregate>` | **KitCli** | The generic contract — `ListAggregate()`, paging. Domain-agnostic. |
| `YnabListAggregator<TAggregate>` | here | The YNAB-shaped source (`Accounts`, `CategoryGroups`, `Transactions`, `Commitments`) and the `BeforeAggregation` hooks. |

Concrete aggregators implement `GenerateAggregate()`:

```csharp
public class CategoryYnabListAggregator(IEnumerable<CategoryGroup> categoryGroups)
    : YnabListAggregator<CategoryAggregate>(categoryGroups)
{
    protected override IEnumerable<CategoryAggregate> GenerateAggregate()
        => CategoryGroups
            .SelectMany(categoryGroup => categoryGroup.Categories)
            .Select(category => new CategoryAggregate(category.Id, category.Name));
}
```

Filtering is composed by the caller through `BeforeAggregation` rather than
baked into the aggregator, so one aggregator serves many commands.

## Shaping: view models and builders

**ViewModel** — owns the column *vocabulary*, as constants plus their
order, so a heading can be referenced by name without a string literal
drifting out of sync with what is printed:

```csharp
public class CategoryViewModel
{
    public const string CategoryName = "Category Name";
    public const string CategoryId = "Category Id";

    public static List<string> GetColumnNames() => [CategoryName, CategoryId];
}
```

**ViewModelBuilder** — derives from `CliTableBuilder<TAggregate>` and maps
aggregates onto that vocabulary:

```csharp
public class CategoryCliTableBuilder : CliTableBuilder<CategoryAggregate>
{
    protected override List<string> BuildColumnNames(IEnumerable<CategoryAggregate> evaluation)
        => CategoryViewModel.GetColumnNames();

    protected override List<List<object>> BuildRows(IEnumerable<CategoryAggregate> aggregates)
        => aggregates.Select(a => new List<object> { a.CategoryName, a.CategoryId }).ToList();
}
```

`CliTableBuilder.WithAggregator` accepts KitCli's `CliListAggregator<T>`,
not `YnabListAggregator` — the table layer genuinely doesn't know the data
came from YNAB.

## Finishing the run

A handler returns `CliCommandOutcome[]` rather than writing to the console,
so rendering stays KitCli's job. `SpendfulnessCliApp` hooks the run
lifecycle to report progress and, importantly, to **persist**:

```csharp
protected override void OnRunComplete(ICliWorkflowRun run, CliCommandOutcome[] outcomes)
{
    // ... report outcome type names and the run's state timeline
    _spendfulnessDbContext.SaveChanges();
}
```

Worth knowing: database changes are saved once at the end of the run, not
inside the handler. A handler that mutates tracked entities does not need
to save them, and two handlers cannot commit independently mid-run.

## Adding a report command

The pipeline in practice — four small files:

1. An `Aggregate` record for the row shape.
2. A `YnabListAggregator<TAggregate>` implementing `GenerateAggregate()`.
3. A `ViewModel` for the column names, and a `CliTableBuilder<TAggregate>`.
4. The command triple, registered in the area's `ServiceCollectionExtensions`.
