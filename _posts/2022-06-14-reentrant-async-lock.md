---
title: Reentrant Async Lock
category: programming
description: A correct implementation
---

Max Fedotov
[wrote](https://itnext.io/reentrant-recursive-async-lock-is-impossible-in-c-e9593f4aa38a):
> If you need a reentrant async lock — you are out of luck and would have to
  get rid of lock reentry in your code-base instead.

I'm here to tell you that you are _not_ out of luck. You just need to try harder
;)

Here's how we'll have our cake and eat it too:

1. Make a custom `SynchronizationContext`
2. Make that `SynchronizationContext` awaitable
3. Use some special semantics

## Custom `SynchronizationContext`

First, implement a `SynchronizationContext` that executes its bits of work
one-at-a-time. Sort of like a `Dispatcher` in WPF.

Here's one I threw together:

```csharp
using System.Collections.Concurrent;

/// <summary>
/// A <see cref="SynchronizationContext"/> in which units of work are executed one-at-a-time on the thread pool.
/// </summary>
public sealed class WorkQueue : SynchronizationContext
{
    /// <summary>
    /// Exposes exceptions thrown on this <see cref="SynchronizationContext"/>.
    /// </summary>
    public event Action<Exception>? ExceptionThrown;

    readonly Queue<Entry> _entries = new();
    readonly object _gate = new();
    bool _isPumping;
    static readonly Action<object?> PumpDelegate;
    static readonly SendOrPostCallback SetManualResetEventSlimDelegate;
    static readonly ConcurrentBag<ManualResetEventSlim> UnusedManualResetEvents = new();

    static WorkQueue()
    {
        PumpDelegate = Pump;
        SetManualResetEventSlimDelegate = SetManualResetEventSlim;
    }

    /// <summary>
    /// Returns a new <see cref="WorkQueue"/>.
    /// </summary>
    public override SynchronizationContext CreateCopy() => new WorkQueue();

    public override void Post(SendOrPostCallback d, object? state)
    {
        lock (_gate)
        {
            _entries.Enqueue(new Entry(d, state));
            if (_isPumping)
                return;
            _isPumping = true;
        }
        ThreadPool.QueueUserWorkItem(PumpDelegate, this, false);
    }

    static void Pump(object? state)
    {
        var me = (WorkQueue)state!;
        while (true)
        {
            Entry entry;
            lock (me._gate)
            {
                if (!me._entries.TryDequeue(out entry))
                {
                    me._isPumping = false;
                    return;
                }
            }
            try
            {
                entry.Callback(entry.State);
            }
            catch (Exception e)
            {
                me.ExceptionThrown?.Invoke(e);
            }
        }
    }

    public override void Send(SendOrPostCallback d, object? state)
    {
        Post(d, state);
        if (!UnusedManualResetEvents.TryTake(out var mre))
            mre = new ManualResetEventSlim();
        Post(SetManualResetEventSlimDelegate, mre);
        mre.Wait();
        mre.Reset();
        UnusedManualResetEvents.Add(mre);
    }

    static void SetManualResetEventSlim(object? state)
    {
        var mre = (ManualResetEventSlim)state!;
        mre.Set();
    }

    record struct Entry(SendOrPostCallback Callback, object? State);
}
```

## Make it awaitable

I'll leave it as an exercise for the reader to figure out how to implement
`AwaitableSynchronizationContext`:

```csharp
public static class SynchronizationContextExtensions
{
    public static AwaitableSynchronizationContext RunBelow(this SynchronizationContext context) => new(context);
}
```

[Hint](https://www.google.com/search?q=c%23+await+anything).

Okay, here are some big clues... the awaiter returned from
`AwaitableSynchronizationContext.GetAwaiter()` needs to:

* Return `false` from `IsCompleted`
* Pass the continuation from `OnCompleted` to `SynchronizationContext.Post`

## Use special semantics

And now the grand finale...

When you want guarded access, just `await yourWorkQueue.RunBelow()`. It'll be
reentrant, asynchronous, and it provides mutual exclusion:

```csharp
[Fact]
public async Task SeeItIsPossible()
{
    var isExclusive = false;
    void ExclusivelyUse(object thing)
    {
        if (isExclusive)
            throw new Exception();
        isExclusive = true;
        try
        {
            Thread.Sleep(10);
        }
        finally
        {
            isExclusive = false;
        }
    }
    var nonThreadSafeResource = new object();
    var asyncGuard = new WorkQueue();
    var exceptions = new List<Exception>();
    asyncGuard.ExceptionThrown += exceptions.Add;
    async Task RecursivelyUseIt(int recursionLevel)
    {
        await asyncGuard.RunBelow();
        if (recursionLevel > 10)
            ExclusivelyUse(nonThreadSafeResource);
        else
            await RecursivelyUseIt(recursionLevel + 1);
    }
    await Task.WhenAll(
        RecursivelyUseIt(0),
        Task.Run(async () =>
        {
            for (var i = 0; i < 100; ++i)
            {
                await asyncGuard.RunBelow();
                ExclusivelyUse(nonThreadSafeResource);
            }
        })
    );
    Assert.Empty(exceptions);
}
```