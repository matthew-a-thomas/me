---
title: ReentrantAsyncLock NuGet Package
category: programming
description: Introducing the ReentrantAsyncLock package
---

In the previous two posts I outlined the concept for a reentrant asynchronous
lock, and explained how it can provide all three of these things at once:

* Reentrance
* Asynchronicity
* Mutual exclusion

In this third post I'll introduce the ReentrantAsyncLock NuGet package which
gives you semantics that will look a little more normal. I think this third post
should take the place in your mind of the second one because the code here works
out some kinks that I inadvertently have there. I consider this NuGet package
the capstone of my efforts in this series.

NuGet package:<br/>
[https://www.nuget.org/packages/ReentrantAsyncLock](https://www.nuget.org/packages/ReentrantAsyncLock)

Source code:<br/>
[https://github.com/matthew-a-thomas/cs-reentrant-async-lock](https://github.com/matthew-a-thomas/cs-reentrant-async-lock)

# The NuGet package and its semantics

The
[ReentrantAsyncLock NuGet package](https://www.nuget.org/packages/ReentrantAsyncLock)
lets you write code like this:

```csharp
var asyncLock = new ReentrantAsyncLock();
var raceCondition = 0;
// You can acquire the lock asynchronously
await using (await asyncLock.LockAsync(CancellationToken.None))
{
    await Task.WhenAll(
        Task.Run(async () =>
        {
            // The lock is reentrant
            await using (await asyncLock.LockAsync(CancellationToken.None))
            {
                // The lock provides mutual exclusion
                raceCondition++;
            }
        }),
        Task.Run(async () =>
        {
            await using (await asyncLock.LockAsync(CancellationToken.None))
            {
                raceCondition++;
            }
        })
    );
}
Assert.Equal(2, raceCondition);
```

In the code comments above I point out the three different aspects of the lock.

You'll also notice that the NuGet package source code has automated tests that
assert the correctness of each of the three aspects (and more).

# How does it work?

It combines `ExecutionContext`/`AsyncLocal` with a special
`SynchronizationContext` and a special awaitable type.

## ExecutionContext and AsyncLocal

If you're familiar with
[the ExecutionContext class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.executioncontext?view=net-6.0)
then you'll know that it "flows" downward through async calls. And it carries
some stuff with it. In particular, it carries the values of
[AsyncLocal](https://docs.microsoft.com/en-us/dotnet/api/system.threading.asynclocal-1?view=net-6.0)
instances.

One such instance holds a value that indicates an asynchronous scope. When you
acquire the lock then your scope is squirreled away as though to say "the lock
belongs to this scope". And then that scope flows downward through async calls.
That's what makes the lock reentrant.

If you're looking at the source code then check out
[the `ReentrantAsyncLock.LocalScope` property](https://github.com/matthew-a-thomas/cs-reentrant-async-lock/blob/deded4441ad895428dc3716852e5fb07c74036af/ReentrantAsyncLock/ReentrantAsyncLock.cs#L84).
It is backed by an `AsyncLocal` instance and stores a value that indicates an
asynchronous scope. When an object is assigned to this property then all nested
async calls also get that value.

Now notice
[the `ReentrantAsyncLock.TryLockImmediately` method](https://github.com/matthew-a-thomas/cs-reentrant-async-lock/blob/deded4441ad895428dc3716852e5fb07c74036af/ReentrantAsyncLock/ReentrantAsyncLock.cs#L149).
That method checks the `LocalScope` against
[the `_owningScope` field](https://github.com/matthew-a-thomas/cs-reentrant-async-lock/blob/deded4441ad895428dc3716852e5fb07c74036af/ReentrantAsyncLock/ReentrantAsyncLock.cs#L63).
When they match then the lock can be acquired because that's a case of
reentrance.

## SynchronizationContext

[The `SynchronizationContext` class](https://docs.microsoft.com/en-us/dotnet/api/system.threading.synchronizationcontext?view=net-6.0)
is Microsoft's abstraction of a synchronization model. It is (usually) the thing
in charge of deciding how asynchronous continuations should be executed.

For example, in WPF when you are executing asynchronous code on the UI thread
then you'll want to still be on the UI thread after an `await` call:

```csharp
partial class MyUserControl
{
  /* Notice this is an "async" method: */ async void OnButtonClick(object sender, EventArgs e)
  {
    await Task.Delay(TimeSpan.FromSeconds(5));
    DoSomethingSynchronousOnTheUIThread(); // <-- This needs to happen on the UI thread
  }
}
```

WPF has
[a special subclass of `SynchronizationContext`](https://docs.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatchersynchronizationcontext?view=windowsdesktop-6.0)
that enables this.

I do something similar in the `ReentrantAsyncLock` package. I subclassed
`SynchronizationContext` so that I could serialize continuations and execute
them one-at-a-time.

Check out
[my `WorkQueue` class](https://github.com/matthew-a-thomas/cs-reentrant-async-lock/blob/deded4441ad895428dc3716852e5fb07c74036af/ReentrantAsyncLock/WorkQueue.cs).
It's the same thing that I described in
[the first post]({% post_url 2022-06-14-reentrant-async-lock %}).

It's just a simple work queue. But that's what gives the lock mutual exclusion.

Recall how you acquire the lock:

```csharp
await asyncLock.LockAsync(token)
```

When you invoke `LockAsync` then you are immediately placed on that
SynchronizationContext. The compiler packages up the code after the `await` into
a continuation, and that continuation is given to the work queue. And since that
work queue will only do one thing at a time you get mutual exclusion.

## A special awaitable type

This brings us to
[the `AsyncLockResult` class](https://github.com/matthew-a-thomas/cs-reentrant-async-lock/blob/deded4441ad895428dc3716852e5fb07c74036af/ReentrantAsyncLock/AsyncLockResult.cs).
This is a special awaitable type and is the thing that lets you asynchronously
get the lock. Microsoft
[describes](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/expressions#11882-awaitable-expressions)
how to make an "awaitable" thing. `AsyncLockResult` follows those rules and so
you're allowed to "await" the thing returned from the `LockAsync` method.

`AsyncLockResult` is really the glue that holds everything together. There are a
couple of competing things going on and this class helps resolve them.

For example, I need to execute asynchronous continuations on a special
`SynchronizationContext`, but it's futile to change the current
`SynchronizationContext` within asynchronous code because the previous context
is restored as soon as execution leaves that context. So how do you change the
`SynchronizationContext` in the asynchronous code _outside of_ the `LockAsync`
method? The answer is to make the `LockAsync` method actually be _synchronous_
but return something that can be awaited&mdash;that something is an instance of
`AsyncLockResult`.

As another example, the `LockAsync` method takes a `CancellationToken`, meaning
"please stop trying to acquire the lock as soon as this token is canceled." But
what if the continuation (for the code following your call to `LockAsync`) has
already been posted to the work queue and the work queue is busy? Then you'll
cancel the `CancellationToken` and nothing will happen until the work queue gets
around to processing your continuation. So how do you safely post a continuation
(which by design is only allowed to be executed once) to the work queue _and_
call it when the `CancellationToken` is canceled? Again, the answer is "with the
`AsyncLockResult` class." It wraps the continuation in such a way that it can be
sent to both places at once but will only get executed a single time.

# The point

The
[ReentrantAsyncLock NuGet package](https://www.nuget.org/packages/ReentrantAsyncLock)
is an asynchronous lock that gives you all three of these things with nice
semantics:

* Reentrance
* Asynchronicity
* Mutual exclusion

Give it a try!