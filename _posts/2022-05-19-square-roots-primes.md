---
title: "Square Roots and Primes"
category: numbers
description: "Why square roots are used in trial division"
---

At the end of
[Making Prime Numbers in Rust]({% link _posts/2020-10-27-making-primes-rust.md %})
I explained how to make a prime number generator. It used trial division. You'll
notice that I don't bother trying to divide the candidate number by any prime
larger than its square root.

Why the square root?

## Graphical answer

<div class="d-flex flex-column">
  <div>100 divided by <span id="divisor-output">10</span></div>
  <input type="range" id="divisor" min="1" value="10">
  <div>...equals <span id="quotient-output">10</span></div>
  <input type="range" id="quotient" min="1" step="0.1" value="10" disabled>
</div>
<script>
  const divisor = document.getElementById('divisor');
  const divisorOutput = document.getElementById('divisor-output');
  const quotient = document.getElementById('quotient');
  const quotientOutput = document.getElementById('quotient-output');
  function updateQuotient() {
    quotient.value = 100 / divisor.value;
    divisorOutput.innerHTML = divisor.value;
    quotientOutput.innerHTML = quotient.value;
  }
  divisor.addEventListener('input', updateQuotient);
</script>
<br>
Play with the top slider for a bit.

Do you get it yet?

Here's a hint: once you've figured out that $$100 \div 2 = 50$$, you then also
know that $$100 \div 50 = 2$$. There's no need to also divide by 50 because you
already know that answer.

The same goes for all these:

|100 divided by|equals|
|-|-|
|2|50|
|3|33.3|
|5|20|
|7|14.3|
|11|9.1|
|13|7.7|
|17|5.9|
|19|5.3|

What do you notice about each of those pairs of numbers?

The square root is always at least as large as one of them.

Once you've divided by primes up through the square root then you already know
the answers for all the primes larger than the square root. So there's no need
to also divide by those larger primes.