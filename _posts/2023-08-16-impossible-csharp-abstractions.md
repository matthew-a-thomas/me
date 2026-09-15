---
title: Things in C# Over Which You Cannot Abstract
description: A short list of concepts and features in C# over which you cannot abstract
category: programming
---

I love abstractions. One of my favorite things about software is finding, making, and using good abstractions.

C# has many wonderful abstractions.

But C# also has concepts and features over which it's simply not possible to abstract.

(A prior version of this article complained about `ref struct`, but [C# 13 fixed that](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/ref-struct), hooray!)

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