# C# 14 language features

C# 14 ships with .NET 10 and is the default for `net10.0`.

Sources: https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-14 and https://learn.microsoft.com/dotnet/csharp/whats-new/breaking-changes/compiler%20breaking%20changes%20-%20dotnet%2010

## Contents

- Extension members
- The field keyword
- Null-conditional assignment
- Implicit span conversions
- Modifiers on simple lambda parameters
- Partial constructors and events
- User-defined compound assignment and increment operators
- nameof with unbound generics
- Breaking changes

## Extension members

An `extension` block inside a static class declares extension members. A block with a named receiver gives instance-style members. A block with only a type gives static-style members on that type.

```csharp
public static class EnumerableExtensions
{
    extension<TSource>(IEnumerable<TSource> source)
    {
        public bool IsEmpty => !source.Any();                     // extension property

        public IEnumerable<TSource> WhereNot(Func<TSource, bool> predicate)
            => source.Where(x => !predicate(x));                  // extension method
    }

    extension<TSource>(IEnumerable<TSource>)
    {
        public static IEnumerable<TSource> Identity => Enumerable.Empty<TSource>();

        public static IEnumerable<TSource> operator +(IEnumerable<TSource> left, IEnumerable<TSource> right)
            => left.Concat(right);
    }
}

bool empty = items.IsEmpty;
var none = IEnumerable<int>.Identity;
var all = first + second;
```

- Extension properties, static extension members and extension operators are new. Classic `this` parameter extension methods still work and can live beside blocks.
- Indexers in extension blocks only arrive in C# 15 (.NET 11).

## The field keyword

`field` inside a property accessor is the compiler-generated backing field.

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}

public SqlConnection Connection => field ??= new SqlConnection(connectionString);

public string DisplayName
{
    get;
    set
    {
        if (field == value) return;
        field = value;
        OnPropertyChanged();
    }
}
```

You can give a body to one accessor and leave the other as `get;` or `set;`.

## Null-conditional assignment

`?.` and `?[]` can appear on the left of `=` and compound assignments.

```csharp
customer?.Order = GetCurrentOrder();   // GetCurrentOrder() runs only when customer is not null
customer?.Total += amount;
values?[0] = 1;
```

`++` and `--` are not allowed with it.

## Implicit span conversions

`T[]`, `Span<T>` and `ReadOnlySpan<T>` now convert implicitly to each other where safe (array to `Span<T>` or `ReadOnlySpan<T>`, `Span<T>` to `ReadOnlySpan<T>`, `string` to `ReadOnlySpan<char>`). Spans can be extension method receivers and take part in generic inference. Span-based overloads are therefore picked more often, which is usually faster. Write APIs that take `ReadOnlySpan<T>` and callers can pass arrays directly.

## Modifiers on simple lambda parameters

```csharp
delegate bool TryParse<T>(string text, out T result);

TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

`scoped`, `ref`, `in`, `out` and `ref readonly` work without types. `params` still needs explicit types.

## Partial constructors and events

Instance constructors and events can be `partial` (useful for source generators).

- Exactly one defining and one implementing declaration.
- Only the implementing constructor can have `: this(...)` or `: base(...)`.
- The implementing event declaration must have `add` and `remove` accessors; the defining one is field-like.
- Only one partial type declaration can have the primary constructor.

## User-defined compound assignment and increment operators

Types can define `+=`, `-=` and similar operators, and `++`/`--`, as instance operators that change the object in place, which avoids creating a new instance. When both exist, the compound operator is preferred over `x = x + y`. See the "user-defined compound assignment" feature spec for the exact signatures before writing one.

## nameof with unbound generics

```csharp
nameof(List<>)   // "List"
```

## Breaking changes

- `field` in an accessor now means the backing field. A member named `field` gives warning CS9258 (use `this.field` or `@field`). A local named `field` in an accessor is error CS9272.
- `extension` is a contextual keyword. Use `@extension` for identifiers with that name.
- `scoped` in a lambda parameter list is always a modifier. Escape a type named `scoped` as `@scoped`.
- `partial` cannot be a method return type without `@partial`.
- Span overloads apply in more places. Calls can become ambiguous (`Assert.Equal([2], longArray)`) or pick a span overload. Fix with `.AsSpan()` or an explicit call. With C# 14 on an older target, `array.Reverse()` binds to the in-place `MemoryExtensions.Reverse` and returns `void`; call `Enumerable.Reverse(array)`.
- `o is not null or 42` gets a warning, since `not` binds tighter than `or`. Write `o is not (null or 42)` if that was the intent.
- Partial properties and events in interfaces are implicitly `virtual` and `public`, like non-partial members.
