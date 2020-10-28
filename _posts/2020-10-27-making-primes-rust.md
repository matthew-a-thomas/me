---
title: Making Prime Numbers in Rust
category: programming
description: The scenic route to generating prime numbers with Rust
comments: 2
---
A prime number can only be evenly divided by itself (and by the number one). The
first few prime numbers are:

* 2
* 3
* 5
* 7
* 11

Two is the only even prime number, just like three is the only prime number that
is a multiple of three, five is the only prime number that is a multiple of
five, and so on.

Recall that the modulo operator &ldquo;%&rdquo; tells you the remainder after
division. For example:

$$ 10 \% 3 = 1 $$

Because ten divided by three has a remainder of one.

Eight is not prime because it is an even multiple of two (the remainder after
dividing eight by two is zero):

$$ 8 \% 2 = 0 $$

## Table

With that in mind, here's one way to figure out the first few prime numbers:

<div class="table-responsive">
{% markdown %}
|Number|%2|%3|%4|%5|%6|%7|%8|%9|%10|%11|
|-|
|2|0|2|2|2|2|2|2|2|2|2|
|3|1|0|3|3|3|3|3|3|3|3|
|4|0|1|0|4|4|4|4|4|4|4|
|5|1|2|1|0|5|5|5|5|5|5|
|6|0|0|2|1|0|6|6|6|6|6|
|7|1|1|3|2|1|0|7|7|7|7|
|8|0|2|0|3|2|1|0|8|8|8|
|9|1|0|1|4|3|2|1|0|9|9|
|10|0|1|2|0|4|3|2|1|0|10|
|11|1|2|3|1|5|4|3|2|1|0|
{% endmarkdown %}
</div>

Don't be intimidated by that large table. Think about the row for the number six
for a moment:

<div class="table-responsive">
{% markdown %}
|Number|%2|%3|%4|%5|%6|%7|%8|%9|%10|%11|
|-|
|6|0|0|2|1|0|6|6|6|6|6|
{% endmarkdown %}
</div>

This means six is evenly divided by two, three, and six. Dividing six by any
other number leaves a remainder. So the first thing I want to point out is the
zeros indicate which numbers six is a multiple of.

So to figure out which numbers are primes, just count the zeros in each row. Any
row with only a single zero is a prime number. From the table above:

* 2
* 3
* 5
* 7
* 11

## Sieve

We can also use something called a sieve to find prime numbers. I'll explain how
the
[Sieve of Eratosthenes](https://en.wikipedia.org/wiki/Sieve_of_Eratosthenes){:target="_blank"}
works.

First, make a list of numbers (starting with two):

* 2
* 3
* 4
* 5
* 6
* 7
* 8
* 9
* 10
* 11
* 12
* 13
* 14
* 15
* 16
* 17
* 18
* 19
* 20
* 21
* 22
* 23
* 24
* 25

Now start with two and remove all the multiples of two from the rest of the
list. Make sure to leave the number two. That gives:

* 2
* 3
* 5
* 7
* 9
* 11
* 13
* 15
* 17
* 19
* 21
* 23
* 25

Move on to the next number in the list and do the same thing. So let's remove
all the multiples of three. That leaves this list:

* 2
* 3
* 5
* 7
* 11
* 13
* 17
* 19
* 23
* 25

Keep doing that. When you're done you'll have a list of prime numbers:

* 2
* 3
* 5
* 7
* 11
* 13
* 17
* 19
* 23

## Shortcut

There's a shortcut we can take with both the table and the sieve. I'll explain
it in terms of the table. I hope this will make sense.

Think about the numbers 24, 25, and 31. Here's how their rows in the above table
would look:

<div class="table-responsive">
{% markdown %}
|Number|%2|%3|%4|%5|%6|%7|%8|%9|%10|%11|
|-|
|24|0|0|0|4|0|3|0|6|4|2|
|25|1|1|1|0|1|4|1|7|5|3|
|31|1|1|3|1|1|3|7|4|1|9|
{% endmarkdown %}
</div>

It's clear that 24 and 25 aren't prime numbers because both of them have at
least one zero in addition to the one we'd get if we divided those numbers by
themselves. It wasn't necessary to continue to find remainders of either number
after we found the first zero. And luckily the zeros happened pretty early.

But what about 31? How many columns do we need to determine if 31 is prime?

Recall that every number is a multiple of its square root. If you divide any
number by its square root then you'll be left with its square root exactly.

For example, if we divide 25 by five then we're left with five (its square
root).

Also, if you divide a number by another number larger than its square root
you'll be left with a number smaller than its square root.

For example, if we divide 25 by six (which is larger than five, its square root)
then we're left with four and one sixth (which is less than its square root).

Furthermore if a number is an even multiple of one number then it's also an even
multiple of another number.

For example, 24 is two times twelve, three times eight, or four times six.

All this taken together means we don't have to worry about anything larger than
a number's square root. If a number is not prime then it will be a multiple of
at least one number less than or equal to its square root.

The square root of 31 is ~5.57. That means our table doesn't need columns for
the numbers greater than five. Since we calculated remainders up through five
and there were no zeros then we know 31 is prime.

This also applies to the sieve method. If you start with this list:

* 2
* 3
* 4
* 5
* 6
* 7
* 8
* 9
* 10
* 11
* 12
* 13
* 14
* 15
* 16
* 17
* 18
* 19
* 20
* 21
* 22
* 23
* 24
* 25

...then you can stop after you remove all the multiples of five, because the
square root of the largest number in the list is five.

All the multiples of six will have been crossed out as multiples of two.

All the multiples of seven will have been crossed out as well. Fourteen is a
multiple of two. 21 is a multiple of three.

All the multiples of eight will have been crossed out as multiples of two.

All the multiples of nine will have been crossed out as multiples of three.

All the multiples of ten will have been crossed out as multiples of two.

All the multiples of eleven will have been crossed out as well. 22 is a multiple
of two.

And so on. I hope you get the picture.

## Generator

The disadvantage of both the table and the sieve is you have to know the largest
prime number you want to find. What if you want to continue finding prime
numbers after that?

Well it's not too difficult to extend the sieve idea into a prime number
generator. Let me explain the algorithm.

It centers on a list to which we'll add prime numbers in order as we find them.
That list starts with the number two.

$$ \text{Primes} = (2) $$

We know that the number three is prime because it is not an even multiple of any
of the numbers in our prime numbers list. We don't have to check any of the
prime numbers larger than the square root of three, so we don't even have to
check it against the number two. So we add three to the list.

$$ \text{Primes} = (2, 3) $$

We know that the number four is not prime because it's an even multiple of a
number in our prime numbers list (it's an even multiple of two). So we move on.

Five is prime because it isn't a multiple of two. Again we don't have to check
it against three because three is larger than the square root of five. So we add
five to the list.

$$ \text{Primes} = (2, 3, 5) $$

Six is not prime because it's a multiple of two.

Seven is prime because it's not a multiple of two.

$$ \text{Primes} = (2, 3, 5, 7) $$

Eight is not prime because it's a multiple of two.

Nine is not prime because it's a multiple of three. Again we don't have to
divide it by five or seven because those are both larger than the square root of
nine.

Ten is not prime because it's a multiple of two.

Eleven is prime because it's not a multiple of two or of three.

$$ \text{Primes} = (2, 3, 5, 7, 11) $$

Repeat as long as you want.

## Rust

Let's make the prime number generator in
[Rust](https://www.rust-lang.org/){:target="_blank"}. Here it is:

```rust
struct PrimeGenerator {
  primes: Vec<usize>,
  candidate: usize
}

impl PrimeGenerator {
  fn new() -> Self {
    Self {
      primes: vec![2],
      candidate: 2usize
    }
  }

  fn next(&mut self) -> usize {
    loop {
      let candidate = self.candidate;
      self.candidate = candidate + 1usize;
      for prime in self.primes.iter() {
        let quotient = candidate / prime;
        let remainder = candidate % prime;
        if quotient < *prime {
          // We have run out of primes to search and candidate passed all the
          // tests. It's prime!
          self.primes.push(candidate);
          return candidate;
        }
        if remainder == 0 {
          break; // candidate is a multiple of a prime
        }
      }
    }
  }
}

fn main() {
  println!("Here are the first ten primes:");
  let mut generator = PrimeGenerator::new();
  for _ in 0..10 {
    let prime = generator.next();
    println!("{}", prime);
  }
}
```

Here's the output:

```
Here are the first ten primes:
2
3
5
7
11
13
17
19
23
29
```

This doesn't work in `[no_std]` because the list of primes is stored on the
heap. Heapless collections do exist but I think that's out of the scope of this
tour.