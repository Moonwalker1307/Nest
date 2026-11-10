# C# 15 language features

C# 15 ships with the .NET 11 SDK and is the default for `net11.0`. Every feature here works without `<LangVersion>preview</LangVersion>`, except memory safety.

Sources: https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15 and https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/union

## Contents

- Union types
- Closed hierarchies
- Collection expression arguments
- Extension indexers
- Labeled break and continue
- Non-virtual static interface members on older runtimes
- Memory safety (preview)

## Union types

A union holds a value that is one of a fixed set of case types.

```csharp
public record class Cat(string Name);
public record class Dog(string Name);
public record class Bird(string Name);

public union Pet(Cat, Dog, Bird);

Pet pet = new Dog("Rex");          // implicit conversion from each case type

string name = pet switch           // exhaustive: no default arm needed
{
    Dog d => d.Name,
    Cat c => c.Name,
    Bird b => b.Name,
};
```

Generic unions and inline case types work:

```csharp
public record class None;
public record class Some<T>(T Value);
public union Option<T>(None, Some<T>);

public union IntOrString(int, string);
```

Case types can be classes, structs, interfaces, type parameters, nullable types, or other unions.

### What the compiler generates

- A **struct** marked `[System.Runtime.CompilerServices.Union]` that implements `IUnion` (both types ship in the .NET 11 runtime).
- One public constructor per case type.
- A `Value` property of type `object?`. **Value-type cases are boxed.**

### Members in a union body

A union can have a body like a struct, with these limits:

- No instance fields, auto-properties or field-like events.
- No public single-parameter constructors (the compiler generates them).
- Any constructor you write must chain with `this(...)` to a generated one.

```csharp
public record class Meters(double Value);
public record class Feet(double Value);

public union Length(Meters, Feet)
{
    public double TotalMeters => this switch
    {
        Meters m => m.Value,
        Feet f => f.Value * 0.3048,
        null => throw new InvalidOperationException("The Length has no value."),
    };

    public Length Add(Length other) => new Meters(TotalMeters + other.TotalMeters);
}
```

### Conversions

- Each case type converts implicitly to the union. A user-defined implicit conversion for the same type wins.
- If two case types are equally good for a value, the conversion is ambiguous and is a compile error. Construct explicitly in that case.
- `Pet? maybe = new Dog("Rex");` and `Pet? none = null;` both work.

### Pattern matching

Type patterns look at the **contents** (`Value`), not the union itself.

| Pattern | Applies to |
| --- | --- |
| `Dog`, `Dog d`, `Dog { Breed: "Lab" }` | the contents |
| `{ Name: "Rex" }` (no type) | the union's own properties |
| list patterns | the union's own count and indexer |
| `var x`, `_` | the union itself, not unwrapped |
| constants, relational, `null` | the contents (`null` also matches a null nullable wrapper) |

- `pet is Pet` is error CS8121. Use `pet is not null` or `pet.Value is not null`.
- `{ }` on a non-nullable union struct always matches, even when `Value` is null. Use `not null` to test the contents.

### Null and exhaustiveness

- `default(Pet)` holds `null`. That includes uninitialized fields and array elements. A `switch` without a `null` arm then throws `SwitchExpressionException` at run time, and the compiler may not warn.
- Prefer a `null` arm over `_` or `var`. A `null` arm keeps exhaustiveness warnings when a case type is added later. `_` hides them.
- If a case type is nullable (`union NullablePet(Cat?, Dog, Bird)`), a `switch` without a `null` arm gives CS8655.

### Custom unions (no boxing, or class-based)

Any class or struct with `[Union]` that follows this pattern is a union to the compiler:

- `[System.Runtime.CompilerServices.Union]` attribute.
- One or more public constructors with exactly one parameter (by value or `in`). Each parameter type is a case type.
- A public `object? Value { get; }`.

Optional, to avoid boxing during pattern matching:

- `public bool HasValue { get; }` (used for `is null` patterns).
- `public bool TryGetValue(out T value)` for each case type (used for `is T` patterns).

```csharp
[System.Runtime.CompilerServices.Union]
public struct IntOrBool : System.Runtime.CompilerServices.IUnion
{
    private readonly int _intValue;
    private readonly bool _boolValue;
    private readonly byte _tag; // 0 = none, 1 = int, 2 = bool

    public IntOrBool(int? value)
    {
        if (value.HasValue) { _intValue = value.Value; _tag = 1; }
    }

    public IntOrBool(bool? value)
    {
        if (value.HasValue) { _boolValue = value.Value; _tag = 2; }
    }

    public object? Value => _tag switch { 1 => _intValue, 2 => _boolValue, _ => null };

    public bool HasValue => _tag != 0;

    public bool TryGetValue(out int value) { value = _intValue; return _tag == 1; }
    public bool TryGetValue(out bool value) { value = _boolValue; return _tag == 2; }
}
```

A class can be a union too, for reference semantics:

```csharp
[System.Runtime.CompilerServices.Union]
public class Result<T> : System.Runtime.CompilerServices.IUnion
{
    private readonly object? _value;
    public Result(T? value) { _value = value; }
    public Result(Exception? value) { _value = value; }
    public object? Value => _value;
}
```

When the union needs private constructors or factories, put the union members on a public nested `IUnionMembers` interface that the type implements, with static `Create(...)` methods per case type and a `Value` property.

The compiler trusts that `Value` is always `null` or one of the case types, and that `HasValue`/`TryGetValue` agree with `Value`. Keep custom unions consistent with that.

### Serialization

System.Text.Json serializes unions as their contents. Use `[JsonUnion]` to tell cases apart that have the same JSON shape. See [libraries.md](libraries.md). ASP.NET Core supports unions in JSON bodies and SignalR's JSON protocol, but not in route, query, header or form binding, and not in the MessagePack or Newtonsoft.Json SignalR protocols.

## Closed hierarchies

`closed` on a class limits direct derived types to the same assembly, so the compiler knows every case.

```csharp
public closed record class GateState;
public record class Closed : GateState;
public record class Open(float Percent) : GateState;

string Describe(GateState state) => state switch
{
    Closed => "closed",
    Open(var percent) => $"{percent}% open",
    // no default arm needed
};
```

- A `closed` class is implicitly `abstract`. Do not add `abstract`, `sealed` or `static`.
- It is not transitive. A derived class that is not itself `closed` can be derived from in other assemblies. Mark intermediate classes `closed` to extend exhaustiveness further down.
- Pick a union when the cases are unrelated existing types. Pick a closed hierarchy when you own the cases and want shared members on a base type.

## Collection expression arguments

A `with(...)` element, first in the collection expression, passes arguments to the constructor or factory.

```csharp
string[] values = ["one", "two", "three"];

List<string> names = [with(capacity: values.Length * 2), .. values];

HashSet<string> set = [with(StringComparer.OrdinalIgnoreCase), "Hello", "HELLO", "hello"];
// set has one element
```

`with(...)` must be the first element.

## Extension indexers

Indexers can be declared in an `extension` block (extension blocks arrived in C# 14). The block needs a named receiver, because indexers are instance members.

```csharp
public static class SequenceIndexer
{
    extension(IEnumerable<int> sequence)
    {
        public int this[int index] => sequence.ElementAt(index);
    }
}

int third = Enumerable.Range(1, 10)[2];
```

## Labeled break and continue

`break` and `continue` can name a label on an enclosing loop or `switch` statement. Put the label directly on that statement. Unlabeled `break`/`continue` work as before.

```csharp
outer: for (int row = 0; row < grid.Height; row++)
{
    for (int column = 0; column < grid.Width; column++)
    {
        if (grid[row, column].IsBlocked) continue outer;
        if (grid[row, column].IsGoal) break outer;
    }
}
```

Use this instead of Boolean flags or `goto`. The IDE0410 style rule points out those patterns.

## Non-virtual static interface members on older runtimes

Non-virtual static members in interfaces now compile even when the target runtime lacks default interface implementation support (for example .NET Framework).

```csharp
public interface IMeasurement
{
    public static double ToMeters(double centimeters) => centimeters / 100;
}
```

Instance default implementations, `static abstract`/`static virtual` members and protected interface members still need runtime support.

## Memory safety (preview, do not use unless asked)

This starts a multi-release change: `unsafe` becomes about operations that touch unmanaged memory, not about pointer types existing.

Turn it on with `<LangVersion>preview</LangVersion>` plus the `updated-memory-safety-rules` compiler feature flag (see https://learn.microsoft.com/dotnet/csharp/language-reference/compiler-options/language#enable-the-updated-memory-safety-rules).

With it on:

- `unsafe` on a member makes it *requires-unsafe*: callers must be in an `unsafe` context. The compiler records `MemorySafetyRulesAttribute`.
- A new `safe` contextual keyword marks `extern` members, and fields in explicit or extended layout types, as safe.
- No `unsafe` context needed for: declaring pointer types, `&x`, `fixed`, converting `stackalloc` to a pointer, `sizeof` on any unmanaged type.
- Still needs `unsafe`: `*p`, `p->member`, `p[i]`, calling function pointers.
- `unsafe(expression)` gives one expression an unsafe context, for example in a field initializer. Needs `AllowUnsafeBlocks` too.

```csharp
static readonly int Signature = unsafe(ReadSignature());
```
