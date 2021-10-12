---
title: Async Method Builders Are Hard
description: A peek under the hood of C#, and a complaint about Microsoft's documentation
category: programming
---

C# lets you use asynchronous methods and awaitable expressions. For example, in
C# you can do this:

```csharp
using System;
using System.Threading.Tasks;

static class Program
{
    static void Main()
    {
        var task = RunAsync();
        task.GetAwaiter().GetResult();
    }

    static async Task RunAsync()
    {
        Console.WriteLine("Just a second...");
        await Task.Delay(TimeSpan.FromSeconds(1));
        Console.WriteLine("Ok!");
    }
}
```

...and your program will output "Just a second...", then nothing will happen for
one second, then it'll output "Ok!".

Asynchronous methods are marked by the `async` keyword. Awaitable expressions
involve the `await` keyword.

But what types can an asynchronous method return?

Any awaitable type that has an asynchronous method builder.

## Builder type

Microsoft's documentation on asynchronous method builders pretty much is limited
to this:

[https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-7.0/task-types](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-7.0/task-types)

In summary, any type can be returned from an asynchronous method as long as it
is:

 * [An awaitable type](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#awaitable-expressions)
 * Decorated with the [`AsyncMethodBuilder`](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.asyncmethodbuilderattribute?view=net-5.0)
   attribute
 * And the attribute points to a [builder type](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-7.0/task-types#builder-type)

## ~~Documentation woes~~

~~But Microsoft's documentation isn't even correct.~~

~~Of the builder type [it says](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-7.0/task-types#:~:text=awaitunsafeoncompleted()%20should%20call%20awaiter.oncompleted(action)):~~

> ~~`AwaitUnsafeOnCompleted()` should call `awaiter.OnCompleted(action)`~~

~~That's not correct. It should call `awaiter.UnsafeOnCompleted(action)`. That's
what Microsoft itself [does](https://github.com/dotnet/runtime/blob/v5.0.10/src/libraries/System.Private.CoreLib/src/System/Runtime/CompilerServices/AsyncTaskMethodBuilderT.cs#L101).~~

~~Also, it says:~~

> ~~If the state machine is implemented as a `struct`, then
  `builder.SetStateMachine(stateMachine)` is called with a boxed instance of the
  state machine that the builder can cache if necessary.~~

~~And that's not correct, either. [See?](https://dotnetfiddle.net/bwBmFj) (Make
sure you run that code on your local PC; .NET Fiddle doesn't run it correctly.)~~

_Edit: These mistakes will be fixed once [PR 5253](https://github.com/dotnet/csharplang/pull/5253) goes live._

## Implementation woes

Can _you_ tell me why the below code is a perfectly fine example of an async
method builder in Debug mode or if `CustomAwaitableAsyncMethodBuilder` is
changed to a `class`, but otherwise will hang and never complete?

```csharp
#nullable enable
namespace BrokenAsyncMethodBuilder
{
    using System;
    using System.Diagnostics;
    using System.Runtime.CompilerServices;
    using System.Threading;
    using System.Threading.Tasks;

    [AsyncMethodBuilder(typeof(CustomAwaitableAsyncMethodBuilder<>))]
    public readonly struct CustomAwaitable<T>
    {
        readonly ValueTask<T> _valueTask;

        public CustomAwaitable(ValueTask<T> valueTask)
        {
            _valueTask = valueTask;
        }

        public ValueTaskAwaiter<T> GetAwaiter() => _valueTask.GetAwaiter();
    }

    public struct CustomAwaitableAsyncMethodBuilder<T>
    //     ^^^^^^ Only works if you change this to `class` or run in Debug mode
    {
        #region fields

        Exception? _exception;
        bool _hasResult;
        SpinLock _lock;
        T? _result;
        TaskCompletionSource<T>? _source;

        #endregion

        #region properties

        public CustomAwaitable<T> Task
        {
            get
            {
                var lockTaken = false;
                try
                {
                    _lock.Enter(ref lockTaken);
                    if (_exception is not null)
                        return new CustomAwaitable<T>(ValueTask.FromException<T>(_exception));
                    if (_hasResult)
                        return new CustomAwaitable<T>(ValueTask.FromResult(_result!));
                    return new CustomAwaitable<T>(
                        new ValueTask<T>(
                            (_source ??= new TaskCompletionSource<T>(TaskCreationOptions.RunContinuationsAsynchronously))
                            .Task
                        )
                    );
                }
                finally
                {
                    if (lockTaken)
                        _lock.Exit();
                }
            }
        }

        public void AwaitOnCompleted<TAwaiter, TStateMachine>(
            ref TAwaiter awaiter,
            ref TStateMachine stateMachine)
            where TAwaiter : INotifyCompletion
            where TStateMachine : IAsyncStateMachine =>
            awaiter.OnCompleted(stateMachine.MoveNext);

        public void AwaitUnsafeOnCompleted<TAwaiter, TStateMachine>(
            ref TAwaiter awaiter,
            ref TStateMachine stateMachine)
            where TAwaiter : ICriticalNotifyCompletion
            where TStateMachine : IAsyncStateMachine =>
            awaiter.UnsafeOnCompleted(stateMachine.MoveNext);

        #endregion

        #region methods

        public static CustomAwaitableAsyncMethodBuilder<T> Create() => new()
        {
            _lock = new SpinLock(Debugger.IsAttached)
        };

        public void SetException(Exception exception)
        {
            var lockTaken = false;
            try
            {
                _lock.Enter(ref lockTaken);
                if (Volatile.Read(ref _source) is {} source)
                {
                    source.TrySetException(exception);
                }
                else
                {
                    _exception = exception;
                }
            }
            finally
            {
                if (lockTaken)
                    _lock.Exit();
            }
        }

        public void SetResult(T result)
        {
            var lockTaken = false;
            try
            {
                _lock.Enter(ref lockTaken);
                if (Volatile.Read(ref _source) is {} source)
                {
                    source.TrySetResult(result);
                }
                else
                {
                    _result = result;
                    _hasResult = true;
                }
            }
            finally
            {
                if (lockTaken)
                    _lock.Exit();
            }
        }

        public void SetStateMachine(IAsyncStateMachine stateMachine) {}

        public void Start<TStateMachine>(ref TStateMachine stateMachine)
            where TStateMachine : IAsyncStateMachine => stateMachine.MoveNext();

        #endregion
    }

    class Program
    {
        static async Task Main()
        {
            var expected = Guid.NewGuid().ToString();
            async CustomAwaitable<string> GetValueAsync()
            {
                await Task.Yield();
                return expected;
            }

            var actual = await GetValueAsync();

            if (!ReferenceEquals(expected, actual))
                throw new Exception();

            Console.WriteLine("Done!");
        }
    }
}
```

[Here's the answer](https://stackoverflow.com/a/69397857/3063273). Guess how
long it took me to figure that one out? I was pondering this _for a day or so_
before I posted the question on StackOverflow. And then the only way I found out
was by using [ILSpy](https://github.com/icsharpcode/ILSpy) on the compiled .dll,
and then only by configuring ILSpy in a specific way.

How are people supposed to know this stuff?

### Inevitable heap allocations

Not long ago I wrote "Async and Await in C# vs Rust". In that article
[I pointed out]({% post_url 2021-09-16-async-await-csharp-rust %}#:~:text=async%2Fawait%20system%3A-,C%23%20awaitable%20expressions%20are,something%20on%20the%20heap.,-And%20invoking%20an):

> C# awaitable expressions are composed with callbacks. These callbacks have the
  type `Action`, which is a delegate. To package something (e.g. a step in a
  state machine) up into an `Action` you have to put something on the heap.

Well, the exact same thing rears its head in async method builders. Async method
builders themselves _must_ create a boxed reference to the async state machine
at some point. Otherwise you run into the above problem.

The StackOverflow answer I linked to above has enough detail to figure out why.
But basically it comes down to the fact that the async state machines are driven
forward by `Action` delegates. It's the other side of the same coin.

Not only is it impossible to compose awaitable expressions without heap
allocations, it's not even possible to have an `async` method at all without
heap allocations.

## The point

Microsoft's implementation of async/await has some flaws. You might not think
they're very serious, but I think you'll at least agree that C#'s async method
builders are hard to implement!