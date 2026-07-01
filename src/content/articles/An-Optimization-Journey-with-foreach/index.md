---
title: "一次foreach的优化之旅"
date: "2026-06-30"
lastmod: "2026-06-30"
draft: false
summary: "一次foreach的极致优化：设计一个通用零分配的遍历方案，实测在.NET 10下，性能提升10倍，内存分配归零。依然优雅的代码，获得极致的效率！"
slug: "An-Optimization-Journey-with-foreach"
tags: ["dotnet"]
series: ["performance"]
---

## 从一个实际的需求说起…

我想很多人都遇到过生成表格的需求，比如将数据导出成Excel文件，或者渲染进PDF文件等等。当项目中有很多处都有类似需求时，往往就希望将表格处理的逻辑封装起来。大体逻辑很简单：

1. 为表格的行建模`TRow`；
2. 以某种方式声明`TRow`到各列数据（通常是字符串）的映射关系；
3. 遍历所有行，执行每行的写入或渲染操作。

大致伪代码如下：

```csharp
public sealed class TableGenerator<TRow>(IReadOnlyList<TRow> Rows) where TRow: ITableRow
{
    public void Generate(ICellWriter writer)
    {
        foreach (var row in Rows)
        {
            foreach (var cellText in Row.Columns())
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

现在假设我们要生成一个简单的玩家表格，行定义如下：

```csharp
public sealed class PlayerRow: ITableRow
{
    public required string Id { get; init; }

    public required string Name { get; init; }
}
```

### 最简单的2个实现

一个很自然的想法是使用数组：

```csharp
public IEnumerable<string> Columns() => [Id, Name];
```

问题是：在遍历每一行时，我们都会创建一个`string[]`。数组占用的堆内存，除了自身的引用头外，还包括字符串元素的体积，即空间复杂度是O(N)，N表示列数量。如果列很多的话，必然会导致大量的内存分配。

于是我们可能会想到使用迭代器，它可以缓解数组方案的内存分配问题：

```csharp
public IEnumerable<string> Columns()
{
    yield return Id;
    yield return Name;
}
```

编译器会将`yield return`编译成一个迭代器类，以状态机的方式实现遍历。示例如下（随编译器升级结果未来可能会不断变化）：

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
			get
			{
				return <>2__current;
			}
		}

		object IEnumerator.Current
		{
			[DebuggerHidden]
			[return: Nullable(0)]
			get
			{
				return <>2__current;
			}
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

		bool IEnumerator.MoveNext()
		{
			//ILSpy generated this explicit interface implementation from .override directive in MoveNext
			return this.MoveNext();
		}

		[DebuggerHidden]
		void IEnumerator.Reset()
		{
			throw new NotSupportedException();
		}

		[DebuggerHidden]
		[return: Nullable(new byte[] { 1, 0 })]
		IEnumerator<string> IEnumerable<string>.GetEnumerator()
		{
			<Columns>d__8 <Columns>d__9;
			if (<>1__state == -2 && <>l__initialThreadId == Environment.CurrentManagedThreadId)
			{
				<>1__state = 0;
				<Columns>d__9 = this;
			}
			else
			{
				<Columns>d__9 = new <Columns>d__8(0);
				<Columns>d__9.<>4__this = <>4__this;
			}
			return <Columns>d__9;
		}

		[DebuggerHidden]
		IEnumerator IEnumerable.GetEnumerator()
		{
			return ((IEnumerable<string>)this).GetEnumerator();
		}
	}
```

原方法被编译为：

```csharp
	[IteratorStateMachine(typeof(<Columns>d__8))]
	public IEnumerable<string> Columns()
	{
		<Columns>d__8 obj = new <Columns>d__8(-2);
		obj.<>4__this = this;
		return obj;
	}
```

可见此时在遍历每一行时，我们都会创建一个`<Columns>d__8`迭代器实例，但它占用的内存就只有自身的引用头等，跟列的数量无关，空间复杂度就降低到了O(1)。

这2种方式，都会在每行遍历时，在堆内存上产生一次性分配。在操作结束后，所分配的数组或迭代器实例，大概率会在Gen 0被GC回收，对于非热路径下的场景，如此已经足够。然而，如果代码是在热路径上执行呢？

### 结构体（struct）带来性能的飞跃

在热路径上的代码，任何一点可测量的优化都会因为海量执行次数放大优化的收益，产生很大的价值。

上述的迭代器实现，虽然使得空间复杂度跟表格列的数量无关，但还是跟行的数量线性相关。编译器出于对象生命周期的安全性考虑，默认生成的迭代器是引用类型，那么迭代器可否是值类型呢？试想若迭代器是一个结构体，那么遍历各行时，迭代器将不会分配在需要GC回收的堆内存上，而是即用即收，堪称一次性能飞跃！

答案是可以的！我们最常用的`List<T>`类，使用`foreach`用的就是`struct`枚举器：

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

            public void Dispose()
            {
            }

            public bool MoveNext()
            {
                List<T> localList = _list;

                if (_version != _list._version)
                {
                    ThrowHelper.ThrowInvalidOperationException_InvalidOperation_EnumFailedVersion();
                }

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
                    {
                        ThrowHelper.ThrowInvalidOperationException_InvalidOperation_EnumOpCantHappen();
                    }

                    return _current;
                }
            }

            void IEnumerator.Reset()
            {
                if (_version != _list._version)
                {
                    ThrowHelper.ThrowInvalidOperationException_InvalidOperation_EnumFailedVersion();
                }

                _index = 0;
                _current = default;
            }
        }
```

`List<T>`类使用`struct`枚举器的原因显而易见，作为使用最广泛的底层基础类型之一，自然极其重视性能，否则每处`foreach`就产生一个引用类型的迭代器分配，极易造成GC压力。那么进一步问下去：为什么`List<T>`类可以使用`struct`枚举器？回答这个问题，我们先需要了解“默认生成的迭代器是引用类型”的设计动机。

迭代器模式一个非常强大和重要的功能是支持挂起（Suspension），比如：

```csharp
    IEnumerable<string> GetData()
    {
        yield return "Hello";
        yield return "World";
    }

    // 调用方
    var enumerator = GetData().GetEnumerator();
    enumerator.MoveNext(); // 拿到 Hello
    // 此时方法执行到一半停住了
    // ... 这里可能隔了很久，甚至跨线程了 ...
    enumerator.MoveNext(); // 拿到 World
```

如果迭代器是值类型，它默认是分配在栈上的，一旦`GetData()`方法执行完毕（或者说当前的栈帧被弹出），这个迭代器就没了。但是`yield return`需要把“执行到一半的状态”保存下来，留给未来的某个时刻（给另一个方法/线程）继续调用。这种跨作用域的生命周期管理，只有堆上的引用类型能做到，而栈上的值类型做不到这一点。

那么，如果我们根本不需要“挂起”的能力呢？就比如`List<T>`，从源代码可以看出，它的成员已经固定在底层堆内存上的数组中了，`Enumerator`只扮演一个索引游标的角色，完全可以活在栈上，随用随收。

搞清楚这里的细微差别，我们自然也可以判断出遍历表格的所有行数据都已经固定，也是不需要挂起能力的，我们可以对`PlayerRow`手写一个结构体版本的`Enumerator`：

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
            int index = _index + 1;
            if (index < _row.CountColumns())
            {
                _index = index;
                return true;
            }

            return false;
        }

        public readonly string Current
        {
            [MethodImpl(MethodImplOptions.AggressiveInlining)]
            get => _row.ColumnText(_index);
        }

        /// <inheritdoc />
        readonly string IEnumerator<string>.Current => Current;

        /// <inheritdoc />
        readonly object IEnumerator.Current => Current!;

        /// <inheritdoc />
        void IEnumerator.Reset() => _index = -1;

        /// <inheritdoc />
        readonly void IDisposable.Dispose() { }
    }
```

上层的循环可以直接写成：

```csharp
    foreach (var row in Rows)
    {
        foreach (var cellText in Row)
        {
            // Do something...
            writer.Write(cellText);
        }
    }
```

使用了结构体版本的迭代器后，我们又进一步抹除了堆内存空间复杂度和遍历行数的关系，实现了Allocation Free。

### 语法进化带来设计的飞跃

结构体迭代器固然美好，但在实际应用中还是有个问题：如果以后加一个 `OrderRow`、一个 `ProductRow`，难道要把迭代器的样板代码复制粘贴N遍吗？这维护起来简直是噩梦，为了性能牺牲可读性，这笔账划不来。

#### 源生成器（Source Generator）

一个方案是编写和使用源生成器。源生成器是现代C#编译器提供的一个非常强大的特性，可以根据编译期的语法语义自定义进一步生成的代码。比如我们定义一个`TableColumnAttribute`:

```csharp
    [AttributeUsage(AttributeTargets.Property)]
    public sealed class TableColumnAttribute(int index, string name) : Attribute
    {
        public int Index => index;

        public string Name => name;
    }
```

将`PlayerRow`调整为：

```csharp
public sealed partial class PlayerRow: ITableRow
{
    [TableColumnAttribute(0, "ID")]
    public required string Id { get; init; }

    [TableColumnAttribute(1, "姓名")]
    public required string Name { get; init; }
}

public interface ITableRow
{
    int CountColumns();

    string ColumnText(int index);
}
```

我们编写一个源生成器：先从语法语义树中筛选出当前项目中所有继承`ITableRow`接口的`TRow`类，再获取所有携带`TableColumnAttribute`特性的公开属性，根据特性上设置的值生成上述结构体版本的`Enumerator`。生成代码的过程都是在编译期完成的，生成的代码结果和手写无异，相对传统的反射等方案确保了最佳的运行时性能。

源生成器带来的便利性是毋庸置疑的：后续新增`OrderRow`等类型时，只需要让它继承接口，为公开属性添加特性，重新编译即可。当然，此方案也不是毫无缺点：

1. 编写和维护源生成器的门槛还是较高的，需要开发者熟悉语法语义的概念；
2. 因为迭代器写在类的内部，生成的代码对`TRow`类有一定侵入性。

#### 共享迭代器及`ref struct`

写源生成器往往会让人嫌麻烦，有没有更直观的方案呢？

有的。但我要先向读者抛出一个问题：什么情况下，我们可以使用`foreach`语法？

相信不少读者和我以前一样，觉得要使用`foreach`，对象必须是一个继承`IEnumerable`接口的集合。

但事实上，`foreach`其实并不关心你的类型是否实现了`IEnumerable`，它只关心一件事：你有没有`GetEnumerator()`方法？

换言之，`foreach`使用的是编译期鸭子类型（Duck Typing），C#编译器在编译时只检查模式（Pattern）是否存在，而不是检查契约（Contract）是否被继承。

典型的案例是`Span<T>`和`ReadOnlySpan<T>`。查看`ReadOnlySpan<T>`的源代码，可以看到它被定义成`public readonly ref struct ReadOnlySpan<T>`，并没有继承`IEnumerable<T>`。但是我们可以正常使用`foreach`来遍历它代表的连续内存，因为它定义了迭代器和`GetEnumerator()`方法：

```csharp
    /// <summary>Gets an enumerator for this span.</summary>
    public Enumerator GetEnumerator() => new Enumerator(this);

    /// <summary>Enumerates the elements of a <see cref="ReadOnlySpan{T}"/>.</summary>
    public ref struct Enumerator : IEnumerator<T>
    {
        /// <summary>The span being enumerated.</summary>
        private readonly ReadOnlySpan<T> _span;
        /// <summary>The next index to yield.</summary>
        private int _index;

        /// <summary>Initialize the enumerator.</summary>
        /// <param name="span">The span to enumerate.</param>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        internal Enumerator(ReadOnlySpan<T> span)
        {
            _span = span;
            _index = -1;
        }

        /// <summary>Advances the enumerator to the next element of the span.</summary>
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        public bool MoveNext()
        {
            int index = _index + 1;
            if (index < _span.Length)
            {
                _index = index;
                return true;
            }

            return false;
        }

        /// <summary>Gets the element at the current position of the enumerator.</summary>
        public ref readonly T Current
        {
            [MethodImpl(MethodImplOptions.AggressiveInlining)]
            get => ref _span[_index];
        }

        /// <inheritdoc />
        T IEnumerator<T>.Current => Current;

        /// <inheritdoc />
        object IEnumerator.Current => Current!;

        /// <inheritdoc />
        void IEnumerator.Reset() => _index = -1;

        /// <inheritdoc />
        void IDisposable.Dispose() { }
    }
```

这就启发了我们的思路，基于泛型写一个`ref struct`共享迭代器：

```csharp
    public readonly ref struct EnumerableColumns<TRow>(TRow row) where TRow : ITableRow
    {
        public Enumerator GetEnumerator() => new(row);

        /// <summary>
        /// 参考<see cref="ReadOnlySpan{T}.Enumerator"/>。
        /// </summary>
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
                int index = _index + 1;
                if (index < _row.CountColumns())
                {
                    _index = index;
                    return true;
                }

                return false;
            }

            public readonly string Current
            {
                [MethodImpl(MethodImplOptions.AggressiveInlining)]
                get => _row.ColumnText(_index);
            }

            /// <inheritdoc />
            readonly string IEnumerator<string>.Current => Current;

            /// <inheritdoc />
            readonly object IEnumerator.Current => Current!;

            /// <inheritdoc />
            void IEnumerator.Reset() => _index = -1;

            /// <inheritdoc />
            readonly void IDisposable.Dispose() { }
        }
    }
```

上层的循环可以写成：

```csharp
    foreach (var row in Rows)
    {
        var e = new EnumerableColumns<PlayerRow>(row);
        foreach (var cellText in e)
        {
            // Do something...
            writer.Write(cellText);
        }
    }
```

如此设计下，`PlayerRow` 只需要提供最基础的“读数据”的能力（一个接口，两个方法），而复杂的迭代逻辑被封装在了泛型结构体迭代器`EnumerableColumns<TRow>`中。

有读者可能会疑问：为什么使用`ref struct`呢？

因为我们这里只需要用在`foreach`上，不需要也不希望出现`IEnumerator<string> e = row.GetEnumerator()`的用法；我们也不需要挂起能力，迭代器的生命周期完全限定在`foreach`循环块内；内存分配和回收都在栈上自动完成，正适用于高频调用场景。

## 微基准测试结果

源代码如下：

```csharp
using System.Collections;
using System.Runtime.CompilerServices;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Jobs;

[SimpleJob(RuntimeMoniker.Net10_0)]
[MemoryDiagnoser]
public class EnumerableTest
{
    private static readonly List<PlayerRow> Rows = [.. Enumerable.Range(1,10000)
        .Select(i => new PlayerRow()
        {
            Id = i.ToString(),
            Name = (10000 - i).ToString(),
        })];

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

        /// <summary>
        /// 参考<see cref="ReadOnlySpan{T}.Enumerator"/>。
        /// </summary>
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
                int index = _index + 1;
                if (index < _row.CountColumns())
                {
                    _index = index;
                    return true;
                }

                return false;
            }

            public readonly string Current
            {
                [MethodImpl(MethodImplOptions.AggressiveInlining)]
                get => _row.ColumnText(_index);
            }

            /// <inheritdoc />
            readonly string IEnumerator<string>.Current => Current;

            /// <inheritdoc />
            readonly object IEnumerator.Current => Current!;

            /// <inheritdoc />
            void IEnumerator.Reset() => _index = -1;

            /// <inheritdoc />
            readonly void IDisposable.Dispose() { }
        }
    }

    public readonly ref struct EnumerableColumns<TRow>(TRow row) where TRow : ITableRow
    {
        public Enumerator GetEnumerator() => new(row);

        /// <summary>
        /// 参考<see cref="ReadOnlySpan{T}.Enumerator"/>。
        /// </summary>
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
                int index = _index + 1;
                if (index < _row.CountColumns())
                {
                    _index = index;
                    return true;
                }

                return false;
            }

            public readonly string Current
            {
                [MethodImpl(MethodImplOptions.AggressiveInlining)]
                get => _row.ColumnText(_index);
            }

            /// <inheritdoc />
            readonly string IEnumerator<string>.Current => Current;

            /// <inheritdoc />
            readonly object IEnumerator.Current => Current!;

            /// <inheritdoc />
            void IEnumerator.Reset() => _index = -1;

            /// <inheritdoc />
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

笔者的测试结果如下：

```

BenchmarkDotNet v0.15.8, Windows 11 (10.0.26200.8737/25H2/2025Update/HudsonValley2)
AMD Ryzen 7 5700X 3.40GHz, 1 CPU, 16 logical and 8 physical cores
.NET SDK 10.0.301
  [Host]    : .NET 10.0.9 (10.0.9, 10.0.926.27113), X64 RyuJIT x86-64-v3
  .NET 10.0 : .NET 10.0.9 (10.0.9, 10.0.926.27113), X64 RyuJIT x86-64-v3

Job=.NET 10.0  Runtime=.NET 10.0  

```
| Method           | Mean      | Error    | StdDev   | Ratio | RatioSD | Gen0    | Allocated | Alloc Ratio |
|----------------- |----------:|---------:|---------:|------:|--------:|--------:|----------:|------------:|
| Array            | 193.39 μs | 2.887 μs | 2.559 μs |  1.00 |    0.02 | 57.3730 |  960000 B |        1.00 |
| Yield            | 152.47 μs | 1.273 μs | 1.191 μs |  0.79 |    0.01 | 23.6816 |  400000 B |        0.42 |
| EnumeratorInner  |  14.71 μs | 0.222 μs | 0.208 μs |  0.08 |    0.00 |       - |         - |        0.00 |
| EnumeratorShared |  15.05 μs | 0.286 μs | 0.318 μs |  0.08 |    0.00 |       - |         - |        0.00 |


结果显示我们将堆内存的分配抹除干净的同时，遍历的性能还提升了10倍！

## 结语

C#早已不再局限于“提升开发效率”的舒适区，而是通过一系列底层能力的持续增强，不断抬升性能的上限。从被迫接受GC开销，到主动掌控内存布局，语言的演进让我们获得了更强的内存控制力，得以用高级语言的优雅语法，写出媲美手动优化般的极致性能；也让.NET在游戏服务器、高频交易、大数据处理等对性能极度敏感的领域拥有了更广阔的想象空间。

希望读者能够借助本文，看到现代.NET的进化轨迹，不被“C#增加的只是语法糖”之类的肤浅认知所误导。在C#的世界里，优雅的代码与极致的性能从来不是互斥的选项，而是可以兼得的现实。