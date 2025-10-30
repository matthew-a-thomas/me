---
title: Void in Dart
description: I can't decide if I like it or not
category: programming
---

Void is a common concept in programming. Functions that don't return anything will be annotated as returning void.

In C:

```c
void foo() {
  printf("This function returns nothing");
}
```

In C#:

```csharp
void Foo()
{
  Console.WriteLine("This function returns nothing");
}
```

Rust uses the empty tuple type:

```rust
fn foo() -> () {
  println!("This function returns nothing");
}
```

And here it is in Dart:

```dart
void foo() {
  print("This function returns nothing");
}
```

## As a generic type parameter

Generic types let you abstract a concept over all types.

But one of the rough edges of C# is the fact that it does not allow you to use `void` as a type parameter:

```csharp
class ThingOf<T> {}

void Foo() {
  var thing = new ThingOf<Void>(); // Error!
}
```

As a result, you have to have `Task` (without a type parameter) to complement `Task<T>`. And so on. It's really annoying.

I'm glad that Dart lets you use `void` as a type parameter:

```dart
Future<void> futureThatReturnsNothing = ...;
```

So Dart wins a point against C#.

## But what is `void`?!

Here's where `void` gets whacky in Dart:

```dart
void foo() {
  print("This function returns nothing");
  return null; // Just kidding!!! I'll return an instance of Null
}
```

_What?_

```dart
void main() {
  print(isItVoid() as Object?);
}

void isItVoid() => 42;
```

Output:

> 42

_What??_

It appears that `void` is the supertype of literally everything.

```dart
// No errors!
final void _ = null;
final void _ = Object() as Object?;
final void _ = "";
final void _ = 42;
final void _ = (() {});
```

_What???_

And yet it's also a subtype of `Object?` somehow?

```dart
// No errors!
final _ = testExtendsObject<void>();
void testExtendsObject<T extends Object?>() {}
```

How is that not a contradiction?

_What????_