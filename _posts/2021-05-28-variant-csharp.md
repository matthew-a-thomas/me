---
title: A Variant Type in C#
category: programming
description: Or at least how to hack one together
---

Let's say your C# function needs to return either an "ok" value or an "error"
value depending on how things go within your function. The happy path will
return an integer, and if something goes wrong then you want to return a string
describing the error.

Let's give it a shot:

```csharp
enum Error
{
  NotEnoughNumbers,
  SomethingElse
}

struct Result<TOkay, TErr>
  where TOkay : struct
  where TErr : struct
{
  public TOkay? Ok;
  public TErr? Err;
}

Result<int, Error> GetNumber()
{
  // ...
  if (success)
    return new Result
    {
      Ok = 42
    };
  else
    return new Result
    {
      Err = enoughNumbers
        ? Error.SomethingElse
        : Error.NotEnoughNumbers
    };
}

void UseIt()
{
  var result = GetNumber();
  if (result.Ok is {} ok)
  {
    Console.WriteLine($"Yay! The number is {ok}");
  }
  else if (result.Err is {} err)
  {
    var message = err switch
    {
      Error.NotEnoughNumbers => "There weren't enough numbers",
      Error.SomethingElse => "Something else went wrong"
    };
    Console.WriteLine($"Uh oh! {message}");
  }
}
```

Okay. That works. But there are some pitfalls:

1. The compiler will let you create a `Result` having _both_ an `Ok` and an `Err`
  value, even though that doesn't make sense
  ```csharp
  var result = new Result<int, Error>
  {
    Ok = 42,
    Err = Error.SomethingElse
  };
  ```
2. The compiler will let you create a `Result` having _neither_ an `Ok` nor an
  `Err` value, which also doesn't make sense
  ```csharp
  var result = new Result<int, Error>();
  ```
3. Given a `Result` you can't know for certain whether it has an `Ok`, an `Err`,
  both, or neither
  ```csharp
  double ReturnSomething(Result<int, Error> result)
  {
    if (result.Ok is {} ok && result.Err is not {}) // This is long and hard to read
    {
      return ok * 42.0;
    }
    else if (result.Ok is {} ok && result.Err is {} err)
    {
      throw new Exception("???");
    }
    else if (result.Err is {} err)
    {
      // In this function we happen to know the correct thing to do with errors:
      return 42.0;
    }
    else
    {
      throw new Exception("???");
    }
  }
  ```

#1 could be fixed by giving `Result` constructors that only assign either an
`Ok` or an `Err` and not both. And that would help a lot with #3.

But #2 and #3 cannot be completely fixed because of how `struct` works in C#.
Specifically, a default constructor _always_ exists for a `struct`, cannot be
removed, and cannot be overridden.

So let's try making it a `class`:

```csharp
public class Result<TOk, TErr>
{
  readonly TErr _err;
  readonly bool _hasOk;
  readonly TOk _ok;

  private Result(TErr err, bool hasOk, TOk ok)
  {
    _err = err;
    _hasOk = hasOk;
    _ok = ok;
  }

  public static Result<TOk, TErr> FromOk(TOk value) =>
}
```