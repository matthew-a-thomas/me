---
title: A More Ergonomic Async Lock (obsolete)
description: Making the work queue look more like an async lock
category: programming
async_series_ordinal: second
---

{% include async_lock_series.md %}

<div class="alert alert-warning">
{% markdown %}
## Warning

The code described in this post has some issues. For example, the
code below doesn't correctly handle cancellation. See
[the next post]({% post_url 2022-06-20-introducing-reentrantasynclock %}) for a
better implementation that is more thoroughly tested.
{% endmarkdown %}
</div>

I previously [described]({% post_url 2022-06-14-reentrant-async-lock %}) how to
make an async lock that supports all three of these at once:

* Reentrance
* Asynchronicity
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
    await using (await _asyncLock.EnterAsync(cancellationToken))
    {
        ExclusivelyUse(_resource);
    }
}
```

## The `EnterAsync` extension method

This extension method sets `SynchronizationContext.Current` to the given
`SynchronizationContext`, then asynchronously enters it. When the returned
`IAsyncDisposable` is disposed of then `SynchronizationContext.Current` will be
reset to whatever it was before.

```csharp
using System;
using System.Diagnostics;
using System.Runtime.CompilerServices;
using System.Threading;
using System.Threading.Tasks;

public static class SynchronizationContextExtensions
{
    static readonly Func<object?, IAsyncDisposable> CastToAsyncDisposable = state => (IAsyncDisposable)state!;
    static readonly Action DoNothing = () => {};

    static readonly ConditionalWeakTable<SynchronizationContext, TaskFactory> SynchronizationContextToTaskFactoryMap = new();

    public static Task<IAsyncDisposable> EnterAsync(
        this SynchronizationContext context,
        CancellationToken cancellationToken)
    {
        if (cancellationToken.IsCancellationRequested)
            return Task.FromCanceled<IAsyncDisposable>(cancellationToken);
        var previousContext = SynchronizationContext.Current;
        SynchronizationContext.SetSynchronizationContext(context);
        var contextFactory = GetOrCreateTaskFactory(context);
        var disposable = AsyncDisposable.Create(() =>
        {
            if (SynchronizationContext.Current == context)
                SynchronizationContext.SetSynchronizationContext(previousContext);
            var task = Task.Run(DoNothing, CancellationToken.None);
            return new ValueTask(task);
        });
        return EnterAsyncCore(contextFactory, disposable, cancellationToken);
    }

    static TaskFactory GetOrCreateTaskFactory(SynchronizationContext context)
    {
        Debug.Assert(SynchronizationContext.Current == context);
        if (SynchronizationContextToTaskFactoryMap.TryGetValue(context, out var factory))
            return factory;
        var scheduler = TaskScheduler.FromCurrentSynchronizationContext();
        factory = new TaskFactory(scheduler);
        SynchronizationContextToTaskFactoryMap.AddOrUpdate(context, factory);
        return factory;
    }

    static async Task<IAsyncDisposable> EnterAsyncCore(
        TaskFactory taskFactory,
        IAsyncDisposable disposable,
        CancellationToken cancellationToken)
    {
        try
        {
            return await taskFactory.StartNew(CastToAsyncDisposable, disposable, cancellationToken);
        }
        catch
        {
            var _ = disposable.DisposeAsync();
            throw;
        }
    }
}

public sealed class AsyncDisposable : IAsyncDisposable
{
    public static readonly AsyncDisposable Empty = new(null);

    Func<ValueTask>? _disposeAsync;

    AsyncDisposable(Func<ValueTask>? disposeAsync)
    {
        _disposeAsync = disposeAsync;
    }

    public static AsyncDisposable Create(Func<ValueTask> disposeAsync) => new(disposeAsync);

    public ValueTask DisposeAsync() => Interlocked.Exchange(ref _disposeAsync, null)?.Invoke() ?? default;
}
```

## A passing test case

The following test passes:

```csharp
[Fact]
public async Task EnterAsyncMethodShouldSwitchIntoAndOutOfGivenContext()
{
    SynchronizationContext.SetSynchronizationContext(null); // Necessary because xUnit's SynchronizationContexts like to waffle back and forth
    var newContext = new SimpleWorkQueue();
    await using (await newContext.EnterAsync(default))
    {
        Assert.Same(newContext, SynchronizationContext.Current);
    }
    Assert.Null(SynchronizationContext.Current);
    var isActuallyOutOfTheContext = false;
    newContext.Post(_ => isActuallyOutOfTheContext = true, null);
    var spinWait = new SpinWait();
    while (!isActuallyOutOfTheContext)
    {
        spinWait.SpinOnce();
    }
}
```