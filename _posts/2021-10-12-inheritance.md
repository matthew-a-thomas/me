---
title: Inheritance
description: Musings about favoring either inheritance or composition
category: programming
---

Inheritance is _subtyping_ plus _subclassing_. It is
**a domain-specific taxonomy with behavior and semantics**. And "semantics"
means dependencies, an interface, and a name. Inheritance is a kind of an "is a"
relationship, as in "a baby _is_ a person".

Here is a UML diagram of inheritance:

![UML diagram of inheritance](/assets/img/inheritance/uml.png){:class="img-fluid"}

## The good

When you need a domain-specific taxonomy with behavior and semantics, then there
really is nothing quite as suited to the task as inheritance.

## The bad

When you use inheritance you get _all_ of that. Even if you'd rather not have it
all.

And can you think of a single time you have _needed_ to glue both semantics and
behavior onto a taxonomy? Was it _impossible_ to accomplish that task without
inheritance?

The only times that come to mind for me are when inheritance was forced on me
by a design that already used and required it. I used to think inheritance was
the best solution to a lot of problems, but looking back I can tell that things
would have been better if done differently.

### Behavior vs semantics

Behavior is what the thing does. Semantics is what it looks like (its public
methods and properties).

What if you only want the semantics of a thing but not its behavior?

Here's an example of using inheritance to get the semantics of a thing when you
don't care about its behavior:

```csharp
// A thing that can drive on roads
abstract class Car
{
  public abstract double WheelSize { get; } // In inches

  public virtual void Drive(double speed)
  {
    Console.WriteLine($"We're driving {speed} MPH, which means the wheels are turning at {speed * 1056 / WheelSize / Math.PI / 2.0} RPM!");
  }

  public virtual void Honk()
  {
    Console.WriteLine("Honk!");
  }
}

// A thing that cars can drive on
class Road
{
  double _speedLimit;

  public Road(double speedLimit)
  {
    _speedLimit = speedLimit;
  }

  public virtual void Drive(Car car)
  {
    car.Drive(_speedLimit);
    car.Honk();
  }
}

// You can go fast on highways
class Highway : Road
{
  public Highway() : base(70.0)
  {}
}

// You have to go slow on driveways
class Driveway : Road
{
  public Driveway() : base(5.0)
  {}
}

// A taxiway is a road, is it not?
class Taxiway : Road
{
  public Taxiway() : base(30.0) // Speed limit for planes
  {}
}

// Planes need to operate on taxiways, so I guess it's a car
abstract class Plane : Car
{
  public override void Drive(double speed)
  {
    // Planes do not drive like normal cars
    if (speed < 50.0)
    {
      Console.WriteLine($"We're taxiing at {speed} MPH!");
    }
    else
    {
      Console.WriteLine($"We're flying at {speed} MPH!");
    }
  }

  public override void Honk()
  {
    // Planes do not have horns
  }
}
```

Notice how `Plane` is implemented. It uses _none_ of the behavior of the
underlying `Car` class. The only reason it inherits from `Car` is so that it
can operate on a `Taxiway` (which is a kind of road)&mdash;it needs the
semantics of a `Car`. But it gets all the baggage of a `Car` as well.

### Constructor parameters and other dependencies

What about constructor parameters?

They're part of the semantics, too&mdash;the constructor is often a
publicly-facing member.

Have you ever added a constructor parameter? Perhaps to refactor and encapsulate
some behavior after you realized it was repeated all over the place?

Inheritance hierarchies tend to get deeper over time. And the deeper it gets,
the harder it is to change constructor parameters.

And sometimes the behavior that uses the constructor parameter is totally
ignored in child classes&mdash;what a waste of effort!

Or how about refactoring a base class. Think about the effort that is required
to do that when there are hundreds of derived classes. You have to understand
_exactly_ what your change means in the context of every single one of those
classes. I've had to do that multiple times. It's no fun!

### Crossing domain boundaries

Let's continue the above example. Planes have to inherit from `Car` because they
operate on taxiways (which is a kind of road).

What about float planes?

[![A float plane](/assets/img/inheritance/DeHavilland_Single_Otter_Harbour_Air.jpg){:class="img-fluid"}](https://commons.wikimedia.org/wiki/File:DeHavilland_Single_Otter_Harbour_Air.jpg)

_Image from [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:DeHavilland_Single_Otter_Harbour_Air.jpg)_

That's definitely a plane. And it's operating on a waterway.

They can sometimes operate on land, too. See?

[![A float plane with wheels](/assets/img/inheritance/Viktoria Amphibs_landing_gear.jpg){:class="img-fluid"}](https://www.vikingair.com/twin-otter-series-400/twin-otter-answers/can-floatplane-land-ground)

_Image from [Viking Air](https://www.vikingair.com/twin-otter-series-400/twin-otter-answers/can-floatplane-land-ground)_

Okay, let's implement waterways and float planes:

```csharp
// A thing that can operate on waterways
class Watercraft
{
  ...
}

// It would be strange to call a waterway a "road", wouldn't it?
class Waterway
{
  public void Operate(Watercraft watercraft)
  {
    ...
  }
}

// A particular kind of watercraft. It's definitely not a car
class Speedboat : Watercraft
{
  ...
}

// A plane that can operate on the waterways
class FloatPlane : ?????
{
  ...
}
```

From what class should `FloatPlane` inherit?

If it inherits from `Plane` then it cannot operate on waterways. And if it
inherits from `Watercraft` then it cannot operate on taxiways. And C# won't let
you inherit from both.

Have you ever found yourself [wishing for multiple inheritance in C#](https://stackoverflow.com/search?q=c%23+multiple+inheritance)?
I bet you were trying to use inheritance to cross domains.

## The fixes

One fix that would enable you to cross between taxonomies would be to create
adapter classes that inherit from one taxonomy while delegating to an instance
of the other. But that's just composition in ugly disguise.

Which brings me to my favorite fix: **composition**.

When something is "composed" of other things, that means it has those other
things. Composition is a "has a" relationship. As in "a plane _has_ an engine".

I favor composition over inheritance. Why relegate composition to only those
times when you need to hack up an inheritance hierarchy?

### Behavior vs semantics

Do you want the semantics of being eligible to operate on a road?

```csharp
// Something that can operate on a road
interface IRoadWorthy
{
  void Operate(double speed);
}

// A road
sealed class Road
{
  readonly double _speed;

  public Road(double speed)
  {
    _speed = speed;
  }

  public void Operate(IRoadWorthy thing)
  {
    thing.Operate(_speed);
  }
}

// Creates highways
sealed class HighwayFactory
{
  public Road Create() => new Road(70.0);
}

// Creates driveways
sealed class DrivewayFactory
{
  public Road Create() => new Road(5.0);
}

// Creates taxiways
sealed class TaxiwayFactory
{
  public Road Create() => new Road(30.0);
}
```

Do you want the behavior of a car?

```csharp
sealed class Car
{
  public void Drive(double speed)
  {
    Console.WriteLine($"We're driving at {speed} MPH!");
  }

  public void Honk()
  {
    Console.WriteLine("Honk honk!");
  }
}
```

Do you want to be able to operate a car on a road in a particular way?

```csharp
sealed class DrivingHonkingCarToRoadWorthyAdapter : IRoadWorthy
{
  readonly Car _car;

  public DrivingHonkingCarToRoadWorthyAdapter(Car car)
  {
    _car = car;
  }

  public void Operate(double speed)
  {
    _car.Drive(speed);
    _car.Honk();
  }
}
```

Do you want a float plane to operate on a road in a particular way?

```csharp
sealed class FloatPlane
{
  public void Crash(string reason) { ... }

  public void Fly(double speed) { ... }

  public void Float() { ... }

  public void RollOnWheels(double speed) { ... }

  public void SkimWater(double speed) { ... }
}

sealed class MustNotFlyFloatPlaneToRoadWorthyAdapter : IRoadWorthy
{
  readonly FloatPlane _floatPlane;

  public MustNotFlyFloatPlaneToRoadWorthyAdapter(FloatPlane floatPlane)
  {
    _floatPlane = floatPlane;
  }

  public void Operate(double speed)
  {
    if (speed <= 50.0)
    {
      _floatPlane.RollOnWheels(speed);
    }
    else
    {
      _floatPlane.Crash("Too fast to stay on the road");
    }
  }
}
```

### "But there are too many classes!"

Did you notice how many more classes there are now than there were before? There
is basically one class for each behavior and also for each kind of usage of the
behavior.

Actually, you should count them. There aren't as many more as you might think
(make sure to remember that `Speedboat`, `Waterway`, and `Watercraft` don't have
equivalents in the above composition example).

But even if there were dozens more classes, would that be a bad thing?

The classes will tend to be smaller. They'll tend to be easier to understand at
a glance. Often, everything you need to know fits on a single screen.

The classes will tend to do less. And just by doing less they will be less prone
to bugs. And smaller [humbler](https://martinfowler.com/bliki/HumbleObject.html)
classes tend to be more _unit testable_.

The classes will tend to change less. If you use composition then you'll find
that you are either creating or deleting classes more than you are modifying
them. And the fewer the modifications, the fewer the regressions.

The classes will tend to be less affected by change. They cannot be affected by
changes upstream in the inheritance hierarchy, because there is no inheritance
hierarchy. As you tailor classes to specific aspects (and then compose those
aspects instead of using inheritance) then you'll find that there are fewer
dependencies between classes. And the fewer the dependencies, the lower the
likelihood of a dependency introducing a breaking change.

And with good design these things will remain true as your codebase grows
to arbitrary complexity and thousands of source code files.

Now, you can definitely go overboard with layer upon layer of interfaces and
abstractions&mdash;I'm guilty of that!&mdash;but that doesn't _necessarily_
happen when you favor composition over inheritance. And neither is that
_necessarily_ prevented by using inheritance.

In fact, I have found more than one misuse of inheritance that encouraged
copy-and-paste instead of encapsulation-and-reuse, and that encouraged
needlessly large and deep inheritance hierarchies (read: needlessly many
classes). And I have personally misused inheritance to create needlessly many
layers of abstraction.

## The point

There is more that could be said, perhaps about how interfaces should be
tailored to the interface's consumer, or the value of unit testing, or how to
make behavior unit-testable, or how following the [SOLID](https://en.wikipedia.org/wiki/SOLID)
principles (especially Single Responsibility!) naturally leads to composition
and unit-testability, or the best ways to create an escape hatch from
inheritance without disturbing too much.

But the point is: **Why are you favoring inheritance over composition? Are you misusing inheritance?**