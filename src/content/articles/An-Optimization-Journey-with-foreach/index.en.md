---
title: "An Optimization Journey with `foreach`"
date: "2026-06-30"
lastmod: "2026-06-30"
draft: false
summary: "A deep dive into optimizing `foreach`: designing a zero-allocation, generic traversal solution that delivers a 10× performance boost with zero memory allocation on .NET 10—all while keeping the code elegant."
slug: "An-Optimization-Journey-with-foreach"
tags: ["dotnet"]
series: ["performance"]
---

## Starting with a Real-World Requirement

Many of us have encountered the need to generate tables—whether exporting data to Excel, rendering it into PDFs, or similar tasks. When such requirements appear across multiple parts of a project, it’s natural to encapsulate the table-generation logic. The core flow is straightforward:

1. Model table rows as `TRow`.
2. Declare a mapping from `TRow` to column data (typically strings).
3. Iterate over all rows and execute write or render operations for each cell.

Here’s some pseudocode illustrating the idea:

```csharp
public sealed class TableGenerator<TRow>(IReadOnlyList<TRow> Rows) where TRow : ITableRow
{
    public void Generate(ICellWriter writer)
    {
        foreach (var row in Rows)
        {
            foreach (var cellText in row.Columns())
            {
                // Do something...
                writer.Write(cellText);
            }
        }
    }
}

public interface ICellWriter
{
    void Write(string text);
}
```

Let’s assume we’re generating a simple player table, with the row definition below:

```csharp
public sealed class PlayerRow : ITableRow
{
    public required string Id { get; init; }

    public required string Name { get; init; }
}
```

### Two Naive Implementations

A natural first approach is to use an array:

```csharp
public IEnumerable<string> Columns() => [Id, Name];
```

The problem: every time we iterate over a row, we allocate a `string[]`. The heap memory consumed by the array includes its object header plus references to the string elements, resulting in **O(N)** space complexity, where N is the number of columns. With many columns, this leads to significant allocations.

We might then turn to an iterator, which alleviates the allocation issue:

```csharp
public IEnumerable<string> Columns()
{
    yield return Id;
    yield return Name;
}
```

The compiler transforms `yield return` into an iterator class implemented as a state machine. Here’s roughly what the generated code looks like (subject to change across compiler versions):

```csharp
[CompilerGenerated]
private sealed class <Columns>d__8 : IEnumerable<string>, IEnumerable, IEnumerator<string>, IEnumerator, IDisposable
{
    private int <>1__state;
    private string <>2__current;
    private int <>l__initialThreadId;

    [Nullable(0)]
    public PlayerRow <>4__this;

    string IEnumerator<string>.Current
    {
        [DebuggerHidden]
        get => <>2__current;
    }

    object IEnumerator.Current
    {
        [DebuggerHidden]
        [return: Nullable(0)]
        get => <>2__current;
    }

    [DebuggerHidden]
    public <Columns>d__8(int <>1__state)
    {
        this.<>1__state = <>1__state;
        <>l__initialThreadId = Environment.CurrentManagedThreadId;
    }

    [DebuggerHidden]
    void IDisposable.Dispose()
    {
        <>1__state = -2;
    }

    private bool MoveNext()
    {
        switch (<>1__state)
        {
            default:
                return false;
            case 0:
                <>1__state = -1;
                <>2__current = <>4__this.Id;
                <>1__state = 1;
                return true;
            case 1:
                <>1__state = -1;
                <>2__current = <>4__this.Name;
                <>1__state = 2;
                return true;
            case 2:
                <>1__state = -1;
                return false;
        }
    }

    bool IEnumerator.MoveNext() => this.MoveNext();

    [DebuggerHidden]
    void IEnumerator.Reset() => throw new NotSupportedException();

    [DebuggerHidden]
    [return: Nullable(new byte[] { 1, 0 })]
    IEnumerator<string> IEnumerable<string>.GetEnumerator()
    {
        <Columns>d__8 iterator;
        if (<>1__state == -2 && <>l__initialThreadId == Environment.CurrentManagedThreadId)
        {
            <>1__state = 0;
            iterator = this;
        }
        else
        {
            iterator = new <Columns>d__8(0);
            iterator.<>4__this = <>4__this;
        }
        return iterator;
    }

    [DebuggerHidden]
    IEnumerator IEnumerable.GetEnumerator() => ((IEnumerable<string>)this).GetEnumerator();
}
```

The original method compiles down to:

```csharp
[IteratorStateMachine(typeof(<Columns>d__8))]
public IEnumerable<string> Columns()
{
    var iterator = new <Columns>d__8(-2);
    iterator.<>4__this = this;
    return iterator;
}
```

With this approach, iterating over each row allocates a `<Columns>d__8` iterator instance on the heap. However, its memory footprint is constant—only the object header and a few fields—so space complexity drops to **O(1)**, independent of column count.

Both approaches allocate memory per row, which will likely be collected by GC in Gen 0. For cold paths, this is acceptable. But what if this code sits on a hot path?

### Struct Enumerators: A Performance Leap

On hot paths, even small optimizations compound dramatically due to high execution frequency.

Although the iterator reduces space complexity relative to column count, it still scales linearly with the number of rows. By default, compilers emit iterators as reference types for lifetime safety. Could an iterator be a value type instead? If so, and the iterator were a `struct`, allocations would move off the managed heap entirely—allocated on the stack and reclaimed automatically. That’s a serious performance win.

The answer is yes. Consider `List<T>`, one of the most heavily used collection types in .NET. Its `foreach` enumeration uses a `struct` enumerator:

```csharp
public struct Enumerator : IEnumerator<T>, IEnumerator
{
    private readonly List<T> _list;
    private readonly int _version;
    private int _index;
    private T? _current;

    internal Enumerator(List<T> list)
    {
        _list = list;
        _version = list._version;
    }

    public void Dispose() { }

    public bool MoveNext()
    {
        var localList = _list;
        if (_version != _list._version)
            ThrowHelper.ThrowInvalidOperationException_InvalidOperation_EnumFailedVersion();

        if ((uint)_index < (uint)localList._size)
        {
            _current = localList._items[_index];
            _index++;
            return true;
        }

        _current = default;
        _index = -1;
        return false;
    }

    public T Current => _current!;

    object? IEnumerator.Current
    {
        get
        {
            if (_index <= 0)
                ThrowHelper.ThrowInvalidOperationException_InvalidOperation_EnumOpCantHappen();
            return _current;
        }
    }

    void IEnumerator.Reset()
    {
        if (_version != _list._version)
            ThrowHelper.ThrowInvalidOperationException_InvalidOperation_EnumFailedVersion();
        _index = 0;
        _current = default;
    }
}
```

Why can `List<T>` use a `struct` enumerator? To understand this, we need to revisit why iterators default to reference types.

A key capability of the iterator pattern is **suspension**. For example:

```csharp
IEnumerable<string> GetData()
{
    yield return "Hello";
    yield return "World";
}

// Caller
var enumerator = GetData().GetEnumerator();
enumerator.MoveNext(); // Gets "Hello"
// Execution suspends here
// ... potentially much later, even on another thread ...
enumerator.MoveNext(); // Gets "World"
```

If the iterator were a value type allocated on the stack, it would vanish once `GetData()` returned (when its stack frame is popped). But `yield return` must preserve execution state for resumption later—possibly in a different scope or thread. Only heap-allocated reference types can safely manage such lifetimes.

Now consider `List<T>`: its underlying data is already fixed in an array on the heap. The `Enumerator` merely acts as a cursor. It doesn’t need suspension semantics and can safely live on the stack.

Our table rows follow the same pattern: row data is already materialized and immutable during iteration. We don’t need suspension. Thus, we can hand-write a `struct`-based enumerator for `PlayerRow`:

```csharp
public int CountColumns() => 2;

public string ColumnText(int index) => index switch
{
    0 => Id,
    1 => Name,
    _ => string.Empty
};

public Enumerator GetEnumerator() => new(this);

public struct Enumerator : IEnumerator<string>
{
    private readonly PlayerRow _row;
    private int _index;

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    internal Enumerator(PlayerRow row)
    {
        _row = row;
        _index = -1;
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public bool MoveNext()
    {
        int nextIndex = _index + 1;
        if (nextIndex < _row.CountColumns())
        {
            _index = nextIndex;
            return true;
        }
        return false;
    }

    public readonly string Current
    {
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        get => _row.ColumnText(_index);
    }

    readonly string IEnumerator<string>.Current => Current;
    readonly object IEnumerator.Current => Current!;
    void IEnumerator.Reset() => _index = -1;
    readonly void IDisposable.Dispose() { }
}
```

The outer loop simplifies to:

```csharp
foreach (var row in Rows)
{
    foreach (var cellText in row)
    {
        writer.Write(cellText);
    }
}
```

With a `struct` enumerator, we eliminate heap allocations entirely during row traversal—achieving **allocation-free** iteration.

### Language Evolution Unlocks Design Evolution

Struct enumerators are great, but they introduce a practical problem: if we add `OrderRow`, `ProductRow`, and dozens of other row types, do we copy-paste the enumerator boilerplate everywhere? That’s a maintenance nightmare. Sacrificing readability for performance isn’t a good trade-off.

#### Source Generators

One solution is to use a **Source Generator**, a powerful feature of modern C# compilers. Source generators run during compilation and can emit additional code based on syntactic and semantic analysis.

Define a `[TableColumn]` attribute:

```csharp
[AttributeUsage(AttributeTargets.Property)]
public sealed class TableColumnAttribute(int index, string name) : Attribute
{
    public int Index => index;
    public string Name => name;
}
```

Update `PlayerRow`:

```csharp
public sealed partial class PlayerRow : ITableRow
{
    [TableColumnAttribute(0, "ID")]
    public required string Id { get; init; }

    [TableColumnAttribute(1, "Name")]
    public required string Name { get; init; }
}

public interface ITableRow
{
    int CountColumns();
    string ColumnText(int index);
}
```

We then write a source generator that:
1. Finds all types implementing `ITableRow`.
2. Inspects properties annotated with `[TableColumn]`.
3. Emits a `struct` enumerator for each row type.

Since generation happens at compile time, the emitted code is indistinguishable from handwritten code—delivering maximum runtime performance without reflection overhead.

This approach has clear benefits: adding new row types is as simple as implementing the interface and annotating properties. However, it also has drawbacks:
1. Writing and maintaining source generators requires deep knowledge of Roslyn APIs.
2. Generated code is embedded within each row type, introducing some degree of intrusion.

#### Shared Iterators and `ref struct`

Source generators can feel heavyweight. Is there a simpler alternative?

First, let’s challenge a common assumption: **When can you use `foreach`?**

Many developers believe the iterated type must implement `IEnumerable`. In reality, `foreach` relies on **compile-time duck typing**. The C# compiler checks whether a `GetEnumerator()` method exists—not whether an interface is implemented.

A canonical example is `Span<T>` and `ReadOnlySpan<T>`. Neither implements `IEnumerable<T>`, yet both work seamlessly with `foreach` because they expose a compatible enumerator:

```csharp
public readonly ref struct ReadOnlySpan<T>
{
    public Enumerator GetEnumerator() => new(this);

    public ref struct Enumerator : IEnumerator<T>
    {
        private readonly ReadOnlySpan<T> _span;
        private int _index;

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        internal Enumerator(ReadOnlySpan<T> span)
        {
            _span = span;
            _index = -1;
        }

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public bool MoveNext()
        {
            int nextIndex = _index + 1;
            if (nextIndex < _span.Length)
            {
                _index = nextIndex;
                return true;
            }
            return false;
        }

        public ref readonly T Current
        {
            [MethodImpl(MethodImplOptions.AggressiveInlining)]
            get => ref _span[_index];
        }

        T IEnumerator<T>.Current => Current;
        object IEnumerator.Current => Current!;
        void IEnumerator.Reset() => _index = -1;
        void IDisposable.Dispose() { }
    }
}
```

This insight inspires a generic, shared iterator using a `ref struct`:

```csharp
public readonly ref struct EnumerableColumns<TRow>(TRow row) where TRow : ITableRow
{
    public Enumerator GetEnumerator() => new(row);

    public ref struct Enumerator : IEnumerator<string>
    {
        private readonly TRow _row;
        private int _index;

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        internal Enumerator(TRow row)
        {
            _row = row;
            _index = -1;
        }

        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public bool MoveNext()
        {
            int nextIndex = _index + 1;
            if (nextIndex < _row.CountColumns())
            {
                _index = nextIndex;
                return true;
            }
            return false;
        }

        public readonly string Current
        {
            [MethodImpl(MethodImplOptions.AggressiveInlining)]
            get => _row.ColumnText(_index);
        }

        readonly string IEnumerator<string>.Current => Current;
        readonly object IEnumerator.Current => Current!;
        void IEnumerator.Reset() => _index = -1;
        readonly void IDisposable.Dispose() { }
    }
}
```

The consuming code becomes:

```csharp
foreach (var row in Rows)
{
    var enumerable = new EnumerableColumns<PlayerRow>(row);
    foreach (var cellText in enumerable)
    {
        writer.Write(cellText);
    }
}
```

With this design, `PlayerRow` only needs to provide basic data access (`CountColumns` and `ColumnText`). All enumeration logic lives in a single, reusable generic `ref struct`.

You might wonder: **why `ref struct`?**

Because:
- We only intend to use this in `foreach` loops—not via `IEnumerator<string> e = row.GetEnumerator()`.
- We don’t need suspension; the enumerator’s lifetime is strictly scoped to the `foreach` block.
- Memory is allocated and reclaimed on the stack, making it ideal for high-frequency scenarios.

## Micro-Benchmark Results

Benchmark code (running on .NET 10):

```csharp
using System.Collections;
using System.Runtime.CompilerServices;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Jobs;

[SimpleJob(RuntimeMoniker.Net10_0)]
[MemoryDiagnoser]
public class EnumerableTest
{
    private static readonly List<PlayerRow> Rows =
    [
        .. Enumerable.Range(1, 10000)
            .Select(i => new PlayerRow
            {
                Id = i.ToString(),
                Name = (10000 - i).ToString()
            })
    ];

    [Benchmark(Baseline = true)]
    public int Array()
    {
        var length = 0;
        foreach (var r in Rows)
        {
            foreach (var s in r.Array())
            {
                length += s.Length;
            }
        }
        return length;
    }

    [Benchmark]
    public int Yield()
    {
        var length = 0;
        foreach (var r in Rows)
        {
            foreach (var s in r.Yield())
            {
                length += s.Length;
            }
        }
        return length;
    }

    [Benchmark]
    public int EnumeratorInner()
    {
        var length = 0;
        foreach (var r in Rows)
        {
            foreach (var s in r)
            {
                length += s.Length;
            }
        }
        return length;
    }

    [Benchmark]
    public int EnumeratorShared()
    {
        var length = 0;
        foreach (var r in Rows)
        {
            var e = new EnumerableColumns<PlayerRow>(r);
            foreach (var s in e)
            {
                length += s.Length;
            }
        }
        return length;
    }

    public sealed class PlayerRow : ITableRow
    {
        public required string Id { get; init; }
        public required string Name { get; init; }

        public IEnumerable<string> Array() => [Id, Name];

        public IEnumerable<string> Yield()
        {
            yield return Id;
            yield return Name;
        }

        public int CountColumns() => 2;

        public string ColumnText(int index) => index switch
        {
            0 => Id,
            1 => Name,
            _ => string.Empty
        };

        public Enumerator GetEnumerator() => new(this);

        public struct Enumerator : IEnumerator<string>
        {
            private readonly PlayerRow _row;
            private int _index;

            [MethodImpl(MethodImplOptions.AggressiveInlining)]
            internal Enumerator(PlayerRow row)
            {
                _row = row;
                _index = -1;
            }

            [MethodImpl(MethodImplOptions.AggressiveInlining)]
            public bool MoveNext()
            {
                int next = _index + 1;
                if (next < _row.CountColumns())
                {
                    _index = next;
                    return true;
                }
                return false;
            }

            public readonly string Current
            {
                [MethodImpl(MethodImplOptions.AggressiveInlining)]
                get => _row.ColumnText(_index);
            }

            readonly string IEnumerator<string>.Current => Current;
            readonly object IEnumerator.Current => Current!;
            void IEnumerator.Reset() => _index = -1;
            readonly void IDisposable.Dispose() { }
        }
    }

    public readonly ref struct EnumerableColumns<TRow>(TRow row) where TRow : ITableRow
    {
        public Enumerator GetEnumerator() => new(row);

        public ref struct Enumerator : IEnumerator<string>
        {
            private readonly TRow _row;
            private int _index;

            [MethodImpl(MethodImplOptions.AggressiveInlining)]
            internal Enumerator(TRow row)
            {
                _row = row;
                _index = -1;
            }

            [MethodImpl(MethodImplOptions.AggressiveInlining)]
            public bool MoveNext()
            {
                int next = _index + 1;
                if (next < _row.CountColumns())
                {
                    _index = next;
                    return true;
                }
                return false;
            }

            public readonly string Current
            {
                [MethodImpl(MethodImplOptions.AggressiveInlining)]
                get => _row.ColumnText(_index);
            }

            readonly string IEnumerator<string>.Current => Current;
            readonly object IEnumerator.Current => Current!;
            void IEnumerator.Reset() => _index = -1;
            readonly void IDisposable.Dispose() { }
        }
    }

    public interface ITableRow
    {
        int CountColumns();
        string ColumnText(int index);
    }
}
```

Results on my machine:

```
BenchmarkDotNet v0.15.8, Windows 11 (10.0.26200.8737/25H2)
AMD Ryzen 7 5700X 3.40GHz, 1 CPU, 16 logical and 8 physical cores
.NET SDK 10.0.301
  [Host]    : .NET 10.0.9
  .NET 10.0 : .NET 10.0.9
```

| Method           | Mean      | Error    | StdDev   | Ratio | RatioSD | Gen0    | Allocated | Alloc Ratio |
|------------------|----------:|---------:|---------:|------:|--------:|--------:|----------:|------------:|
| Array            | 193.39 μs | 2.887 μs | 2.559 μs |  1.00 |    0.02 | 57.3730 |  960000 B |        1.00 |
| Yield            | 152.47 μs | 1.273 μs | 1.191 μs |  0.79 |    0.01 | 23.6816 |  400000 B |        0.42 |
| EnumeratorInner  |  14.71 μs | 0.222 μs | 0.208 μs |  0.08 |    0.00 |       - |         - |        0.00 |
| EnumeratorShared |  15.05 μs | 0.286 μs | 0.318 μs |  0.08 |    0.00 |       - |         - |        0.00 |

We eliminated heap allocations entirely while achieving a **10× performance improvement**.

## Conclusion

C# is no longer confined to the “developer productivity” comfort zone. Through continuous enhancements to low-level capabilities, the language keeps pushing performance boundaries. Moving from passively accepting GC overhead to actively controlling memory layout, modern C# empowers us to write code that is both elegant and blazingly fast—challenging the outdated notion that “C# is just syntax sugar.”

Elegance and extreme performance are not mutually exclusive; they are achievable realities in the C# ecosystem. Whether in game servers, high-frequency trading, or large-scale data processing, modern .NET opens up exciting new possibilities for performance-critical applications.

I hope this article helps you see the evolutionary trajectory of modern .NET—and inspires you to write code that’s as fast as it is beautiful.