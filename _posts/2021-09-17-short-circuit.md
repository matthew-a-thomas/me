---
title: Short Circuit Protection
description: I'm happy to report that my power supply has it
category: computers
---

My PC power supply recently bit the dust. It was during a storm when the power
cut out. My PC wouldn't turn back on afterward. Maybe a power surge? At any
rate, I ordered a new power supply and it finally arrived yesterday evening.

And I'm happy to report that my new power supply has short circuit protection.
Here's how I found out.

---

I took the cover off my PC case and freed the old power supply from its screws.
Both the old and the new power supply are modular: instead of all the cables
dangling out of it you only plug in the ones you need. So I unplugged all the
cables from the power supply end. "This will be an easy job" I thought to
myself, "all I have to do is drop the new power supply in and replug all these
modular cables."

Wrong.

First I noticed the main motherboard power connector (the big 24 pin one) was
different. Of course it's the same on the motherboard end, but on the power
supply the 24 pins are split up into two plugs: something random like 10 pins on
one plug and the remaining 12 pins on another one; two different modular slots
separated by a little distance on the power supply. Did motherboards used to
only take 10 or 12 pin connectors? It was clear that my old plug was not going
to work.

"No problem, I'll just use the cable that came with the new power supply."

Next was the auxiliary 8 pin motherboard power plug. Both ends fit.

"Good."

The graphics card power plug was different on the power supply side. It had the
same number of pins in the same pattern, but they were keyed slightly different.
I don't really like the new cord. It is flat instead of braided so it doesn't
sit right in the case. And it has two graphics card plugs on the one end even
though I only have one graphics card. Who wants an extra plug dangling in their
case?

_Mumble grumble._ Out with the old cord, in with the new.

Finally, the two or three strings of hard drive power plugs. Again, exactly the
same on the hard drive side of things, and exactly the same number of pins in
the same pattern on the power supply side. But keyed differently.

"Ugh, why are they doing this to me?"

The other side of the case came off to expose the important ends of my hard
drives. I carefully dissected the kraken's tangled mess of tentacles. After some
time and effort I had the old power strings in my hand. Next I cautiously
inserted the new strings' plugs.

"I hope I didn't accidentally bump any of these SATA connectors."

And I was done!

On went the case panels, up went my PC onto its little shelf, in went the power
cord and USB plugs and display cords, down went my finger onto the power button.

Nothing.

"I _think_ I pressed the power button?"

I poked it again. Still nothing.

"Hmm... Maybe the power surge destroyed my motherboard too? Why do I buy such
cheap surge protectors?"

I knew I still had the bent paperclip laying around that I had used to jumper
the old power supply as I diagnosed it. I grabbed that, took the side panel off,
unplugged the motherboard end of the fat 24-pin motherboard power cable, and
carefully inserted the paperclip into the correct two pins.

There was a very faint click from a solenoid in the power supply, then nothing.

I did it again just to see. Same thing. Again. And, there were a couple of faint
clicks, like it was groggily waking up but then something was urgently shouting
at it "Quick!! Save yourself!!" in the midst of other terrified screams and in a
panic it would stumble over itself as it rushed to disconnect itself from the
mains.

Perplexed, I unplugged everything from the power supply.

"Good thing this power supply is modular!"

And I tried again. Just a single click this time.

"That sounds like a normal turning on click to me. Not a panicky click."

Time for process of elimination. In went the graphics card plug. Fine. The hard
drive plugs. Fine. The big fat 24 pin motherboard plug and the smaller 8 pin aux
power plug.

Not fine.

Actually, there was a very brief flash of light coming from somewhere in the
case.

Now, a flash of light by itself is nothing unusual. They stick LEDs on
everything. I hate them, but the WiFi card has one, some aim out the back of my
motherboard, they're everywhere. I feel like I had to pay extra to buy RAM
without LEDs. Why would someone want LEDs on their RAM??

"Okay, it doesn't like my motherboard. But where is that light coming from?"

I kept turning it on. Sometimes there would be two clicks from the power supply.
Sometimes there would be several. The poor power supply was going to have PTSD.
But I had to find that brief flash of light.

And I discovered it wasn't coming from the open side of the case. It wasn't
coming from anything in or on the _top_ of the motherboard, but instead I could
see the flash coming from _behind_ the lower corner.

"Oh great, did I drop a screw in there or something?"

I imagined all the things that could have fallen into the holes in the back of
the case.

Out came all the SATA plugs. Out came the graphics card and WiFi card. Out came
the motherboard power plugs. Out came all the screws holding the motherboard in
place. Out came the motherboard.

"Hmm. No loose screws laying around behind the motherboard. Just dust."

I flipped it around and searched for the scorch marks. Nothing.

Well there had to be something making that light. I looked closer.

And then I saw it: two strings of surface mount LEDs.

On the _back_ of my motherboard.

_Groan!_

"I am never buying a single LED ever for the rest of my life. I'm not even going
to talk to anyone who has anything to do with LEDs or anything that LEDs go on,
in, or around. I don't even like the letter 'L' anymore."

What in the world were they doing on the _back_ of my motherboard?? Diagnostic
lights? Mood lighting? Well all this diagnosing wasn't putting me in a light
mood.

Okay, it definitely wasn't arcing. I should have known: the hue wasn't quite
right, and it was silent. Well at least the light has a legitimate claim to
existence; _someone_ wanted light to come out the back of the motherboard, and
here light was coming out the back. I couldn't blame the light for doing what it
was designed to do.

In went the motherboard to the case. In went the motherboard screws.

"Hmm. Maybe I'll do process of elimination again, just to be extra sure."

In went the 24 pin motherboard plug (and the case power switch plug). I poked
the power button.

And everything sprang to life! I had left the hard drive power cords plugged
into the power supply; they were happily whirring away. My CPU fan was spinning
its little heart out.

Wow! What a wonderful chorus.

"Maybe my graphics card?"

In went the graphics card power. Fine! I could just hear all the money I was
saving by not replacing all these components. What a beautiful noise.

"A short in some case accessory, maybe one of the USB plugs?"

In went all the case accessories and fans. Fine!

I was getting tired of saving money. There was a problem to solve. What in the
world went wrong??

"Wait a minute, what's this one cord still dangling here?"

I had forgotten about the 8-pin motherboard auxiliary power plug. I guess it
really is auxiliary after all? But it was dangling and begging for a home, so I
plugged it back in.

_Click._

Aha! The culprit, caught red-handed!

I looked closer at that cord and compared it to the new one. Of course the
motherboard side was identical on both. And the power supply side was physically
keyed the same. But there was an empty pin in one place on the one cord, and an
empty pin in a different place on the other cord.

They were wired differently! I was sending 12V straight to ground! A short
circuit!

# 🤦‍♂️

## The point

Don't mix and match modular power supply cables from different manufacturers.