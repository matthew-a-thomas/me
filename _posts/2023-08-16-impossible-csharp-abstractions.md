---
title: Things in C# Over Which You Cannot Abstract
description: A short list of concepts and features in C# over which you cannot abstract
category: programming
---

I love abstractions. One of my favorite things about software is finding, making, and using good abstractions.

C# has many wonderful abstractions.

But C# also has concepts and features over which it's simply not possible to abstract.

## `ref struct`

The `ref` keyword was introduced to enable performance optimizations. When you make a `ref` type then you're telling the compiler that its instances can only live on the stack. They cannot live on the heap.

This has [the following consequences](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/ref-struct):
> * A `ref struct` can't be the element type of an array.
> * A `ref struct` can't be a declared type of a field of a class or a non-`ref struct`.
> * A `ref struct` can't implement interfaces.
> * A `ref struct` can't be boxed to System.ValueType or System.Object.
> * A `ref struct` can't be a type argument.
> * A `ref struct` variable can't be captured by a lambda expression or a local function.
> * A `ref struct` variable can't be used in an async method. However, you can use `ref struct` variables in synchronous methods, for example, in methods that return Task or Task<TResult>.
> * A `ref struct` variable can't be used in iterators.

That's a long list of things you can't do with them!

Let's zoom in on the fact that they can't implement interfaces. Which are one of the primary tools for abstraction in C#.

Suppose you want a function that will return the sum of all the odd elements of a sequence of numbers:

```csharp
public int SumOdd(IEnumerable<int> sequence)
{
  var even = true;
  var sum = 0;
  foreach (var number in sequence)
  {
    if (even)
    {
      even = false;
    }
    else
    {
      even = true;
      sum += number;
    }
  }
  return sum;
}
```

Because of the power of abstraction you can use that function on `int[]`, `List<int>`, `ImmutableList<int>`, `ArraySegment<int>`, `IReadOnlyCollection<int>`, and so on:

```csharp
Assert.Equal(4, SumOdd(new int[] { 0, 1, 2, 3, 4 }));
Assert.Equal(4, SumOdd(new List<int> { 0, 1, 2, 3, 4 }));
```

But you cannot use that function on `ReadOnlySpan<int>`. Instead you have to make a choice:
1. Duplicate the function with minimal changes to support `ReadOnlySpan<int>`
   ```csharp
   public int SumOdd(ReadOnlySpan<int> sequence)
   {
     var even = true;
     var sum = 0;
     foreach (ref readonly var number in sequence)
     {
       if (even)
       {
         even = false;
       }
       else
       {
         even = true;
         sum += number;
       }
     }
     return sum;
   }
   ```
2. Pointlessly copy the `ReadOnlySpan<int>` into an array:
   ```csharp
   ReadOnlySpan<int> span = new int[] { 0, 1, 2, 3, 4 };
   Assert.Equal(4, SumOdd(span.ToArray()));
   ```
3. _Sometimes_ you'll have the `ReadOnlyMemory<int>` within scope from which the `ReadOnlySpan<int>` came. In which case you can make an adapter for `ReadOnlyMemory<int>`:
   ```csharp
   public sealed record ReadOnlyMemoryToEnumerableAdapter<T>(ReadOnlyMemory<T> Memory) : IEnumerable<T>
   {
     public IEnumerator<int> GetEnumerator() => ...;
     IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
   }
   ```
   ...but now we're no longer talking about abstracting over `ReadOnlySpan<int>`. Which is kind of the point

So you cannot abstract over `ref struct`.

## Most (all?) C# coding-by-convention concepts

Here are some examples of what I mean by "coding by convention":
* [`foreach` loops](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/iteration-statements#the-foreach-statement)
  > You can use it with an instance of any type that satisfies the following conditions:
  >
  > * A type has the public parameterless `GetEnumerator` method. Beginning with C# 9.0, the `GetEnumerator` method can be a type's extension method.
  > * The return type of the `GetEnumerator` method has the public `Current` property and the public parameterless `MoveNext` method whose return type is `bool`.
* [`using` statements](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/using)
  > You can also use the `using` statement and declaration with an instance of a ref struct that fits the disposable pattern. That is, it has an instance `Dispose` method, which is accessible, parameterless and has a `void` return type.
* [`await` operator](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#12982-awaitable-expressions)
  > An expression `t` is awaitable if one of the following holds:
  > 
  > * `t` is of compile-time type `dynamic`
  > * `t` has an accessible instance or extension method called `GetAwaiter` with no parameters and no type parameters, and a return type `A` for which all of the following hold:
  >   * `A` implements the interface `System.Runtime.CompilerServices.INotifyCompletion` (hereafter known as `INotifyCompletion` for brevity)
  >   * `A` has an accessible, readable instance property `IsCompleted` of type `bool`
  >   * `A` has an accessible instance method `GetResult` with no parameters and no type parameters

There are a growing number of things like this in C# where you'll be able to use this or that feature if you follow a bunch of rules.

But it's impossible to abstract over all `foreach`-able types, or all disposable types, or all awaitable types. Because with each of these features C# introduced a convention, and a convention is different than an interface. It doesn't matter if they also introduced an interface (as is the case with `foreach` and `IEnumerable`, or with `using` and `IDisposable`)&mdash;the fact that they introduced a convention at all means there will be types which are `foreach`-able that do not implement an interface.

It's impossible to abstract over _all_ manifestations of any of these concepts.

## Colored functions

C# pioneered `async`/`await`. And so [colored functions](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) are C#'s fault.

Yes, async cancer is a real thing. I've experienced it. You are twelve layers deep in a project with 10,000 source code files and discover that you need to call an asynchronous function from a synchronous function. Then you have to waste the rest of the day refactoring the entire application to support async _from the top all the way down_. And you can only hope that you didn't break all the completely unrelated things you had to modify just to support this.

You cannot abstract over the color of a function.