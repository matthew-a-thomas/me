---
title: Air Spring Preload
description: Adding preload makes an air spring stiffer, just like any spring
category: misc
---

You can make an [air spring](https://en.wikipedia.org/wiki/Air_suspension) from an air piston like this:

```
         load
          V
          |
          |
|         |         |
|         |         |
| shaft > | < shaft | <
|         |         | <
|---------+---------| < barrel
|                   | <
|    ^ piston ^     | <
|                   |
|      * air *      |
|                   |
+-------------------+

   ^ cylinder cap ^
```

As long as the air inside the piston has nowhere to go, it'll be springy.

What follows is a quick proof that adding preload makes the air spring stiffer.

## Proof

According to gas laws, an isothermic (meaning thermal equilibrium) air spring obeys this principle:

$$ P_1 \cdot V_1 = P_2 \cdot V_2 $$

In other words, the pressure multiplied by the volume remains the same.

Now, a force is pressure multiplied over an area:

$$ F = P \cdot A $$

$$ P = \frac{F}{A} $$

Which means,

$$ \frac{F_1}{A_1} \cdot V_1 = \frac{F_2}{A_2} \cdot V_2 $$

$$ F_1 \cdot h_1 = F_2 \cdot h_2 $$

$$ \frac{F_1}{F_2} = \frac{h_2}{h_1} $$

In other words, the ratio between the forces equals the inverse ratio between the heights.

Now, suppose that we preload it. Meaning add something to _both_ forces, not just one of them. Using the picture above, if you increase the load on top then that's increasing one force, and that will be greeting by an equally opposing force (as the air gets squashed).

$$ F = \alpha + G $$

$$ \frac{\alpha + G_1}{\alpha + G_2} = \frac{h_2}{h_1} $$

"Stiffness" can be understood in terms of the change in that ratio $$ \frac{h_2}{h_1} $$. The smaller the change, the stiffer it is.

So let's pretend that the forces and heights are constant. What effect does the preload have on the rate of change of that ratio?

Take the partial derivative of the left side:

$$ \frac{\partial}{\partial \alpha} \frac{\alpha + G_1}{\alpha + G_2} = \frac{G_2 - G_1}{(G_2 + \alpha)^2} $$

See the $$ \alpha $$ (squared) in the denominator? That means the rate of change of that ratio goes to zero as $$ \alpha $$ (the preload) increases.

In other words, an air spring gets stiffer when you add preload.

Just like any spring.

## Applications

There are multiple applications of this.

### Compressed gas springs

There are multiple ways to add preload. One way is to put a weight on top like my example above. But another way is to enclose the top of the air piston and fill that space with _compressed_ air. Then you get a compressed gas spring.

### Make an air piston less slingshot-y

Suppose you replaced all the hydraulic cylinders in your excavator with air pistons. You have to work _really_ hard to lift a heavy rock by pumping a ton of very compressed air into the air pistons. Then as the rock begins to lift it slips and falls out of the bucket. Guess what happens to all that compressed air? It decompresses by violently slingshotting the bucket off of your excavator and sending it into orbit. I hope you have insurance.

That's a silly example. But there are real life usages of air cylinders that move loads, and sometimes loads shift.

How can you make it less like a slingshot? Add preload.
