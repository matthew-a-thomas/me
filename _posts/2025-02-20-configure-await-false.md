---
title: ConfigureAwait(false)
description: And its impact on ReentrantAsyncLock
category: programming
---

`ConfigureAwait(false)` is common and recommended. But it can sometimes lead to problems with the `ReentrantAsyncLock` NuGet package. I'd like to break the issue down and talk about how to mitigate it.

# `ConfigureAwait(false)`

Sometimes you'll want to `await` a `Task` while on a special `SynchronizationContext`, but don't want to return to that special `SynchronizationContext` after the `await`. For example, imagine you've written the following WPF click handler that does something special:

```csharp
async void HandleButtonClick(object sender, EventArgs args)
{
    await DoSomethingSpecialAsync();
}
```

Click handlers in WPF run in a `DispatcherSynchronizationContext`. And in this case there's no point returning to the UI thread after `DoSomethingSpecialAsync()` completes, but that's just what will happen by default.

That's what `ConfigureAwait(false)` is for. `ConfigureAwait` is a method on `Task` that gives you some control over where the asynchronous continuation (the stuff after an `await`) runs. Let me explain with examples.

The following code prints "null". If you remove the `ConfigureAwait(false)` then it prints "FancySynchronizationContext":

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

SynchronizationContext.SetSynchronizationContext(new FancySynchronizationContext());
await Task.Delay(1).ConfigureAwait(false);
Console.WriteLine(
    SynchronizationContext.Current?.GetType().Name ?? "null"
);

sealed class FancySynchronizationContext : SynchronizationContext
{
    public override void Post(SendOrPostCallback d, object? state) => Task.Run(() =>
    {
        SetSynchronizationContext(this);
        d(state);
    });
}
```

So you can see that `ConfigureAwait(false)` wipes out the current `SynchronizationContext` for the asynchronous continuation.

# The ill effects on `ReentrantAsyncLock` of wiping out `SynchronizationContext`

As I explained in [a previous post in this series]({% link _posts/2022-06-20-introducing-reentrantasynclock.md %}), the `ReentrantAsyncLock` library uses a special `SynchronizationContext`. It is similar to WPF's `DispatcherSynchronizationContext` because it only executes one thing at a time&mdash;this is the foundation of the lock's mutual exclusion.

And so if you wipe out the current `SynchronizationContext` while inside the guarded section of a `ReentrantAsyncLock` then you risk losing the lock's guarantee of mutual exclusion.

Here is an example. This code chugs along forever, which is another demonstration that `ReentrantAsyncLock` provides mutual exclusion and reentrance at the same time:

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

var asyncLock = new ReentrantAsyncLock.ReentrantAsyncLock();
var mutualExclusionTest = 0;
for (long i = 1;; i++)
{
    await Task.WhenAll(
        Go(),
        Go(),
        Go(),
        Go(),
        Go()
    );
    if (mutualExclusionTest != 0)
    {
        Console.WriteLine($"There was a mutual exclusion violation on try {i:N0}");
        return;
    }

    if (i % 100_000 == 0 && i > 0)
    {
        Console.WriteLine($"So far so good after try {i:N0}");
    }
}

async Task Go()
{
    await using (await asyncLock.LockAsync(CancellationToken.None))
    {
        await Task.CompletedTask.ConfigureAwait(
            ConfigureAwaitOptions.ForceYielding
            | ConfigureAwaitOptions.ContinueOnCapturedContext // Remove this line to be the same as ConfigureAwait(false)
        );

        var reenterTask = Task.Run(async () =>
        {
            await using (await asyncLock.LockAsync(CancellationToken.None))
            {
                // If mutual exclusion is preserved then the following two lines will have no net effect
                mutualExclusionTest++;
                mutualExclusionTest--;
            }
        });

        // If mutual exclusion is preserved then the following two lines will have no net effect
        mutualExclusionTest++;
        mutualExclusionTest--;

        await reenterTask;
    }
}
```

However, if I remove the `| ConfigureAwaitOptions.ContinueOnCapturedContext` line from the `Go()` method then it becomes equivalent to `ConfigureAwait(false)`. And then the above code prints out "There was a mutual exclusion violation on try 146,747" (or some other number).

In other words, when you wipe out the current `SynchronizationContext` then it is no longer safe to reenter the `ReentrantAsyncLock`.

# It's fine if you don't reenter the lock

Note: _everything is fine as long as you don't reenter the lock_. For example, this code chugs along happily forever:

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

var asyncLock = new ReentrantAsyncLock.ReentrantAsyncLock();
var mutualExclusionTest = 0;
for (long i = 1;; i++)
{
    await Task.WhenAll(
        Go(),
        Go(),
        Go(),
        Go(),
        Go()
    );
    if (mutualExclusionTest != 0)
    {
        Console.WriteLine($"There was a mutual exclusion violation on try {i:N0}");
        return;
    }

    if (i % 100_000 == 0 && i > 0)
    {
        Console.WriteLine($"So far so good after try {i:N0}");
    }
}

async Task Go()
{
    await using (await asyncLock.LockAsync(CancellationToken.None))
    {
        // The following two lines have the same effect as .ConfigureAwait(false)
        SynchronizationContext.SetSynchronizationContext(null);
        await Task.Yield();

        // If mutual exclusion is preserved then the following two lines will have no net effect
        mutualExclusionTest++;
        mutualExclusionTest--;
    }
}
```

And so `ConfigureAwait(false)` (and its friends) does not immediately break the lock. Instead, it breaks the lock after it has been reentered.

# It's fine if you put the `ConfigureAwait(false)` inside an awaited method

Also note: _everything is fine if you put the `ConfigureAwait(false)` inside an awaited method_.

For example, this code chugs along happily:

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

var asyncLock = new ReentrantAsyncLock.ReentrantAsyncLock();
var mutualExclusionTest = 0;
for (long i = 1;; i++)
{
    await Task.WhenAll(
        Go(),
        Go(),
        Go(),
        Go(),
        Go()
    );
    if (mutualExclusionTest != 0)
    {
        Console.WriteLine($"There was a mutual exclusion violation on try {i:N0}");
        return;
    }

    if (i % 100_000 == 0 && i > 0)
    {
        Console.WriteLine($"So far so good after try {i:N0}");
    }
}

async Task Go()
{
    await using (await asyncLock.LockAsync(CancellationToken.None))
    {
        await WrappedConfigureAwaitFalse();

        var reenterTask = Task.Run(async () =>
        {
            await using (await asyncLock.LockAsync(CancellationToken.None))
            {
                // If mutual exclusion is preserved then the following two lines will have no net effect
                mutualExclusionTest++;
                mutualExclusionTest--;
            }
        });

        // If mutual exclusion is preserved then the following two lines will have no net effect
        mutualExclusionTest++;
        mutualExclusionTest--;

        await reenterTask;
    }
}

async Task WrappedConfigureAwaitFalse()
{
    await Task.Run(() => { }).ConfigureAwait(false);
}
```

# Why does this happen?

The three properties that `ReentrantAsyncLock` gives you are:
1. Asynchronicity
2. Mutual exclusion
3. Reentrance

Properties \#2 and \#3 come from an interplay between `ExecutionContext` and `SynchronizationContext`. The one allows the lock to be reentered by "descendent" asynchronous code while the other makes sure that bits of work can only happen one at a time. When the `SynchronizationContext` is lost then you break the interplay and you can no longer get all three properties at the same time.

# The point

As soon as you lose the special `SynchronizationContext` that `ReentrantAsyncLock` gives you, you should treat it as though you have left the guarded section of the lock.

This is why the instructions for `ReentrantAsyncLock` [advise against doing that](https://github.com/matthew-a-thomas/cs-reentrant-async-lock/tree/50d2632ac2903293bfadf4c4bf7d364ee960544c#dont-change-the-current-synchronizationcontext-once-youre-in-the-guarded-section).