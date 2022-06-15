---
title: A More Ergonomic Async Lock
description: Making the work queue look more like an async lock
category: programming
async_series_ordinal: second
---

{% include async_lock_series.md %}

I previously [described]({% post_url 2022-06-14-reentrant-async-lock %}) how to
make an async lock that supports all three of these at once:

* Reentrance
* Asynchronousity
* Mutual exclusion

Now I'm going to show how to make it more ergonomic. With
[the `WorkQueue` class from that post]({% post_url 2022-06-14-reentrant-async-lock %})
and with the `EnterAsync` extension method shown below you'll be able to write
code like this:

```csharp
readonly WorkQueue _asyncLock = new();
readonly object _resource = new();

async Task DoItAsync(CancellationToken cancellationToken)
{
    using (await _asyncLock.EnterAsync(cancellationToken))
    {
        ExclusivelyUse(_resource);
    }
}
```

## The `EnterAsync` extension method

This extension method sets `SynchronizationContext.Current` to the given
`SynchronizationContext`, then asynchronously enters it. When the returned
`IDisposable` is disposed of then `SynchronizationContext.Current` will be reset
to whatever it was before.

```csharp
using System;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Threading.Tasks;

public static class SynchronizationContextExtensions
{
    static readonly Func<object?, IDisposable> CastToDisposable = state => (IDisposable)state!;

    static readonly ConditionalWeakTable<SynchronizationContext, TaskFactory> SynchronizationContextToTaskFactoryMap = new();

    public static Task<IDisposable> EnterAsync(
        this SynchronizationContext context,
        CancellationToken cancellationToken)
    {
        if (cancellationToken.IsCancellationRequested)
            return Task.FromCanceled<IDisposable>(cancellationToken);
        var previousContext = SynchronizationContext.Current;
        SynchronizationContext.SetSynchronizationContext(context);
        var disposable = Disposable.Create(() =>
        {
            if (SynchronizationContext.Current != context)
                return;
            SynchronizationContext.SetSynchronizationContext(previousContext);
        });
        if (!SynchronizationContextToTaskFactoryMap.TryGetValue(context, out var factory))
        {
            var scheduler = TaskScheduler.FromCurrentSynchronizationContext();
            factory = new TaskFactory(scheduler);
            SynchronizationContextToTaskFactoryMap.AddOrUpdate(context, factory);
        }
        return EnterAsyncCore(factory, disposable, cancellationToken);
    }

    static async Task<IDisposable> EnterAsyncCore(
        TaskFactory taskFactory,
        IDisposable disposable,
        CancellationToken cancellationToken)
    {
        try
        {
            return await taskFactory.StartNew(CastToDisposable, disposable, cancellationToken);
        }
        catch
        {
            disposable.Dispose();
            throw;
        }
    }
}

public sealed class Disposable : IDisposable
{
    public static readonly Disposable Empty = new(null);

    Action? _dispose;

    Disposable(Action? dispose)
    {
        _dispose = dispose;
    }

    public Disposable Create(Action dispose) => new(dispose);

    public void Dispose() => Interlocked.Exchange(ref _dispose, null)?.Invoke();
}
```