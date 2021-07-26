---
title: WPF's Problem with Axes
description: Chart axes, that is
category: programming
---

WPF performs layout in two passes. The first is called the "measure" pass. The
second is called the "arrange" pass.

During the measure pass a control is queried for the minimum amount of space it
needs. Between that and the arrange pass the parent control decides how much
space to actually give it. Then the arrange pass happens: the control is told
the maximum amount of space it can use and it replies with how much space it
actually used.

That's a very powerful way to do layout. The system is flexible enough to make
all these different layouts possible:

 * Grid/table. The sizes of child controls are related through rows
   and columns
 * Stack. In a vertical stack the widths of child controls are related, and they
   get as much vertical space as they want
 * Wrapped list. Child controls are lined up. When one passes the end of the
   line then it and all subsequent children drop down to the next line
 * Canvas. No size constraints or relations whatsoever between child controls.
   They are manually placed and can take up as much space as they want
 * And more!

Sounds great! But can you tell what they all have in common?

Their number of children can be known ahead of time.

## The challenge

Everyone is familiar with line graphs. Here's one I threw together in Excel:

![Line graph](/assets/img/wpf-problem-with-axes/line-graph.png){:class="img-fluid"}

Let's focus on the vertical axis (the numbers on the left). How would you make
that in WPF?

## Vertical stack

It looks a lot like a vertical stack of text blocks, doesn't it? Easy enough:

```xml
<StackPanel Orientation="Vertical">
  <TextBlock Text="25">
  <TextBlock Text="20">
  <TextBlock Text="15">
  <TextBlock Text="10">
  <TextBlock Text="5">
  <TextBlock Text="0">
</StackPanel>
```

Add some styling and you're done, right?

Wrong. How did you know to use increments of five? Excel used increments of five
because that's just how large my chart happened to be. But if I squash it some:

![Squashed line graph](/assets/img/wpf-problem-with-axes/squashed.png){:class="img-fluid"}

...then it uses a different increment.

So how many text blocks should there be? You won't know until you try to fit the
axis into a certain area.

A chart axis is different than all of the layouts I listed above because a chart
axis does not know how many child tick marks it has until you put it somewhere.

## Custom control

Okay, so what is the soonest we can know how much space our axis has to work
with, so that we can decide how many child tick marks it should have?

Well... we have access to that information as soon as the arrange pass. And we
can hook into that. How about this?

```csharp
class Axis : FrameworkElement
{
  public static readonly DependencyProperty TickTemplateProperty = ...;

  readonly List<FrameworkElement> _children = new();

  protected override IEnumerator LogicalChildren => _children.GetEnumerator();

  public DataTemplate TickTemplate
  {
    get => (DataTemplate)GetValue(TickTemplateProperty);
    set => SetValue(TickTemplateProperty, value);
  }

  protected override int VisualChildrenCount => _children.Count;

  protected override Size ArrangeOverride(Size finalSize)
  {
    // Clear the current children
    foreach (var child in _children)
    {
      RemoveVisualChild(child);
      RemoveLogicalChild(child);
    }
    _children.Clear();

    // Figure out how many we need and add them
    var sizeOfOneTick = ???
    var numTicks = finalSize.Height / sizeOfOneTick;
    for (var i = 0; i < numTicks; ++i)
    {
      var child = (FrameworkElement)TickTemplate.LoadContent();
      _children.Add(child);
      AddLogicalChild(child); // You want bindings to work in styles, right?
      AddVisualChild(child); // You want it to show up, right?
    }
    ApplyTickValues();
  }

  void ApplyTickValues()
  {
    var tickInterval = 30.0 / _children.Count;
    for (var i = 0; i < _children.Count; ++i)
    {
      var child = _children[i];
      var value = tickInterval * i;
      child.DataContext = value;
    }
  }

  protected override Visual GetVisualChild(int index) => _children[index];

  protected override Size MeasureOverride(Size availableSize)
  {
    var infiniteSize = new Size(double.PositiveInfinity, double.PositiveInfinity);
    foreach (var child in _children)
      child.Measure(infiniteSize);
    return new Size(0, 0);
  }
}
```

There is a lot of cruft since we're manually managing both logical and visual
children. And we make some assumptions about the axis range and how tick
intervals are calculated. But those aren't serious challenges.

We managed to hook into the measure and arrange passes, and in the arrange pass
we are making the correct number of tick marks. And we even abstracted away the
job of creating a tick mark so that our axis control is more reusable.

Mission accomplished? Nope! Do you see the issue?

How do we know how large one tick mark is on the screen? If a tick is just a
simple text block with unchanging text then it's very easy. WPF has
[just the thing](https://docs.microsoft.com/en-us/dotnet/api/system.windows.media.formattedtext?view=net-5.0)
for measuring text. But that doesn't work for the data template above. What if
someone wants to make the tick marks a little more complex than plain old text?
We could manually perform measure and arrange passes on the first one within our
own arrange pass. But a lot of things only work within a logical tree. So we
would have to ensure that the first tick mark (the one we're measuring) is
temporarily added to our logical tree during our arrange pass. But guess what
happens when our logical tree changes? WPF performs another layout pass on us!
So in figuring out how large one tick mark is within the arrange pass we end up
triggering another arrange pass, which will have to calculate how large a tick
mark is which will trigger another arrange pass, on and on forever.

Note that bindings in the `TickTemplate` might also cause the tick marks to
change size. I'm glossing over this detail because ideally the next layout pass
would account for this change.

## The DCC law

We cannot alter the number of children in either our measure or our arrange pass
because doing so triggers another layout pass and so on forever. I'm going to
call this the WPF Dynamic Child Count (DCC) law:

> At least one layout pass must be able to complete without altering the logical
  or visual trees

If we violate this law then we run into infinite loops and other trouble.

This means that the happy path through both our measure and arrange passes must
operate on children that have already been created and added. In other words,
they should look as close as possible to this:

```csharp
protected override Size ArrangeOverride(Size finalSize)
{
  foreach (var child in _children)
  {
    var childRect = new Rect(
      ..., // Get location from somewhere
      child.DesiredSize
    );
    child.Arrange(childRect);
  }
  return new Size(...); // Decide how much space we actually used
}

protected override Size MeasureOverride(Size availableSize)
{
  foreach (var child in _children)
  {
    var childSize = ...; // Compute child size somehow
    child.Measure(childSize);
  }
  return new Size(...); // Decide what the minimum viable size is
}
```

There's a lot of hand waving there, but:

 * Neither the measure pass nor the arrange pass alters the logical tree
 * Neither pass creates children, but instead operates on children that are
   already there

So we aren't breaking the DCC law.

But now we have a chicken-and-egg problem. Where do the child tick marks come
from?

## The compromise

The solution is a compromise: create the children in response to size changes:

```csharp
// Note: this isn't the only way to do this
protected override void OnRenderSizeChanged(SizeChangedInfo sizeInfo)
{
  _children = ...; // Create tick mark children based on the current RenderSize
  base.OnRenderSizeChanged(sizeInfo);
}
```

Why is this a compromise? Because now it's impossible for our axis control to
be `Auto`-sized. If we drop it into an `Auto`-sized cell in a `Grid` then it
won't show up because it'll get scrunched out of view.

Why is that? To answer that we have to look a little more closely at our axis in
this sequence of events:

 1. Parent control performs a measure pass on us, telling us we have so much
    space available and asking how much of it we need
 2. We don't have any children yet because our size hasn't yet changed. What
    amount of space will we say that we need?
 3. Parent control decides on some amount of space to give us
 4. Parent control performs an arrange pass on us, telling us we have so much
    space available
 5. We don't have any children yet because our size hasn't yet changed
 6. After our arrange pass completes, WPF changes our size to the amount of
    space we said we used
 7. Since we have hooked into our own size changes, we decide that so many
    children will fit, instantiate that number of children, and add them to our
    logical and visual trees
 8. If we added any children then WPF will cause our parent to perform another
    layout pass over us, and the above steps might repeat

In step #2, what amount of space will we say we need when we have no children
yet? Note: we cannot say we need as much space as we're given (in other words we
cannot just return the `availableSize` parameter) because it's common for
controls to be offered _infinite_ space and WPF throws an exception when you
return that. Some finite number has to be returned. But what? Let's settle on
returning `new Size(0, 0)`, or no space. That tells our parent that we're
really flexible and can fit anywhere. (As an aside: this is what `Canvas` does.)

Okay, so our parent knows that we don't need any space. Guess what happens if
our parent is a `Grid` and we're in an `Auto`-sized cell? The `Grid` gives us
no space! Step #4 happens and we're told we have no space. Our only possible
reply in step #5 is that we used no space. So our `RenderSize` gets set to zero,
obviously no child tick marks will fit in zero space so we don't create any
children, so we don't add any children, so our logical and visual trees don't
change, so we're done. Our axis won't show up!

It has to be told how large it is. It cannot be `Auto`-sized.

Pretty lame, right?

_Caution: the code above will likely not compile. It's just inspirational
pseudocode written off the cuff._