---
title: Custom Music Notation
description: A toy file format for simple songs
category: programming
---

I made a toy file format for simple songs. Here's what [_Amazing Grace_](https://hymnary.org/media/fetch/96169) looks like:

```
 -               A 3 1

 -  1 4      1 4  2 1 1         1 4
       2        2                  2
G- 4 1  4   4 1        4  11   4 1  A
 #
 -       2               3  1
  2       42            2    42
R
```

You read it from left to right. The first column tells where G4 is (G4 is the second line from the bottom in the treble clef), and also tells where the "rest" line is. The second column is for the key signature: "b" means the notes on that line are flat; "#" means they are sharp. _Amazing Grace_ is in the key of G major so all the lines for "F"s are sharp. Then a single number is in each of the following columns.

You use the number "1" for the shortest note in the song. _Amazing Grace_ has eighth notes (but not sixteenth notes) so for it "1" means "eighth note", "2" means "quarter note", and so on. The number has to fit in a single column but 1&ndash;9 is a pretty limited range, so actually it's a base 26 number (A = 10, B = 11, and so on).

Maybe later I'll post my code for parsing this format into C# data objects, and for determining the frequency and duration of each note. I'm currently using that code to generate stepper motor commands to make a stepper motor sing songs for me.

I find this format easy to author (once you get the hang of it) and not too bad to parse... perfect for toy projects.