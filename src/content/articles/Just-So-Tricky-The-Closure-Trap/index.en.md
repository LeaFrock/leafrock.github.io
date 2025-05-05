---
title: "Just So Tricky: The Closure Trap"
date: "2023-12-03"
lastmod: "2025-05-05"
draft: false
summary: "This article delves into the pitfalls of closures in programming, revealing common issues caused by variable capture through practical examples in C# and Blazor, and provides effective solutions to help developers avoid these mistakes."
slug: "Just-So-Tricky-The-Closure-Trap"
tags: ["dotnet", "blazor"]
series: ["just-so-tricky"]
---

## Concept

A closure is an important concept in functional programming. It refers to a function that captures variables from its enclosing scope. When the function is called, it retains the values of these variables, even if they are outside the function's defined scope.

Closures can be used to implement many useful features:

- **Lazy Evaluation:** The execution of a function can be delayed until it is called.  
- **Callback Functions:** Functions that are invoked when a specific event occurs.  
- **Decorators:** Functions that modify the behavior of other functions.

## Basic Pitfalls

In C#, a common pitfall occurs when dealing with delayed evaluation in a for loop.

For example, consider the following class design (based on C# 12 syntax):

```csharp
        public sealed class ClosureBuilder(Action<Closure> builder)
        {
            public Closure Build()
            {
                var value = new Closure();
                builder.Invoke(value);
                return value;
            }
        }

        public sealed class Closure
        {
            public int Id { get; set; }

            public override string ToString() => $"{nameof(Closure)}:{Id}";
        }
```

Assume we want to "delay" the creation of three `Closure` objects with `Id` values from 1 to 3. Here is the initial code structure:

```csharp
var builders = new List<ClosureBuilder>();
for (int i = 1; i <= 3; i++)
{
    // How should we implement this?
}
foreach (var builder in builders)
{
    Console.WriteLine(builder.Build()); // The `Build` method is called to actually create the `Closure` objects
}
```

For readers who are new to this concept, you might want to pause and think about how to implement the for loop.

***

Many people might initially write the following code:

```csharp
            var builders = new List<ClosureBuilder>();
            for (int i = 1; i <= 3; i++)
            {
                builders.Add(new(c => c.Id = i));
            }
            foreach (var builder in builders)
            {
                Console.WriteLine(builder.Build());
            }
```

However, this will lead to a common mistake... When we run this code, the output is:

```log
Closure:4
Closure:4
Closure:4
```

This seems counterintuitive. Why do all `Closure` objects have an `Id` of 4?

The variable i in the for loop is defined outside the Action (function), so this is a typical closure scenario. Let's look at how the compiler handles this code:

```csharp
        List<ClosureBuilder> list = new List<ClosureBuilder>();
        <>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
        <>c__DisplayClass0_.i = 1;
        while (<>c__DisplayClass0_.i <= 3)
        {
            list.Add(new ClosureBuilder(new Action<Closure>(<>c__DisplayClass0_.<M>b__0)));
            <>c__DisplayClass0_.i++;
        }
        List<ClosureBuilder>.Enumerator enumerator = list.GetEnumerator();
        try
        {
            while (enumerator.MoveNext())
            {
                Console.WriteLine(enumerator.Current.Build());
            }
        }
        finally
        {
            ((IDisposable)enumerator).Dispose();
        }
```

The compiler generates a hidden class for the closure:

```csharp
    [CompilerGenerated]
    private sealed class <>c__DisplayClass0_0
    {
        public int i;

        [System.Runtime.CompilerServices.NullableContext(1)]
        internal void <M>b__0(Closure c)
        {
            c.Id = i;
        }
    }
```

The compiler creates a single hidden class instance for the closure. When creating the `Builder`, the actual Action used is the `<M>b__0` method of the hidden class. This method assigns the value of `<>c__DisplayClass0_0.i` to the `Id` of the `Closure` object.

In other words, after the for loop completes, all `Builders` use the value of `<>c__DisplayClass0_0.i` when creating Closure objects, which is the final value of `i` after the loop: 4.

***

To avoid this pitfall, we need to avoid capturing the same variable in the closure. For this example, we can simply modify the for loop as follows:

```csharp
            var builders = new List<ClosureBuilder>();
            for (int i = 1; i <= 3; i++)
            {
                int x = i; // Use a new temporary variable
                builders.Add(new(c => c.Id = x));
            }
            foreach (var builder in builders)
            {
                Console.WriteLine(builder.Build());
            }
```

The output now matches our expectations:

```log
Closure:1
Closure:2
Closure:3
```

Let's look at how the compiler handles this. The hidden class for the closure remains unchanged (except for variable naming):

```csharp
    [CompilerGenerated]
    private sealed class <>c__DisplayClass0_0
    {
        public int x;

        [System.Runtime.CompilerServices.NullableContext(1)]
        internal void <M>b__0(Closure c)
        {
            c.Id = x;
        }
    }
```

However, the loop code is now different:

```csharp
        List<ClosureBuilder> list = new List<ClosureBuilder>();
        int num = 1;
        while (num <= 3)
        {
            <>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();
            <>c__DisplayClass0_.x = num;
            list.Add(new ClosureBuilder(new Action<Closure>(<>c__DisplayClass0_.<M>b__0)));
            num++;
        }
        List<ClosureBuilder>.Enumerator enumerator = list.GetEnumerator();
        try
        {
            while (enumerator.MoveNext())
            {
                Console.WriteLine(enumerator.Current.Build());
            }
        }
        finally
        {
            ((IDisposable)enumerator).Dispose();
        }
```

Notice that the line `<>c__DisplayClass0_0 <>c__DisplayClass0_ = new <>c__DisplayClass0_0();` is now inside the loop. This means that when creating `Closure` objects, the `Id` value is assigned from the `x` value of the corresponding `<>c__DisplayClass0_0` object, which is the value of `i` at the time the object was created in the loop.

In this example, we also see that closures can introduce additional memory allocation overhead because the compiler generates hidden types and instances for them. Compared to common interview topics like boxing, the overhead of closures is often overlooked. However, in high-performance or low-memory scenarios, it is important for developers to be aware of this.

## Hidden Pitfalls

There are already many articles on closures in various programming languages. If I were to end the article here, it would be rather cliché.

In fact, even experienced developers who know how to use closures correctly can still fall into the closure trap. This is because, in a project, developers often start with a certain platform or framework and may not have a comprehensive understanding of the underlying layers. As a result, they may inadvertently trigger a closure trap in the upper layers due to improper usage.

Take Blazor, for example. Consider the following Razor page code:

```qml
@for (int i = 1; i <= 3; i++)
{
    <p>@i</p>
}
```

This code displays as follows:

![1](1.png)

Now, let's use a very simple [Tag component](https://antblazor.com/zh-CN/components/tag) from Ant Design Blazor to make the numbers look better:

```qml
@for (int i = 1; i <= 3; i++)
{
    <p><Tag>@i</Tag></p>
}
```

And let's see the display:

![2](2.png)

**What happened!?**

When encountering this issue, my colleague must have had a moment of doubt about Blazor and the Ant Design Blazor component. After all, this is counterintuitive and not easily noticeable or related to the closure trap at first glance.

In fact, the Blazor official documentation [directly points out](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/#loop-variables-with-component-parameters-and-child-content) this issue:

> Rendering components inside a for loop requires a local index variable if the incrementing loop variable is used by the component's parameters or RenderFragment child content.

However, many developers do not read the entire Blazor official documentation and thus overlook this point. Moreover, the documentation only vaguely tells us what to do without explaining why, making it difficult for readers to have a concise and clear understanding.

***

To understand the problem, we first need to understand how Blazor handles Razor page code: Essentially, a source generator converts the Razor code into C# code, which mainly overrides the component's (or page's) method for building the render tree (`BuildRenderTree`).

By adding `<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>` to the Blazor project's `.csproj` file, we can find the C# code generated by the source generator during compilation in the `obj\Debug\net8.0\generated\Microsoft.NET.Sdk.Razor.SourceGenerators\Microsoft.NET.Sdk.Razor.SourceGenerators.RazorSourceGenerator` directory.

For the initial code `<p>@i</p>`, it is compiled into:

```csharp
        protected override void BuildRenderTree(global::Microsoft.AspNetCore.Components.Rendering.RenderTreeBuilder __builder)
        {
            __builder.OpenComponent<global::Microsoft.AspNetCore.Components.Web.PageTitle>(0);
            __builder.AddAttribute(1, "ChildContent", (global::Microsoft.AspNetCore.Components.RenderFragment)((__builder2) => {
                __builder2.AddContent(2, "Counter");
            }
            ));
            __builder.CloseComponent();
#nullable restore
#line 27 "G:\CodeRepos\demos\BlazorWasmTest\BlazorWasmTest\Pages\Counter.razor"
 for (int i = 1; i <= 3; i++)
{
    

#line default
#line hidden
#nullable disable
            __builder.OpenElement(3, "p");
#nullable restore
#line (30,9)-(30,10) 24 "G:\CodeRepos\demos\BlazorWasmTest\BlazorWasmTest\Pages\Counter.razor"
__builder.AddContent(4, i);

#line default
#line hidden
#nullable disable
            __builder.CloseElement();
#nullable restore
#line 31 "G:\CodeRepos\demos\BlazorWasmTest\BlazorWasmTest\Pages\Counter.razor"
}

#line default
#line hidden
#nullable disable
        }
```

Readers can ignore the lines starting with `#`, which are compiler directives. Focus on the for loop code. Let's compare the for loop part of the generated code for `<p><Tag>@i</Tag></p>` (with irrelevant lines and directives removed):

```csharp
  for (int i = 1; i <= 3; i++)
{
            __builder.OpenElement(3, "p");
            __builder.OpenComponent<global::AntDesign.Tag>(4);
            __builder.AddAttribute(5, "ChildContent", (global::Microsoft.AspNetCore.Components.RenderFragment)((__builder2) => {
__builder2.AddContent(6, i);
            }
            ));
            __builder.CloseComponent();
            __builder.CloseElement();
}
```

Notice that in the first case, the `p` tag is an Element, so the for loop simply contains:

```csharp
__builder.AddContent(4, i);
```

However, in the second case, `Tag` is a Component, and `@i` is presented as child content. Therefore, in the for loop, `i` is passed into a delegate for the `ChildContent` of the `Tag` component:

```csharp
__builder.AddAttribute(5, "ChildContent", (global::Microsoft.AspNetCore.Components.RenderFragment)((__builder2) => 
{
    __builder2.AddContent(6, i);
}
```

Ah-ha! This is the classic closure trap!

Understanding this, we know how to fix it:

```qml
@for (int i = 1; i <= 3; i++)
{
    var x = i;
    <p><Tag>@x</Tag></p>
}
```

Here is the result:

![1](1.png)

Similarly, the key part of the generated C# code now looks like this:

```csharp
 for (int i = 1; i <= 3; i++)
{
    var x = i;
            __builder.OpenElement(3, "p");
            __builder.OpenComponent<global::AntDesign.Tag>(4);
            __builder.AddAttribute(5, "ChildContent", (global::Microsoft.AspNetCore.Components.RenderFragment)((__builder2) => {
__builder2.AddContent(6, x);
}
            ));
            __builder.CloseComponent();
            __builder.CloseElement();
}
```

**Isn't this the same familiar recipe, with the same familiar taste?**
