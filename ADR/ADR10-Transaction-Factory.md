# Transaction Factory

## Premise

When working with YNAB transaction data, different types of transactions may require specialized behavior, additional properties, or custom business logic beyond the standard transaction model. For example, taxi transactions might need to parse location information from the memo field, or recurring subscriptions might need to track renewal patterns.

The challenge is providing a flexible way to create custom transaction types while maintaining a consistent interface for transaction retrieval and querying across the codebase.

## Problem

Without a structured approach to creating specialized transactions, the codebase would suffer from:

1. **Rigid Transaction Model** - All transactions forced into a single class regardless of their specific needs
2. **Conditional Logic Scatter** - Type-checking and custom behavior logic spread across multiple clients and command handlers
3. **Tight Coupling** - Direct dependencies on specific transaction types throughout the codebase
4. **Limited Extensibility** - Adding new transaction types requires modifying existing clients and handlers
5. **Inconsistent Creation** - Different parts of the codebase creating transactions in different ways

## Solution

The Transaction Factory pattern introduces a flexible, extensible system for creating specialized transaction objects from YNAB API responses. The pattern uses a chain of responsibility approach where multiple factories can be registered, each capable of handling specific transaction types.

### 1. Factory Interface

The `ITransactionFactory` interface defines a simple contract for transaction creation:

```csharp
public interface ITransactionFactory
{
    bool CanWorkWith(TransactionResponse transactionResponse);
    Transaction Create(TransactionResponse transactionResponse);
}
```

**Key Methods:**
- **CanWorkWith()**: Determines if this factory can handle the given transaction response (query capability)
- **Create()**: Creates the appropriate transaction object from the response

### 2. Default Factory Implementation

The `DefaultTransactionFactory` serves as a fallback factory that handles all standard transactions:

```csharp
public class DefaultTransactionFactory : ITransactionFactory
{
    public bool CanWorkWith(TransactionResponse transactionResponse)
        => true;

    public Transaction Create(TransactionResponse transactionResponse)
        => new(transactionResponse);
}
```

**Characteristics:**
- Always returns `true` for `CanWorkWith()` (universal fallback)
- Creates standard `Transaction` objects
- Automatically registered as the default factory

### 3. Custom Factory Implementation

Custom factories can be created for specialized transaction types. Example with `TaxiTransactionFactory`:

```csharp
public class TaxiTransactionFactory : ITransactionFactory
{
    private Regex _taxiRegex = new Regex(@"\b(.+?)\s+to\s+(.+?)\b", RegexOptions.IgnoreCase);

    public bool CanWorkWith(TransactionResponse transactionResponse)
        => transactionResponse.Memo != null && _taxiRegex.IsMatch(transactionResponse.Memo);

    public Transaction Create(TransactionResponse transactionResponse)
        => new TaxiTransaction(transactionResponse);
}
```

**Pattern Features:**
- Custom logic to identify applicable transactions (memo pattern matching)
- Creates specialized transaction subclasses
- Can use any criteria (memo content, amount ranges, payee names, etc.)

### 4. Dependency Injection Registration

Factories are registered with dependency injection in priority order:

```csharp
var serviceProvider = new ServiceCollection()
    .AddYnab() // Registers DefaultTransactionFactory
    .AddYnabTransactionFactory<TaxiTransactionFactory>() // Registers custom factory
    .BuildServiceProvider();
```

**Registration Mechanism:**
- `AddYnab()` registers the default factory as a singleton
- `AddYnabTransactionFactory<T>()` registers custom factories with priority
- Custom factories are placed **before** the default factory in the DI container
- Ensures custom factories are queried before the default fallback

### 5. Factory Selection and Usage

Transaction clients inject all registered factories and use them via a selection process:

```csharp
public class TransactionClient : YnabApiClient
{
    private readonly IEnumerable<ITransactionFactory> _transactionFactories;

    private Transaction CreateTransaction(TransactionResponse transactionResponse)
    {
        var factory = _transactionFactories.FirstOrDefault(s => s.CanWorkWith(transactionResponse));
        return factory.Create(transactionResponse);
    }
}
```

**Selection Logic:**
1. Iterate through factories in registration order
2. Call `CanWorkWith()` on each factory
3. Use the **first** factory that returns `true`
4. Default factory catches any unmatched transactions

### 6. Integration Points

The Transaction Factory pattern is integrated into multiple YNAB API clients:

- **TransactionClient**: Creates transactions when fetching transaction data
- **AccountClient**: Uses factories when retrieving account transactions
- **BudgetsClient**: Passes factories to connected budget instances
- **SpendfulnessBudgetClient**: Uses factories for local transaction processing

All clients receive the same collection of factories via constructor injection, ensuring consistent transaction creation throughout the application.

## Constraints

### Factory Order Dependency
- Custom factories must be registered **before** using the API clients
- Registration order determines factory priority
- The default factory should always be last to serve as a catch-all
- Multiple custom factories are evaluated in registration order

### Query Pattern Requirements
- `CanWorkWith()` must be deterministic and side-effect free
- Factory selection happens for every transaction retrieval
- Performance-sensitive queries should be optimized
- Avoid expensive operations in `CanWorkWith()` (e.g., API calls, heavy regex)

### Type Safety and Extensibility
- Custom transaction types must inherit from `Transaction` base class
- Factories are decoupled from clients through the interface
- New transaction types require both a factory and a transaction class
- Breaking changes to `TransactionResponse` affect all factories

### Null Handling
- Currently, the factory collection is assumed to always contain at least the default factory
- The code uses `FirstOrDefault()` without null checking, relying on the default factory always being present
- A null check is recommended in production code (warning CS8602 exists in TransactionClient.cs:49)
- Empty factory collections would cause runtime exceptions
- Consider adding defensive null checks: `var factory = _transactionFactories.FirstOrDefault(s => s.CanWorkWith(transactionResponse)) ?? throw new InvalidOperationException("No factory found");`

## Questions & Answers

### Why use FirstOrDefault instead of more sophisticated pattern matching?

The simple linear search using `FirstOrDefault` is sufficient for the expected number of factories (typically < 10). The pattern prioritizes simplicity and readability over premature optimization. If factory collections grow significantly, consider implementing a more sophisticated selection strategy.

### How do you handle transactions that match multiple factories?

The first matching factory wins. This is intentional - factory registration order determines priority. If a transaction could match multiple patterns, register the more specific factory first. For example, register a "Premium Taxi Transaction" factory before the general "Taxi Transaction" factory.

### Should factories perform validation or data transformation?

Factories should focus on object creation and type selection. Complex validation or data transformation is better handled in the transaction objects themselves or in separate validation services. The factory's role is to determine the appropriate type and instantiate it.

### How does this integrate with the Connected API concept?

The Transaction Factory pattern works seamlessly with Connected API objects. Factories receive `TransactionResponse` DTOs from the API and create domain objects (`Transaction` or subclasses). Connected objects then wrap these domain objects with operational capabilities. The factory pattern operates at the domain layer, while Connected objects operate at the service layer.

### What about caching factory selection results?

Currently, factory selection happens for each transaction independently. If performance becomes an issue with large transaction sets, consider caching selection results based on transaction characteristics (e.g., payee name patterns). However, this adds complexity and should only be implemented if profiling indicates it's necessary.

### Can factories be async?

The current interface is synchronous. If async factory logic is needed (e.g., calling external APIs to determine transaction type), consider creating an async variant of the interface or performing async initialization during factory construction rather than during transaction creation.

### How do you test custom factories?

Custom factories can be unit tested by:
1. Creating mock `TransactionResponse` objects with specific characteristics
2. Testing `CanWorkWith()` with various response patterns
3. Verifying `Create()` returns the correct transaction type
4. Testing edge cases (null memos, empty fields, etc.)

The factory pattern's simple interface makes it highly testable in isolation.

### Should all specialized transaction logic use factories?

Factories are best suited for scenarios where different transaction types need fundamentally different classes or behavior. If the difference is only data-level (e.g., filtering or categorization), consider using standard queries or specifications instead. Use factories when polymorphism provides clear value.
