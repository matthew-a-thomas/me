---
title: Async and Await in C# vs Rust
description: Why Rust futures are better because they are polled
category: programming
---

Both [Rust](https://www.rust-lang.org/) and
[C#](https://docs.microsoft.com/en-us/dotnet/csharp/tour-of-csharp/) support
asynchronous methods and awaitable expressions.

However, in C# you have to allocate things on the heap and perform dynamic
dispatch when you compose awaitable expressions. Using the heap puts pressure on
the garbage collector which slows things down, and dynamic dispatch is opaque to
the compiler and puts pressure on the JIT engine which slows things down.

Rust doesn't have these problems. Instead, async/await is a zero-cost
abstraction. It's even possible to use async/await in `[no_std]` (a heapless
environment).

I'm going to explain how async/await works in both languages, then show how the
fact that Rust's `Future` trait is polled means Rust doesn't have to allocate
on the heap or perform dynamic dispatch when you compose awaitable expressions
in Rust. I also hope to clear up a common misunderstanding about the nature of
polling the futures in Rust (no, polling doesn't make them slow or inefficient).

I think explaining why or how to use async/await is out of the scope of this
article. In fact, if you aren't totally comfortable using async/await then this
article might not be for you. But if you love it and want to learn a little more
about why Rust's polled futures are great then keep reading.

It's surprising, but _polled_ futures are better!

## Overview

_Warning: Probably none of the code in this article compiles or works. It was
written off the cuff and should only be taken for inspiration._

First, let's take a look at some example async code in both languages.

In C# you can do this:

```csharp
using System;
using System.Threading.Tasks;

static class Program
{
    static void Main()
    {
        var task = RunAsync(); // Immediately prints "Just a second..."
        task.GetAwaiter().GetResult(); // Blocks as it delays for one second then prints "Ok!"
    }

    static async Task RunAsync()
    {
        Console.WriteLine("Just a second...");
        await Task.Delay(TimeSpan.FromSeconds(1));
        Console.WriteLine("Ok!");
    }
}
```

And in Rust you can write this:

```rust
use async_std::task;
use futures::executor::block_on;
use std::time::Duration;

async fn run() {
    println!("Just a second...");
    task::sleep(Duration::from_secs(1)).await;
    println!("Ok!");
}

fn main() {
    let future = run(); // Doesn't print anything
    block_on(future); // Blocks as it prints "Just a second...", delays for one second, and then prints "Ok!"
}
```

Both programs will output "Just a second...", then nothing will happen for one
second, then they'll both output "Ok!".

The **asynchronous methods** in both languages are marked by the `async`
keyword:

```csharp
static async Task RunAsync()
//     ^^^^^
```

```rust
   async fn run()
// ^^^^^
```

And the **awaitable expressions** in both languages involve the `await` keyword:

```csharp
   await Task.Delay(TimeSpan.FromSeconds(1));
//       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expression
// ^^^^^ keyword
```

```rust
   task::sleep(Duration::from_secs(1)).await;
//                                     ^^^^^^ keyword
// ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expression
```

Functionally, C#'s tasks are pretty similar to Rust's futures. They're both
pretty ergonomic, and they both bring all the benefits of async/await code.

You'll notice that one difference is that a C# task automatically starts as soon
as you create it, and it runs itself, but a future in Rust is inert and has to
be ushered along by something else.

Rust's futures have to be polled.

_What?!_ Yes you heard right. _Polled_.

But that's a good thing. To understand why, we need to do a deep dive into C#.

## Deep dive&mdash;C#

Let's turn our attention to C#. I'll explain how things work in C# and then show
why heap allocations and dynamic dispatch are inevitable.

### How the `await` keyword works

C#'s compiler automatically transforms `async` methods into a state machine. And
within an `async` method you can `await`
[awaitable expressions](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#awaitable-expressions).

For example, when you execute this `RunAsync()` method:

```csharp
async Task RunAsync()
{
  Console.WriteLine("Running!");
  int value = await GetValueAsync();
  if (value % 2 == 0)
  {
    await DoEvenThingAsync();
  }
  else
  {
    object oddResult = await DoOddThingAsync();
    Console.WriteLine(oddResult.ToString());
  }
}
```

...then you're actually executing something like this monstrosity that the
compiler automatically generated for you:

```csharp
Task RunAsync()
{
  int value;
  object oddResult;
  var stateMachine = new StateMachine(
    (int step) =>
    {
      switch (step)
      {
        case 0:
        {
          Console.WriteLine("Running!");
          var awaitable = GetValueAsync();
          var awaiter = awaitable.GetAwaiter();
          if (awaiter.IsCompleted)
          {
            value = awaiter.GetResult();
            stateMachine.RunStep(1);
          }
          else
          {
            awaiter.OnCompleted(x =>
            {
              value = x;
              stateMachine.RunStep(1);
            });
          }
        }
          break;
        case 1:
        {
          if (value % 2 == 0)
          {
            var awaitable = DoEvenThingsAsync();
            var awaiter = awaitable.GetAwaiter();
            if (awaiter.IsCompleted)
            {
              awaiter.GetResult();
            }
            else
            {
              awaiter.OnCompleted(() => stateMachine.RunStep(2));
            }
          }
          else
          {
            var awaitable = DoOddThingsAsync();
            var awaiter = awaitable.GetAwaiter();
            if (awaiter.IsCompleted)
            {
              oddResult = awaiter.GetResult();
              stateMachine.RunStep(3);
            }
            else
            {
              awaiter.OnCompleted(x =>
              {
                oddResult = x;
                stateMachine.RunStep(3);
              });
            }
          }
        }
          break;
        case 2:
        {
          stateMachine.Finish();
        }
          break;
        case 3:
        {
          Console.WriteLine(oddResult.ToString());
          stateMachine.Finish();
        }
          break;
      }
    }
  );
  stateMachine.RunStep(0);
  return stateMachine.ToTask();
}
```

Of course the above is just inspirational pseudocode. But can you trace through
it and see how it works?

Please don't get lost. What I want you to see is if you want to get a value out
of a C#
[awaitable expression](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#awaitable-expressions),
you have to do something like this:

```csharp
Action<T> handleTheResult = ...;

var awaiter = (expression).GetAwaiter();
if (awaiter.IsCompleted)
{
  var result = awaiter.GetResult(); // Will return immediately; might throw an exception
  handleTheResult(result);
}
else
{
  // .GetResult() won't immediately return because the task isn't yet complete
  awaiter.OnCompleted(() =>
  {
    // This code will be executed when the task is completed
    var result = awaiter.GetResult(); // _Now_ it will immediately return (or throw an exception)
    handleTheResult(result);
  });
}
```

...and the compiler automatically does that for you with a state machine when
you use the `await` keyword in an `async` context.

#### Weeds

If you want to learn how to tell C# to build the async state machine in a
specific way then take a look at C#'s
[task type documentation](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-7.0/task-types).

If you look at how the `Task<T>` class's state machine
[is constructed](https://referencesource.microsoft.com/#mscorlib/system/runtime/compilerservices/AsyncMethodBuilder.cs,5916df9e324fc0a1)
then you'll see that it's a bit different than my above pseudocode. Among other
things, it captures the current `SynchronizationContext` and `ExecutionContext`
so that continuations can be executed on them.

The fact that `SynchronizationContext` and `ExecutionContext` are _static_ state
in C# was significant to the Rust language team when they decided how to design
their `Future` trait. Rustaceans don't like static state. Initial
implementations of futures in Rust relied on thread local storage, but that was
a dead end because thread local storage isn't available in `[no_std]` Rust code.

Okay let's quit wandering around in these weeds and see if we can make some more
progress.

### How to create your own awaitable expressions

You don't have to rely only on `Task`, `ValueTask`, or any of the other
framework-provided awaitable types. You can create your own
[awaitable type](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#awaitable-expressions).

Basically it comes down to implementing this interface on a thing:

```csharp
using System.Runtime.CompilerServices;

public interface IAwaiter<out T> : INotifyCompletion
{
    bool IsCompleted { get; }

    T GetResult();
}
```

...then returning that thing from your awaitable type's `GetAwaiter()` method.

That interface doesn't exist in the .Net framework. You can create it if you
want, but any type that has that signature&mdash;implement `INotifyCompletion`,
have a `bool IsCompleted` property, have a `T GetResult()` method&mdash;will
work as an awaiter. And any type that returns an awaiter from a `GetAwaiter()`
method can be awaited.

Like this:

```csharp
class MyAwaitable<T>
{
  public MyAwaiter<T> GetAwaiter() => ...;
}

class MyAwaiter<T> : IAwaiter<T>
{
  public bool IsCompleted => ...;

  public T GetResult() => ...;

  public void OnCompleted(Action continuation) => ...;
}

static class Foo
{
  public static async Task RunAsync()
  {
    Action<T> handleTheResult = ...;
    Awaitable<int> awaitable = ...;
    int result = await awaitable; // You can await it!
    handleTheResult(result);
  }
}
```

Recall from the previous section how the compiler will create a state machine
that essentially does this with your awaitable type:

```csharp
// Almost the same as the RunAsync method above:
Action<int> handleTheResult = ...;
Awaitable<int> awaitable = ...;

var awaiter = awaitable.GetAwaiter();
if (awaiter.IsCompleted)
{
  var result = awaiter.GetResult(); // Will return immediately; might throw an exception
  handleTheResult(result);
}
else
{
  // .GetResult() won't immediately return because the task isn't yet complete
  awaiter.OnCompleted(() =>
  {
    // This code will be executed when the task is completed
    var result = awaiter.GetResult(); // _Now_ it will immediately return (or throw an exception)
    handleTheResult(result);
  });
}
```

You should now understand how you need to implement your awaitable type. The
awaiter type returned from `GetAwaiter()` needs to do these things:

 * Return `true` from `IsCompleted` when the asynchronous work is completed
 * Accept a continuation/callback passed to the `OnCompleted` method. This
   callback should be executed when the asynchronous work is completed. Keep in
   mind a callback won't be passed to `OnCompleted` if `IsCompleted` returned
   true when it was checked
 * Synchronously return the result or throw an exception from the `GetResult`
   method. Some implementations will choose to make `GetResult` synchronously
   block if the asynchronous work isn't yet complete

[The documentation](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#awaitable-expressions)
explains how to create an awaitable type that returns nothing/void. Basically,
`GetResult` should return `void` instead of `T`. If only C# would let us use
`void` as a generic type parameter!

### How to compose awaitable expressions

So how do you compose (as in combine) awaitable expressions? You now have
everything you need to answer this question.

As I showed in the "How the `await` keyword works" section above, the C#
compiler transforms code like this:

```csharp
async Task<T> FooAsync()
{
  // Synchronous block 1
  var a = GetA();
  var b = GetB(a);

  // await keyword
  var c = await GetCAsync(b);

  // Synchronous block 2
  var d = GetD(c);
  var e = GetE(d);

  // await keyword
  var f = await GetFAsync(e);

  return f;
}
```

...into a state machine.

The state machine has one step for each synchronous code block. The steps are
wrapped up as continuations/callbacks which get passed to the various awaiters
along the way. The awaiters decide when to execute the callbacks.

You can do the same thing by hand if you wish.

So the answer is: in C#, awaitable expressions are composed _with callbacks_.

### Why heap allocations and dynamic dispatch are inevitable when you compose awaitable expressions

You should now see why heap allocations and dynamic dispatch are inevitable in
C#'s async/await system: C# awaitable expressions are composed with callbacks.
These callbacks have the type `Action`, which is a delegate.

To package something (e.g. a step in a state machine) up into an `Action` you
have to put something on the heap.

And invoking an `Action` is dynamic dispatch.

#### But what about `ValueTask`?

I can just hear it now. You're wondering if this applies to `ValueTask`. After
all, Microsoft created `ValueTask` expressly
[so that](https://devblogs.microsoft.com/dotnet/understanding-the-whys-whats-and-whens-of-valuetask/#:~:text=nothing%20need%20be%20allocated%3A%20we%20can%20simply%20initialize%20this%20valuetask%3Ctresult%3E%20struct%20with%20the%20tresult%20and%20return%20that):

> nothing need be allocated [in certain cases]: we can simply initialize this
  `ValueTask<TResult>` struct with the `TResult` and return that.

But keep reading. That only applies to the _synchronous_ case, when the result
of an asynchronous method is already known and can be returned synchronously.

In other words, when the awaiter's `IsCompleted` property returns true then its
`GetResult` method is immediately/synchronously invoked and the result is
immediately available. There's no need to instantiate an `Action` to pass to its
`OnCompleted` method.

But when the result isn't available synchronously: heap allocation and dynamic
dispatch.

Now, `ValueTask` has a trick up its sleeve. You can create a `ValueTask` from an
[`IValueTaskSource`](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.sources.ivaluetasksource?view=net-5.0).
That does help minimize heap allocations in certain cases, but it's kind of
complicated to implement that interface. And I think that implementations
necessarily come with compromises.

For example: when you create a `ValueTask` with an `IValueTaskSource` then you
create it
[with a token](https://docs.microsoft.com/en-us/dotnet/api/system.threading.tasks.valuetask.-ctor?view=net-5.0#System_Threading_Tasks_ValueTask__ctor_System_Threading_Tasks_Sources_IValueTaskSource_System_Int16_).
And that token is only a 16-bit integer. You can only have so many `ValueTask`s
in play at once for a given `IValueTaskSource`. And how do you know when a
`ValueTask` should be taken out of play and its token recycled? If I have a
`ValueTask` then what's to keep me from continuing to call
`valueTask.GetAwaiter().GetResult()` on it over and over? There is nothing built
into `ValueTask` or `IValueTaskSource` to let you know when the token can be
recycled. You can only recycle the tokens if you have full control over the
`ValueTask`s that are created with your `IValueTaskSource`. That limits the
usefulness of this trick.

And again, at the end of the day, even with `ValueTask` and all its tricks, if
the awaiter returns `false` from `IsCompleted` then there is going to be at
least one heap allocation and dynamic dispatch.

This is why the Rust language devs
[say](http://aturon.github.io/tech/2016/09/07/futures-design/#:~:text=we%20were%20unable%20to%20make%20the%20%E2%80%9Cstandard%E2%80%9D%20future%20abstraction%20provide%20zero-cost%20composition%20of%20futures%2C%20and%20we%20know%20of%20no%20%E2%80%9Cstandard%E2%80%9D%20implementation%20that%20does%20so.):

> We were unable to make the “standard” future abstraction provide zero-cost
  composition of futures, and we know of no “standard” implementation that does
  so.

## Shallow dive&mdash;Rust

Let's turn our attention to Rust.

I'm less familiar with Rust so I won't be able to provide as much detail. But I
think you'll still see why Rust futures are better.

### How the `await` keyword works

Actually I don't entirely know how the `await` keyword works in Rust. I'm only
able to find documentation on _what_ it does.

My best guess is the Rust compiler generates a state machine similar to how the
C# compiler does it.

But keep in mind that in Rust, closures are strongly typed, can be statically
dispatched, and can live on the stack. So if async blocks are broken up into
various steps in a state machine, that state machine and all its steps will be
transparent to the compiler. And its steps can be statically dispatched without
heap allocations.

However it works, the Rust language devs are clear that async/await is a
zero-cost abstraction. So dynamic dispatch and heap allocation are definitely
_not_ necessary.

### How to create your own awaitable expressions

In Rust it comes down to implementing
[the `Future` trait](https://doc.rust-lang.org/core/future/trait.Future.html):

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

For example:

```rust
struct OneShotTimer {
  bool completed,
  Option<Waker> waker,
}

impl Future for OneShotTimer {
  type Output = ();

  fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
    match self.completed {
      true => Poll::Ready(()),
      false => {
        self.waker = Some(cx.waker().clone())
      }
    }
  }
}

impl OneShotTimer {
  pub fn schedule(...) -> Self {
    // Do stuff like wire up the .waker to be notified when the timer goes off
    ...
  }
}
```

See?

Of course there very well might be heap allocations in _scheduling_ this
particular implementation of a timer. But that's up to you. It's not forced upon
you by the async/await system.

And there are no heap allocations in polling this future. Even when it's not yet
ready nothing needs to be placed on the heap. Sure there is a `clone()` call in
there, but that places something into memory that has already been reserved
within the `OneShotTimer`. This `OneShotTimer` might happily live on the stack
and you could call `poll()` in a `[no_std]` environment.

#### A quick note about what polling is not

You don't have to always continually poll every single future. Rust's futures
tell you when they're ready to be polled again. And you know which one told you
because either you or your task executor gave it a specific `Waker`, and you
did remember to create that `Waker` in a way that associates it with the
currently executing task, right? So you don't have to poll _all_ the futures.

Here is what
[the documentation](https://doc.rust-lang.org/core/future/trait.Future.html#runtime-characteristics)
says:

> The `poll` function is not called repeatedly in a tight loop – instead, it
  should only be called when the future indicates that it is ready to make
  progress (by calling `wake()`). If you’re familiar with the `poll(2)` or
  `select(2)` syscalls on Unix it’s worth noting that futures typically do _not_
  suffer the same problems of “all wakeups must poll all events”; they are more
  like `epoll(4)`.

### How to compose awaitable expressions

Here's an example of joining two futures into a single future that returns the
result of both of them only once they've completed:

```rust
enum FutureOrResult<T : Future> {
  Future(T),
  Result(T::Output),
  None,
}

struct Joined<A : Future, B : Future>(FutureOrResult<A>, B, Option<Waker>);

impl<A : Future, B : Future> Future for Joined<A, B> {
  type Output = (A::Output, B::Output);

  fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
    match self.0 {
      FutureOrResult::Future(future_a) => match future_a.poll(cx) {
        Poll::Ready(result_a) => match self.1.poll(cx) {
          Poll::Ready(result_b) => {
            self.0 = FutureOrResult::None;
            Poll::Ready((result_a, result_b))
          },
          Poll::Pending => {
            self.0 = FutureOrResult::Result(result_a);
            Poll::Pending
          }
        },
        Poll::Pending => Poll::Pending
      },
      FutureOrResult::Result(result_a) => match self.1.poll(cx) {
        Poll::Ready(result_b) => {
          self.0 = FutureOrResult::None;
          Poll::Ready((result_a, result_b))
        },
        Poll::Pending => {
          Poll::Pending
        }
      },
      FutureOrResult::None => panic!("You called poll too many times")
    }
  }
}
```

Maybe there's a more idiomatic way, I dunno. But do you see any heap allocations
anywhere in this composition of two futures?

Do you see how instead of being composed with callbacks and dynamic dispatch,
Rust futures are composed _with static dispatch_?

### Why heap allocations and dynamic dispatch are avoidable when you compose futures

It's redundant to say this again, but the `Future::poll()` method does not
require heap allocation or dynamic dispatch. Instead it can be statically
dispatched on things on the stack.

## Can async/await in C# be like Rust?

What's to keep you from implementing your own `IFuture` interface in C# and
making your own async/await framework?

Almost nothing. Here's an attempt in C# that I think would get most of the way
there. But Rust's ownership and lifetime system is really excellent, and I think
this example suffers without it:

```csharp
public readonly struct Context
{
  // Include a Waker here
}

public readonly struct Maybe<T>
{
    public static readonly Maybe<T> None = default;

    readonly T _value;

    public Maybe(T value)
    {
        HasValue = true;
        _value = value;
    }

    public bool HasValue { get; }

    public T Value => HasValue
        ? _value
        : throw new Exception();
}

public interface IFuture<T>
{
    Maybe<T> Poll(in Context context);
}

public struct Joined<TA, TAFuture, TB, TBFuture> : IFuture<(TA, TB)>
    where TAFuture : IFuture<TA>
    where TBFuture : IFuture<TB>
{
    bool _hasFutureA;
    TAFuture _futureA;
    TA _valueA;
    bool _hasFutureB;
    TBFuture _futureB;
    TB _valueB;

    public Joined(TAFuture futureA, TBFuture futureB)
    {
        _hasFutureA = true;
        _hasFutureB = true;
        _futureA = futureA;
        _futureB = futureB;
        _valueA = default!;
        _valueB = default!;
    }

    public Maybe<(TA, TB)> Poll(in Context context)
    {
        TA valueA = default!;
        TB valueB = default!;
        var hasBoth = true;

        if (_hasFutureA)
        {
            var pollResult = _futureA.Poll(in context);
            if (pollResult.HasValue)
            {
                valueA = _valueA = pollResult.Value;
                _hasFutureA = false;
                _futureA = default!;
            }
            else
            {
                hasBoth = false;
            }
        }
        else
        {
            valueA = _valueA;
        }

        if (_hasFutureB)
        {
            var pollResult = _futureB.Poll(in context);
            if (pollResult.HasValue)
            {
                valueB = _valueB = pollResult.Value;
                _hasFutureB = false;
                _futureB = default!;
            }
            else
            {
                hasBoth = false;
            }
        }
        else
        {
            valueB = _valueB;
        }

        return hasBoth
            ? new Maybe<(TA, TB)>((valueA, valueB))
            : Maybe<(TA, TB)>.None;
    }
}
```

There are neither heap allocations nor dynamic dispatch anywhere in sight.

You would have to be very careful with ownership and lifetimes of these structs,
though. I'm sure havoc would ensue the moment you clone a `Joined` struct and
`Poll()`'ed both instances.

But if you walked _very_ carefully then I'm sure it could work.

## The point

You can compose Rust futures without heap allocations or dynamic dispatch.
That's not true of C#'s tasks (or any other possible awaitable type).

Rust's heapless composition power comes from the fact that Rust's futures are
polled. Because they are polled they can be composed with static dispatch
instead of callbacks.

Polled futures are great!