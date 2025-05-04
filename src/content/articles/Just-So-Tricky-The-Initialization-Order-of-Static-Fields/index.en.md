---
title: "Just So Tricky: The Initialization Order of Static Fields"
date: "2025-04-27"
lastmod: "2025-05-04"
draft: false
summary: "This article discusses the issue of static field initialization order in C#, which is an easily overlooked pitfall that can lead to program exceptions. Through this article, readers can better understand the rules of static field initialization and avoid similar pitfalls in actual development."
slug: "Just-So-Tricky-The-Initialization-Order-of-Static-Fields"
tags: ["just-so-tricky"]
---

In the "ancient times," programming languages were roughly divided into interpreted and compiled languages, with the former having stricter requirements for the order of code writing. However, with the development of compilers and JIT technology, such classifications have become outdated. For established compiled languages like C#, not writing the code order correctly can also lead to pitfalls!

This article will describe a simplified scenario and introduce some noteworthy "little-known facts." Without further ado, let's look at the following code:

```csharp
public class KengStatic
{
    private static readonly List<int> _nums = InitNumbers();

    private static readonly Random Rand = Random.Shared;

    public static int Get(int index) => _nums[index];

    private static List<int> InitNumbers()
        => [.. Enumerable.Repeat(100, 10).Select(p => Rand.Next(p))];
}
```

If you, the reader, do not see any issues with this code, you might try calling it in the console: `Console.WriteLine(KengStatic.Get(0))`. You will encounter a `System.NullReferenceException` in actual projects, this exception might be wrapped inside a `TypeInitializationException`, making it even more confusing.

The reason is quite simple: the initialization code for the Rand field is in the wrong position. Since the initialization of the `_nums` field depends on the initialization of Rand, the order of the code needs to be swapped:

```csharp
public class KengStatic
{
    private static readonly Random Rand = Random.Shared;

    private static readonly List<int> _nums = InitNumbers();

    public static int Get(int index) => _nums[index];

    private static List<int> InitNumbers()
        => [.. Enumerable.Repeat(100, 10).Select(p => Rand.Next(p))];
}
```

It runs correctly. It can be seen that during runtime, static fields are initialized in the order they are written in the code. Therefore, we need to ensure that the fields that are depended upon are written first, and those that depend on other fields are written afterward.

Of course, some readers might smirk and say: just initialize everything in a static constructor, right?

```csharp
public class KengStatic
{
    private static readonly List<int> _nums;

    private static readonly Random Rand;

    static KengStatic()
    {
        Rand = Random.Shared;
        _nums = InitNumbers();
    }

    public static int Get(int index) => _nums[index];

    private static List<int> InitNumbers()
        => [.. Enumerable.Repeat(100, 10).Select(p => Rand.Next(p))];
}
```

Indeed, with a static constructor, we don't have to care about the order of field declarations but instead write the assignment code in the order of field initialization. However, the devil is in the details. When we do this, we should at least know: what is the cost of explicitly declaring a static constructor?

Let's first look at the IL of the code part that runs correctly without a static constructor:

```gradle
.class public auto ansi beforefieldinit KengStatic
    extends [System.Runtime]System.Object
{
   //...
}
```

You can see that the keyword `beforefieldinit` is marked on the class `KengStatic`. Then, let's switch to the IL of the code with an explicitly declared static constructor:

```gradle
.class public auto ansi KengStatic
    extends [System.Runtime]System.Object
{
   //...
}
```

Comparing the two, it is easy to notice that the keyword `beforefieldinit` is missing. What does it do?

There are many blogs online that have "translated" or "borrowed" from the chapter "C# and beforefieldinit" in "C# in Depth" (this link may be unstable).

The book introduces the differences with and without `beforefieldinit`:

* If marked, the type initialization method may be executed before or when the first static field is accessed.
* If unmarked, the type initialization method is strictly executed when the first static field or method is accessed.

The key excerpt from the book is as follows:

![The CLI specification (ECMA 335)](cli.png)

In fact, the official documentation has a more precise and concise description of their differences, but it's a bit tricky to find—in the table of the ["TypeAttributes Enum"](https://learn.microsoft.com/en-us/dotnet/api/system.reflection.typeattributes#fields) documentation, `BeforeFieldInit` is described as: "Specifies that calling static methods of the type does not force the system to initialize the type."

In summary, whether to explicitly declare a static constructor affects the timing of type loading. If we do not explicitly declare a static constructor, the CLR can decide when to initialize the type, especially when it deems it appropriate to initialize the type in advance; on the other hand, the CLR will only initialize the type when the static class is first used. Therefore, declaring a static constructor is equivalent to constraining the behavior of the CLR and limiting runtime optimizations.

```csharp
do
{
    if(!CheckTypeInitialized())
    {
        // call type's initializer method
    }
}
while(condition);
```

This can affect performance, and while the impact is often minimal, it can sometimes lead to measurable performance regressions.

In the official "C# Programming Guide" documentation, the "Static Constructors" section [Remarks](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/static-constructors) describes the basic points of static constructors. It includes:

> * If a static constructor throws an exception, the runtime doesn't invoke it a second time, and the type remains uninitialized for the lifetime of the application domain. Most commonly, a TypeInitializationException exception is thrown when a static constructor is unable to instantiate a type or for an unhandled exception occurring within a static constructor. For static constructors that aren't explicitly defined in source code, troubleshooting might require inspection of the intermediate language (IL) code.
> * The presence of a static constructor prevents the addition of the BeforeFieldInit type attribute. This limits runtime optimization.
> * A field declared as static readonly can only be assigned as part of its declaration or in a static constructor. When an explicit static constructor isn't required, initialize static fields at declaration rather than through a static constructor for better runtime optimization.

It can be seen that, from the perspective of CLR friendliness, it is more recommended not to declare a static constructor, but to pay attention to the order of field code to avoid this pitfall; it can also be seen that in order to allow developers of different levels to write good code, the official documentation has put a lot of effort into it, haha.
