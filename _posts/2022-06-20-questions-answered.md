---
title: Questions Answered
category: programming
description: Answering some questions about ReentrantAsyncLock
async_series_ordinal: fourth
---

{% include async_lock_series.md %}

This series started because I needed a reentrant asynchronous lock for my job.
We were already using an asynchronous lock but I noticed it deadlocked when you
tried to reenter it. I thought that was silly so I set out to find one that
works. That's when I found Max Fedotov's article titled
"Reentrant (Recursive) Async Lock is Impossible in C#", along with
[his Reddit thread](https://www.reddit.com/r/dotnet/comments/rklmby/reentrant_recursive_async_lock_is_impossible_in_c/)
about the same.

I took that as a challenge and set out to prove him wrong. It's not personal, I
just like a good programming challenge. And it happens that the solution to this
one will satisfy a real world need in my job.

So after publishing
"[Reentrant Async Lock]({% post_url 2022-06-14-reentrant-async-lock %})" I
linked to it
[in Max's Reddit thread](https://www.reddit.com/r/dotnet/comments/rklmby/comment/icdr70a/?utm_source=reddit&utm_medium=web2x&context=3).

He gave it some thought and then replied with
[an update to his article](https://itnext.io/reentrant-recursive-async-lock-is-impossible-in-c-e9593f4aa38a#:~:text=article%20still%20stands.-,2,-Another%20challenge%20to)
and
[a reply on Reddit](https://www.reddit.com/r/dotnet/comments/rklmby/comment/icthvsy/?utm_source=reddit&utm_medium=web2x&context=3).

I really appreciate his thoughtful replies&mdash;something too often absent from
other people on the internet&mdash;and I'd like to address the concerns he
raised. I'll frame them as questions and then I'll answer them.

In summary, I think that
[the ReentrantAsyncLock NuGet package](https://www.nuget.org/packages/ReentrantAsyncLock)
correctly satisfies the need for a reentrant async lock. And while Max's
concerns do require consideration I don't think they are showstoppers.

# Is ConfigureAwait(false) a problem?

When you await a `Task` in C#, you can configure it to take the code after the
await and run it on the thread pool synchronization context instead of returning
to whatever context you were on to begin with.

This can be very handy. And as Max pointed out this is sometimes even
recommended. He points to
[an article written by Stephen Toub](https://devblogs.microsoft.com/dotnet/configureawait-faq/)
and to
[an article written by Stephen Cleary](https://blog.stephencleary.com/2012/02/async-and-await.html#avoiding-context).
If you've been around C# long enough then you'll recognize both of those names
as heavy hitters. Both Stephens have made significant contributions to the .Net
world and when they say something a lot of people listen. So let's consider what
they say.

First let me give an example of when `ConfigureAwait(false)` is a good thing.

Pretend that you have written the following library for the world to use. It's
really useful so a lot of people use your special function. A lot of CPU time
the world over is being spent on your asynchronous function, but burning
dinosaurs isn't your hobby so you'd like to save some electricity and make it
faster:

```csharp
class MyLibrary
{
  public async Task DoSpecialStuffAsync(CancellationToken token)
  {
    await DoFirstThingAsync(token);
    await DoSecondThingAsync(token);
    await DoLastThingAsync(token);
  }
}
```

One way you can do that is by using `ConfigureAwait(false)`:

```csharp
class MyLibrary
{
  public async Task DoSpecialStuffAsync(CancellationToken token)
  {
    await DoFirstThingAsync(token).ConfigureAwait(false);
    await DoSecondThingAsync(token).ConfigureAwait(false);
    await DoLastThingAsync(token).ConfigureAwait(false);
  }
}
```

But _should_ you do that?

Well, it depends on what you're doing. There is no hard-and-fast rule, but it's
merely a performance optimization that you can use _if it makes sense_:

|What you're doing|Use `ConfigureAwait(false)`?|
|-|-|
|Writing application code|No (Toub)|
|Writing framework code|Sometimes (Toub, Cleary)|
|Something that needs to preserve context|No (Toub, Cleary)|
|Eeking out every last CPU cycle|Maybe (Toub, Cleary)|
|Trying to avoid deadlocks|Maybe (Toub)|

As an aside, if you're trying to avoid deadlocks by using
`ConfigureAwait(false)` then you're probably doing it wrong. Why are you
synchronously blocking on a `Task` at all?

But here's the thing: both Stephens agree if you need to preserve the context
then you should not use `ConfigureAwait(false)`. And in the case of
`ReentrantAsyncLock` that context is the very thing that makes it tick&mdash;you
must preserve it, so don't use `ConfigureAwait(false)`.

Here's an example of how `ConfigureAwait(false)` can mess up a
`ReentrantAsyncLock`:

```csharp
readonly ReentrantAsyncLock _asyncLock = new();

public async Task DoStuffAsync(CancellationToken token)
{
  await using (await _asyncLock.LockAsync(token))
  {
    DoNonThreadSafeStuff();
    await DoAsyncIOOperationAsync(token).ConfigureAwait(false);
    //                                  ^^^^^^^^^^^^^^^^^^^^^^
    DoNonThreadSafeStuff(); // <-- Uh oh!! Caused by this ^
  }
}
```

That second call to `DoNonThreadSafeStuff()` will **not** be guarded by the lock
because you escaped the special synchronization context that the lock uses to
guarantee mutual exclusion.

So when you're inside the guarded section of a `ReentrantAsyncLock` I think both
Stephens would say "don't use `ConfigureAwait(false)`."

Important!&mdash;it doesn't matter if `ConfigureAwait(false)` is used by
asynchronous methods that you call. Because when execution resumes after those
methods it'll resume back on the special synchronization context. For example:

```csharp
readonly ReentrantAsyncLock _asyncLock = new();

public async Task DoStuffAsync(CancellationToken token)
{
  await using (await _asyncLock.LockAsync(token))
  {
    DoNonThreadSafeStuff();
    await Task.Run(async () => await DoAsyncIOOperationAsync(token).ConfigureAwait(false));
    //                                                             ^^^^^^^^^^^^^^^^^^^^^^
    DoNonThreadSafeStuff(); // <-- This call is still guarded, even with this ^
  }
}
```

So you really only have to worry about the asynchronous code immediately within
the guarded section of the async lock. And if you're using the async lock then
that means you are also writing the code in that guarded section&mdash;nobody is
talking about some third party function "out there" that you can't control. So
just don't use `ConfigureAwait(false)` in the immediate guarded section and
you'll be fine.

# Can you synchronously block and wait for other tasks in the same synchronization context?

I think the question is if you can do this:

```csharp
readonly ReentrantAsyncLock _asyncLock = new();

public async Task DoSomething1Async(CancellationToken token)
{
  var task = DoSomething2Async(token);
  await using (await _asyncLock.LockAsync(token))
  {
    task.Wait();
  }
}

public async Task DoSomething2Async(CancellationToken token)
{
  var task = DoSomething1Async(token);
  await using (await _asyncLock.LockAsync(token))
  {
    task.Wait();
  }
}
```

The answer is no. That will deadlock.

But this isn't an issue with the async lock. The problem is you've written
terrible code that deadlocks. The potential for deadlocks is the reason it's a
code smell to `.Wait()` a `Task`.

Note: this deadlocks, too:

```csharp
readonly ReentrantAsyncLock _asyncLock = new();

public async Task DoSomething1Async(CancellationToken token)
{
  var task = DoSomething2Async(token);
  await using (await _asyncLock.LockAsync(token))
  {
    await task;
  }
}

public async Task DoSomething2Async(CancellationToken token)
{
  var task = DoSomething1Async(token);
  await using (await _asyncLock.LockAsync(token))
  {
    await task;
  }
}
```

To be fair, Max also doesn't think this is really an issue.

**Update**: Actually you'll get stack overflows from those examples. But pretend
for a moment that it deadlocks from the fact that one is waiting on the other
which is waiting on the first. Pretend we're talking about this instead:

```csharp
readonly ReentrantAsyncLock _asyncLock = new();

public async Task DeadlockAsync(CancellationToken token)
{
  var tcs = new TaskCompletionSource();
  var task1 = Task.Run(async () =>
  {
    await using (await _asyncLock.LockAsync(token))
    {
      await tcs.Task;
    }
  });
  var task2 = Task.Run(async () =>
  {
    await using (await _asyncLock.LockAsync(token))
    {
      tcs.TrySetResult();
    }
  });
  await task2;
  await task1;
}
```

In this example there is a race condition between `task1` and `task2`. If
`task2` wins the race then everything is hunky dory. But if `task1` wins the
race then there is a deadlock: `task1` will be waiting on the `tcs` which
can only be set by `task2`, but `task2` is waiting to acquire the lock and can't
until `task1` releases it.

Let me go into a little more detail about _why_ that would deadlock, and why I
think that's exactly what should happen.

The reason it deadlocks is because it's not an example of re-entering the lock.
Two **different** contexts are vying for the lock. Sometimes one of them gets it
and sometimes the other one gets it, but not both at the same time.

And they are two different contexts because they'll have _sibling_
`ExecutionContext` instead of one "inheriting" from the other.

I think deadlocking is the right thing. Think about the synchronous analogy:

```csharp
readonly object _gate = new();

public void Deadlock()
{
  var mre = new ManualResetEventSlim();
  var thread1 = new Thread(() =>
  {
    lock (_gate)
    {
      mre.Wait();
    }
  });
  thread1.Start();
  var thread2 = new Thread(() =>
  {
    lock (_gate)
    {
      mre.Set();
    }
  });
  thread2.Start();

  thread2.Join();
  thread1.Join();
}
```

I don't think anyone will complain about the deadlock in the synchronous
analogy. Instead I think they'll be content to learn that the problem is their
code :)

# Can't someone just replace SynchronizationContext.Current somewhere down the call chain inside the guarded section of an async lock?

I think the question here is if this has any effect on the performance of the
lock:

```csharp
readonly ReentrantAsyncLock _asyncLock = new();

public async Task DoSomethingAsync(CancellationToken token)
{
  await using (await _asyncLock.LockAsync(token))
  {
    await Task.Run(() => ChangeSynchronizationContext());
    DoNonThreadSafeStuff();
  }
}

void ChangeSynchronizationContext()
{
  SynchronizationContext.SetSynchronizationContext(null);
}
```

The answer is: no. When execution returns from the awaited `Task` then the
async state machine will have restored the synchronization context to what it
was before awaiting that `Task`. So in this case `DoNonThreadSafeStuff()` will
still be guarded by the lock.

But perhaps that's not what Max was getting at. Perhaps he meant this:

```csharp
readonly ReentrantAsyncLock _asyncLock = new();

public async Task DoSomethingAsync(CancellationToken token)
{
  await using (await _asyncLock.LockAsync(token))
  {
    ChangeSynchronizationContext();
    await Task.Yield();
    DoNonThreadSafeStuff();
  }
}

void ChangeSynchronizationContext()
{
  SynchronizationContext.SetSynchronizationContext(null);
}
```

The answer here is: yes, that'll cause `DoNonThreadSafeStuff()` to execute on
the thread pool. You'll have broken the lock.

So don't do that :)

If you're concerned about calling third party synchronous functions within the
guarded section of the async lock then you can always package them up into a
`Task.Run` and you will never have issues. Like this:

```csharp
readonly ReentrantAsyncLock _asyncLock = new();

public async Task DoSomethingAsync(CancellationToken token)
{
  await using (await _asyncLock.LockAsync(token))
  {
    await Task.Run(() => ExecuteStrangeThirdPartyFunction());
    DoNonThreadSafeStuff();
  }
}
```

# What about the synchronization context from which you enter the async lock?

Max said:

> There could be another `SynchronizationContext` already when you apply your
  lock, so you have to consider if you want to wrap it and post things onto it
  instead of posting them to the thread pool.

This is a valid concern. Let me illustrate with a pretend WPF example:

```csharp
readonly ReentrantAsyncLock _asyncLock = new();

// Event handler for the "Click" event on a button named "Button"
public async void OnButtonClick(object sender, EventArgs e)
{
  Button.Tag = "This works"; // This will work
  await using (await _asyncLock.LockAsync(default))
  {
    Button.Tag = "Uh oh!"; // This will throw an exception
  }
}
```

The above code illustrates that when a button's "Click" event handler executes
it'll do so on the UI thread. But then execution leaves that thread inside the
async lock. This change in threads might be unexpected to developers. The second
assignment to `Button.Tag` will throw an exception because in WPF that property
can only be assigned from the thread that is running that button's dispatcher.

What's the solution?

In this case you would have to do the following:
```csharp
readonly ReentrantAsyncLock _asyncLock = new();

public async void OnButtonClick(object sender, EventArgs e)
{
  Button.Tag = "This still works";
  await using (await _asyncLock.LockAsync(default))
  {
    await Button.Dispatcher.InvokeAsync(() => Button.Tag = "Now this works, too!"); // No more exception
  }
}
```

While it is not a showstopper, it is something you have to be conscious of. And
I think that's Max's point; entering the async lock changes some things that
don't usually change.

# Should the async lock have a synchronous locking API, too?

Max points out that many async lock implementations also have synchronous lock
methods. Then the async lock can be used in both synchronous and asynchronous
contexts.

Personally I don't think that's a good choice. In fact I would go so far as to
call it an anti-pattern. I personally think that a synchronous lock should be
used for synchronous contexts, and an asynchronous lock should be used for
asynchronous contexts. I suspect that if you want to use one for the other then
there are probably some things going amuck in your code. I think that if you're
okay with
[colored functions](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
then you should also be okay with colored locks.

But I'm writing this in a country where the First Amendment gives citizens the
right to freely express the following extension method, and where copyright law
won't hamper them because this whole site
[is MIT-licensed](https://github.com/matthew-a-thomas/me/blob/master/LICENSE):

```csharp
public static class ReentrantAsyncLockExtensions
{
    public static IDisposable LockSynchronously(this ReentrantAsyncLock asyncLock, CancellationToken cancellationToken)
    {
        var mre = new ManualResetEventSlim();
        var lockResult = asyncLock.LockAsync(cancellationToken);
        var awaiter = lockResult.GetAwaiter();
        awaiter.OnCompleted(mre.Set);
        mre.Wait(CancellationToken.None); // Token has already been given to LockAsync(...) above
        var asyncDisposable = awaiter.GetResult();
        return new Disposable(() =>
        {
            var _ = asyncDisposable.DisposeAsync();
        });
    }

    sealed class Disposable : IDisposable
    {
        Action? _dispose;

        public Disposable(Action? dispose)
        {
            _dispose = dispose;
        }

        public void Dispose() => Interlocked.Exchange(ref _dispose, null)?.Invoke();
    }
}
```

I _think_ that'll work. It takes into account some of the nuances of
`ReentrantAsyncLock`.

But no guarantees!

And I don't like it.

# The point

I think that `ReentrantAsyncLock` provides all the important aspects of an
asynchronous lock. It does sometimes require some careful thought when you use
it, but all of the rough spots that I know of can be addressed and aren't
showstoppers.

Feel free to [try it out](https://www.nuget.org/packages/ReentrantAsyncLock/)
and please
[report any bugs you find](https://github.com/matthew-a-thomas/cs-reentrant-async-lock/issues/new)!