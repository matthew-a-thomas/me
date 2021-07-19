---
title: Fountain Codes
description: What they are and how they work
category: programming
---
A [fountain code](https://en.wikipedia.org/wiki/Fountain_code) is a
[forward error correction code](https://en.wikipedia.org/wiki/Error_correction_code#Forward_error_correction)
for lossy channels. In other words a fountain code is an
[erasure code](https://en.wikipedia.org/wiki/Erasure_code).

That's just a fancy way to say that a fountain code is one of those things that
lets you send a message somewhere, have parts of your communication fail to make
it, and the other party will still be able to understand you clearly.

Fountain codes aren't the only thing that can do this. But they are interesting
for some reasons I'll get into.

## The short explanation

Send enough redundant information to the other party that on a good day you
wasted some time talking and on a bad day they can still piece together exactly
what you said.

## The long explanation

What you will do:

1. Split your binary message up into $$k$$ parts
1. Pick a random subset of those $$k$$ parts and XOR them together
1. Call that jumbled mess (with an indication of which pieces were XOR'd
   together) a "row"
1. Send that row
1. Repeat at least $$k$$ times
1. Repeat some more for good measure
  * On a good day you'll only have to repeat a small number more times
  * On a bad day you'll have to repeat several more times depending on how many 
    of your transmissions were lost

What the other party will do:

1. Gather as many rows as possible
1. Once they have at least $$k$$ rows, try to solve a system of equations

And that's it!

## An example

Suppose you want to send this message:

> Hi!

Here are its ASCII bytes in hexadecimal:

> 0x48 0x69 0x21

And here are its ASCII bytes in binary:

> `01001000 01101001 00100001`

### Sending the message

There are three bytes, so let's choose $$k = 3$$ and split the message up
accordingly:

|Part|Binary|
|-|-|
|1|`01001000`|
|2|`01101001`|
|3|`00100001`|

The next step is to choose a random subset of these three parts and XOR them
together. Let's choose parts one and two. Keep in mind that $$\oplus$$ means XOR
and $$01001000 \oplus 01101001 = 00100001$$:

|Parts|$$\oplus$$|
|-|-|
|1, 2|`00100001`|

And we do that again. Let's choose parts two and three:

|Parts|$$\oplus$$|
|-|-|
|1, 2|`00100001`|
|2, 3|`01001000`|

And let's do that a third time, picking parts one, two, and three:

|Parts|$$\oplus$$|
|-|-|
|1, 2|`00100001`|
|2, 3|`01001000`|
|1, 2, 3|`00000000`|

### Receiving the message

Now let's pretend like we have received those three rows. We can represent them
as a system of equations:

|Row|Part 1|Part 2|Part 3|$$\oplus$$|
|-|-|-|-|-|
|1|✓|✓||`00100001`|
|2||✓|✓|`01001000`|
|3|✓|✓|✓|`00000000`|

Now just solve the system of equations. The row operation we'll use is XOR. Here
are the steps to solve it:

<ol>
<li>XOR row 1 into row 3:
{% markdown %}
|Row|Part 1|Part 2|Part 3|$$\oplus$$|
|-|-|-|-|-|
|1|✓|✓||`00100001`|
|2||✓|✓|`01001000`|
|3|||✓|`00100001`|
{% endmarkdown %}
</li>
<li>XOR row 3 into row 2:
{% markdown %}
|Row|Part 1|Part 2|Part 3|$$\oplus$$|
|-|-|-|-|-|
|1|✓|✓||`00100001`|
|2||✓||`01101001`|
|3|||✓|`00100001`|
{% endmarkdown %}
</li>
<li>XOR row 2 into row 1:
{% markdown %}
|Row|Part 1|Part 2|Part 3|$$\oplus$$|
|-|-|-|-|-|
|1|✓|||`01001000`|
|2||✓||`01101001`|
|3|||✓|`00100001`|
{% endmarkdown %}
</li>
</ol>

And it's solved! Did you notice what is now in that last column? Look a little
more closely:

|Binary|Hex|ASCII|
|-|-|-|
|`01001000`|0x48|H|
|`01101001`|0x69|i|
|`00100001`|0x21|!|

I'll leave it as an exercise for you to figure out
[what would have happened](https://en.wikipedia.org/wiki/Overdetermined_system)
if we had gathered four rows instead of three. Or if we had gathered a different
three rows. Or if we had chosen something for $$k$$ other than three. Or what
the probability is of being unable to decode the message when you have gathered
exactly $$n \geq k$$ rows.

## Some interesting properties

Fountain codes are _rateless_. That means that you don't have to decide up front
how many rows you're going to send. You can just keep sending them until you get
tired of it. This is different than
[Reed Solomon](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction).

Fountain codes can easily be multicasted. Meaning several parties can send a
message over individually slow connections to a single party and the receiving
party will get it at the combined speed of all those connections. There are
$$2^k - 1$$ unique and useful rows for every message of $$k$$ parts. Because
that number gets very large very quickly there is no need to coordinate between
all the senders: each sender can just randomly generate rows and they're more
than likely going to be unique and useful to the receiving party.

## Some downsides

It's pretty I/O intensive to generate a row. On average you have to read half of
the message each time.

It's pretty I/O intensive to solve the system of equations. If you use
[Gaussian Elimination](https://en.wikipedia.org/wiki/Gaussian_elimination) then
you're looking at $$O(n^2)$$ row operations. And rows can be pretty large
depending on what $$k$$ is and how large your message is.

Fountain codes only help in _erasure_ channels. But many communication channels
in real life are _noisy_. In practice that usually means you might have to
transmit each row wrapped in an error detection code (like CRC) which adds
overhead.

Throughput is strongly dependent on the size of each row. Rows that are too
large will usually have a greater chance of being lost through an erasure
channel. But row size is inversely proportional to $$k$$, so too small a row
size means it will take a very long time for the receiving party to solve the
system of equations.

You have to send information about which parts were XOR'd together which adds
overhead. This overhead can be very significant for small messages or small row
sizes.

## See also

[https://github.com/matthew-a-thomas/Fountain](https://github.com/matthew-a-thomas/Fountain)&mdash;fountain
codes applied to files.

[https://github.com/matthew-a-thomas/cs-fountain-codes](https://github.com/matthew-a-thomas/cs-fountain-codes)&mdash;simulated
performance of a few different fountain codes (or fountain-code-esque things).
The graphs on that page can be pretty confusing. Open an issue in that
repository if anything isn't clear.