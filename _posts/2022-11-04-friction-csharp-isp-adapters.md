---
title: 'Friction: C# + ISP → Adapter Pattern'
description: As a student of Interface Segregation Principle, you'll find that C# forces you to make adapters
category: programming
---

I'd like to explain why you, the diligent student of Interface Segregation Principle (ISP), will find yourself making
many adapter classes in C#. I think C# adds unnecessary friction in this area.

**ISP** [states](https://en.wikipedia.org/wiki/Interface_segregation_principle):

> No code should be forced to depend on methods it does not use.

The intent of the **Adapter Pattern** [is](https://archive.org/details/designpatternsel00gamm/page/139):

> Convert the interface of a class into another interface clients expect.

## A `FillAsync` method

Let's start with a function that will asynchronously fill a buffer from a stream.

As you know,
[`Stream.ReadAsync`](https://learn.microsoft.com/en-us/dotnet/api/system.io.stream.readasync?view=net-6.0#system-io-stream-readasync(system-memory((system-byte))-system-threading-cancellationtoken))
looks like this:

```csharp
public ValueTask<int> ReadAsync (Memory<byte> buffer, CancellationToken cancellationToken)
```

That method promises to copy some bytes from the stream into your `buffer`. But it only returns the number of bytes that
it happened to copy, which "can be less than the number of bytes allocated in the buffer if that many bytes are not
currently available".

We want to completely _fill_ the buffer. So here's our method:

```csharp
public static async ValueTask FillAsync(
  Stream stream,
  Memory<byte> buffer,
  CancellationToken token)
{
  while (buffer.Length > 0)
  {
    var numBytes = await stream.ReadAsync(buffer, token);
    if (numBytes <= 0)
    {
      throw new Exception("The stream ended before the buffer could be filled");
    }
    buffer = buffer[numBytes..];
  }
}
```

## Interface Segregation Principle

Remember:

> No code should be forced to depend on methods it does not use.

Our `FillAsync` method has a "stream" parameter of type `Stream`. Which methods in the `Stream` type does `FillAsync` use? Only
`ReadAsync`:

> <code>await stream.<b><u>ReadAsync</u></b>(buffer, token)</code>

But what methods does it depend on? **All of them from the Stream class**. Your ISP alarm bell should be going off.

What's the solution? **Introduce a smaller interface**.

## A smaller interface

Here's an interface that only does precisely as much as `FillAsync` needs:

```csharp
interface IReadableStream
{
  ValueTask<int> ReadAsync(Memory<byte> buffer, CancellationToken token);
}
```

And here's the corresponding change to our `FillAsync` method:

```diff
 public static async ValueTask FillAsync(
-  Stream stream,
+  IReadableStream stream,
   Memory<byte> buffer,
   CancellationToken token)
 {
   while (buffer.Length > 0)
   {
     var numBytes = await stream.ReadAsync(buffer, token);
     if (numBytes <= 0)
     {
       throw new Exception("The stream ended before the buffer could be filled");
     }
     buffer = buffer[numBytes..];
   }
 }
```

But now `FillAsync` can no longer accept instances of `Stream`! What's the _only_ solution? **Adapter pattern**.

## Adapter pattern

Because we followed ISP, we now have to introduce an adapter class like this:

```csharp
public sealed class StreamToReadableStreamAdapter : IReadableStream
{
  readonly Stream _stream;

  public StreamToReadableStreamAdapter(Stream stream)
  {
    _stream = stream;
  }

  public ValueTask<int> ReadAsync(Memory<byte> buffer, CancellationToken token) => _stream.ReadAsync(buffer, token);
}
```

And now you can use `FillAsync` on instances of `Stream`:

```csharp
var stream = new MemoryStream();
var adapter = new StreamToReadableStreamAdapter(stream);
var buffer = new byte[42];
await FillAsync(adapter, buffer, CancellationToken.None);
```

## The point

It's too bad that we had to introduce a new type. This is such a common thing in well-designed software. I think in this
area C# adds friction to writing high quality software.