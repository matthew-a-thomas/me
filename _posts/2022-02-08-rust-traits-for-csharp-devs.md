---
title: Rust Traits
description: Explained for C# developers
category: programming
---

The [Rust](https://www.rust-lang.org/) programming language relies heavily on
_traits_. But what are they? How do they compare to C#'s interfaces?

In a nutshell, a trait is like a C# interface:
 * Multiple types can implement it
 * You can write code that abstracts over it
 * You can do dynamic dispatch with it
 * You can write default methods for it

...but it is also _not_ like a C# interface:
 * You can implement a trait for a foreign type
 * A trait can have associated types

Let's look at each of these in turn.

## Like a C# interface

The similarities are simple. So I'll go quick.

### Multiple types can implement it

In C#, multiple types can implement an interface:

```csharp
interface IFoo
{
  void Bar();
}

struct ThingA : IFoo
{
  public void Bar()
  {
    Console.WriteLine("Thing A");
  }
}

struct ThingB : IFoo
{
  public void Bar()
  {
    Console.WriteLine("Thing B");
  }
}
```

The same is true of traits in Rust:

```rust
trait Foo {
  fn bar(&self);
}

struct ThingA {}

impl Foo for ThingA {
  fn bar(&self) {
    println!("Thing A");
  }
}

struct ThingB {}

impl Foo for ThingB {
  fn bar(&self) {
    println!("Thing B");
  }
}
```

### You can write code that abstracts over it

In C# you can abstract over an interface:

```csharp
interface IBinaryOperator
{
  double Operate(double a, double b);
}

struct Add : IBinaryOperator
{
  public double Operate(double a, double b) => a + b;
}

struct Multiply : IBinaryOperator
{
  public double Operate(double a, double b) => a * b;
}

static class Example
{
  static double Apply<T>(T op, double a, double b)
  where T : IBinaryOperator =>
  op.Operate(a, b);
}
```

And the same is true of Rust's traits:

```rust
trait BinaryOperator {
  fn operate(&self, a: f64, b: f64) -> f64;
}

struct Add {}

impl BinaryOperator for Add {
  fn operate(&self, a: f64, b: f64) -> f64 {
    a + b
  }
}

struct Multiply {}

impl BinaryOperator for Multiply {
  fn operate(&self, a: f64, b: f64) -> f64 {
    a * b
  }
}

fn apply<T: BinaryOperator>(op: &T, a: f64, b: f64) -> f64 {
  op.operate(a, b)
}
```

### You can do dynamic dispatch with it

Dynamic dispatch happens so often in C# that we usually don't notice it. But
here's an example of dynamic vs static dispatch in action:

```csharp
interface IFoo
{
  void Bar();
}

static class Example
{
  static void DynamicDispatch(IFoo foo)
  {
    foo.bar();
  }

  static void StaticDispatch<T>(T foo)
  where T : IFoo
  {
    foo.bar();
  }
}
```

Here's the same picture in Rust:

```rust
trait Foo {
  fn bar(&self);
}

fn dynamic_dispatch(foo: &dyn Foo) {
  foo.bar();
}

fn static_dispatch<T: Foo>(foo: &T) {
  foo.bar();
}
```

### You can write default methods for it

One of the recent versions of C# added default interface implementations. I'm
writing this off the cuff, but I think it looks like this:

```csharp
interface IFoo
{
  void Bar();

  void Baz()
  {
    this.Bar();
  }
}

class Thing : IFoo
{
  public void Bar()
  {
    // This is the only method we're required to implement
  }
}
```

It's very similar in Rust:

```rust
trait Foo {
  fn bar(&self);

  fn baz(&self) {
    self.bar();
  }
}

struct Thing {}

impl Foo for Thing {
  fn bar(&self) {
    // This is the only method we're required to implement
  }
}
```

## Not like a C# interface

In my opinion this is where Rust's trait system really begins to shine.

### You can implement a trait for a foreign type

In C#, when you use a type from some package you found on Nuget or in the
framework (or anywhere for that matter), you're stuck with whatever interfaces
they decided to implement on that type.

```csharp
interface IReadableIndex<T>
{
  T this[int index] { get; }
}

static class Example
{
  static void DoImportantStuff<T>(IReadableIndex<T> indexable)
  {
    // Very important things happen here. We don't want to repeat ourselves so
    // we've encapsulated the behavior into this function and we're trying to
    // make it work for multiple types by abstracting with generics.
  }

  static void WontWork()
  {
    var array = new[] { 2, 3, 5, 7, 11 };
    DoImportantStuff(array); // Compile error: the array doesn't implement the interface!
    var span = array.AsSpan();
    DoImportantStuff(span); // Compile error: Span doesn't implement the interface!
    var memory = array.AsMemory();
    DoImportantStuff(memory); // Compile error: Memory doesn't implement the interface!
  }
}
```

The adapter pattern will come to your rescue:

```csharp
class ArrayReadableIndexAdapter<T> : IReadableIndex<T>
{
  readonly T[] _array;

  public ArrayReadableIndexAdapter(T[] array)
  {
    _array = array;
  }

  public T this[int index] => _array[index];
}

ref struct SpanReadableIndexAdapter<T> : IReadableIndex<T>
{
  readonly Span<T> _span;

  public SpanReadableIndexAdapter(Span<T> span)
  {
    _span = span;
  }

  public T this[int index] => _span[index];
}

struct MemoryReadableIndexAdapter<T> : IReadableIndex<T>
{
  readonly Memory<T> _memory;

  public MemoryReadableIndexAdapter(Memory<T> memory)
  {
    _memory = memory;
  }

  public T this[int index] => _memory.Span[index];
}

static class Example
{
  static void DoImportantStuff<T>(IReadableIndex<T> indexable)
  {
    // Same important stuff as before
  }

  static void WillWork()
  {
    var array = new[] { 2, 3, 5, 7, 11 };
    DoImportantStuff(new ArrayReadableIndexAdapter(array));
    var span = array.AsSpan();
    DoImportantStuff(new SpanReadableIndexAdapter(span));
    var memory = array.AsMemory();
    DoImportantStuff(new MemoryReadableIndexAdapter(memory));
  }
}
```

But the adapter pattern comes with a cost. Do you see all the boilerplate? We
were trying to follow the [Interface Segregation Principle](https://en.wikipedia.org/wiki/Interface_segregation_principle)
which says:
> ISP splits interfaces that are very large into smaller and more specific ones
  so that **clients will only have to know about the methods that are of
  interest to them**.

Indeed we were following it, because the `IReadableIndex<T>` interface exposes
the absolute minimum surface area needed for our "important stuff" method. But
to do so we had to introduce a _lot_ of boilerplate!

Rust makes this much easier!

```rust
trait Index<Idx> {
    type Output;
    fn index(&self, index: Idx) -> &Self::Output;
}

// Implement for arrays
impl<T, const N: usize> Index<i32> for [T; N] {
  type Output = T;
  fn index(&self, index: i32) -> &Self::Output {
    &self[index]
  }
}

// Implement for lists
impl<T> Index<i32> for Vec<T> {
  type Output = T;
  fn index(&self, index: i32) -> &Self::Output {
    &self[index]
  }
}

// etc

// Now just use the trait!
fn do_important_stuff<T: Index<i32>>(indexable: &T) {
  // Important code here
}
```

(The astute reader will notice that Rust already has [this trait](https://doc.rust-lang.org/core/ops/trait.Index.html))

Did you notice how it was possible to implement the trait for _foreign_ types?
It doesn't matter that we don't have access to the source code of `[T; N]` or
`Vec<T>`. We could still make those types implement this trait, and we didn't
have to introduce any additional types!

It is often much easier in Rust to _tailor interfaces to consumers_.

To me it seems Microsoft has fallen in love with a form of programming by
convention which they usually apply
[like this](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-8.0/ranges#implicit-range-support):
> The language will provide an instance indexer member with a single parameter
  of type `Range` for types which meet the following criteria:
> * The type is Countable.
> * The type has an accessible member named `Slice` which has two parameters of
    type `int`.
> * The type does not have an instance indexer which takes a single `Range` as
    the first parameter. The `Range` must be the only parameter or the remaining
    parameters must be optional.
>
> For such types, the language will bind as if there is an indexer member of the
  form `T this[Range range]` where `T` is the return type of the `Slice` method
  including any `ref` style annotations. The new member will also have matching
  accessibility with `Slice`.

Ignore the minutiae. The thing I want you to notice is how they didn't bother to
give us an interface for this new functionality. Instead they said "if you put
this here and name this other thing a certain way then the compiler will
magically do this". _They gave us no help with abstracting over this behavior in
C#._ If you wanted to write code that would work for "range-aware" types then
you'd have to create an interface and adapters for all the different types. (And
once you've done that then who cares about _implicit_ range support?)

The benefit that Rust's traits would bring to this situation is there would be
far less boilerplate and _no additional types_ needed to abstract over the very
same feature! Actually, they have already
[done](https://doc.rust-lang.org/core/ops/trait.Index.html#impl-Index%3CI%3E-1)
[this](https://doc.rust-lang.org/core/ops/struct.Range.html#impl-SliceIndex%3C%5BT%5D%3E).

### A trait can have associated types

In C#, an interface may have:
 * Methods
 * Properties (which are just syntax sugar for methods)

But in Rust, a trait may have:
 * Methods
 * Associated types

For example, how would you represent this in C#?

```rust
trait Add<Rhs> {
  type Output;
  fn add(&self, rhs: Rhs) -> Self::Output;
}

struct Foo {}

struct Bar {}

struct Baz {}

impl Add<Bar> for Foo {
  type Output = Baz;
  fn add(&self, rhs: Bar) -> Baz {
    // Make a baz from this foo and the given bar
  }
}

let foo = Foo {};
let bar = Bar {};
let baz: Baz = foo + bar;
```

I think in C# the interface would have to look like this:

```csharp
interface IAdd<TRight, TOut>
{
  TOut Add(TRight right);
}
```

But then you have a second generic type parameter that you have to carry
everywhere.

Rust's associated types are great for hiding generic types in certain
situations. You can even have associated _constants_ (although support for that
isn't yet complete).

---

_Caution: the code in this article was written off the cuff and on the fly. It
probably won't compile._