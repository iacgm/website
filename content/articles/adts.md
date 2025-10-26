---
date: '2025-10-24'
draft: false
title: 'Lists are Geometric Series'
summary: 'Taylor Expansions of Recursive Data Types'
---

## Introduction 

Functional programmers get a lot of flack for being too theoretically-minded. ["Real"](https://xkcd.com/378/) programmers think of them as being passionate, perhaps even noble idealists, but ultimately best left to their corner of the woods, [away from those actually getting stuff done](https://www.youtube.com/watch?v=iSmkqocn0oQ). The functional programmers, on the other hand, high up in their ivory tower, don't think about the rest of us at all. Instead, they spend their days writing about category theory, [arranging languages into blocks](https://en.wikipedia.org/wiki/Lambda_cube), and, on occasion, discovering [pearls](https://wiki.haskell.org/Research_papers/Functional_pearls).

This article will explain one simple, lovely fact from the PL world. It's unlikely to inform how you actually program, but if you don't have one already, it might give you a taste for what the whole PL/FP fuss is all about, anyway.

## Algebraic Data Types

Before we get to the pearl, we have to briefly discuss Algebraic Data Types (ADTs). These are just datatypes which we can combine to form new types. Specifically, we do this with **_sum_** and **_product_** types: 

```haskell
type Product a b = (a, b)     -- an `a` and a `b`
type Sum     a b = A a | B b  -- an `a` or a `b`
```

These names are especially intuitive if we think of these as sets: If \(A\) has \(n\) elements, and \(B\) has \(m\) elements, then \(A \times B\) has \(n * m\) elements and \(A+B\) has \(n+m\) elements. Extending this logic, we get an **_exponential_** type, as well as unit and a void type, with one and zero elements respectively:

```haskell
type Pow a b = b -> a  -- an `a` for every `b`
type Unit = Unit
type Void
```

This analogy extends to our familiar [high-school algebra identities](https://www.cs.ox.ac.uk/people/daniel.james/iso/iso.pdf) (up to isomorphism):

\[
\begin{align}
  A + 0 &\cong A \\
  B + A &\cong B + A \\
  (A + B) + C &\cong A + (B + C) \\
  \\
  A \times 0 &\cong 0 \\
  A \times 1 &\cong A \\
  A \times B &\cong B \times A \\
  (A \times B) \times C &\cong A \times (B \times C) \\
  \\
  1^A &\cong 1 \\
  A^0 &\cong 1 \\
  A^1 &\cong A \\
  A^{B + C} &\cong A^B \times A^C \\
  (A^B)^C &\cong A^{B \times C} \\
  \\
  (A + B) \times C &\cong (A \times C) + (B \times C) \\
  (A \times B)^C &\cong A^C \times B^C
\end{align}
\]

## The Pearl 

Armed with this notation, we can turn our attention to (linked) lists:

```haskell
data List a = Nil | Cons a (List a)
```

Rewriting this in our new notation, we can expand this out into an infinite, non-recursive definition:

\[
\begin{align}
  L &= 1 + a \times L \\ 
    &= 1 + a (1 + a \times L) \\
    &= 1 + a + a^2 \times L \\
    &= \cdots \\
    &= 1 + a + a^2 + a^3 + a^4 + \cdots
\end{align}
\]

What's the intuition here? What this tells us is that a `List a` is either: zero `a`s, one `a`, two `a`s, etc.

That's exactly what a list is! But this correspondance doesn't end here. For example, consider these two types:

```haskell
data Tree a = Nil    | Tree a (Tree a) (Tree a)
data Fork a = Leaf a | Fork (Fork a) (Fork a)
```

These become:

\[
\begin{align}
  T &= 1 + a \times T^2 \\ 
    &= 1 + a (1 + a \times T^2)^2 \\
    &= 1 + a + 2 a^2 \times T^2 + a T^4 \\
    &= \cdots \\
    &= 1 + a + 2 a^2 + 5 a^3 + 14 a^4 + \cdots \\
  \\
  F &= a + F^2 \\
    &= a + (a^2 + 2 a F^2 + F^4) \\
    &= \cdots \\
    &= a + a^2 + 2 a^3 + 5 a^4 + 14 a^4 + \cdots
\end{align}
\]

These are the [Catalan Numbers](https://oeis.org/A000108), and indeed, there is one empty `Tree`, one `Tree` with one element, 2 `Tree`s with 2 elements, and so on. The pattern's holding! [^admit]

[^admit]: Admittedly, it's not clear that these expansions are correct, but they follow from the recurrence \(C_{n+1} = \sum_{i=0}^n C_i C_{n-i}\).

Moreover, we can see that \(F \cong A \times T\)! These two data structures, one with all its data on its leaves, and all its leaves bare, are actually (almost) the same! 

But what about going the other way? Starting from a series and _deriving_ a data structure? We'll use a simple example, where the coefficients just count up:

\[
\begin{align}
  1 + 2 a + 3 a^2 + \cdots = 1 + a + &a^2 + a^3 + \cdots
  \\ + a + &a^2 + a^3 + \cdots
  \\ + &a^2 + a^3 +  \cdots
  \\ & \quad \vdots
\end{align}
\]

This is just \(L^2\)! And, as we now expect, there are 3 ways a pair of `List a`s can have 2 elements, 4 ways it can have 3 elements, and so on. [^lists]

[^lists]: Namely: ([a,a], []), ([a], [a]), ([], [a,a]); ([a,a,a], []), ([a,a], [a]), ([a], [a,a]), ([], [a,a,a]); etc.

By now, you can probably guess, for example, what might happen if we introduced several variables:

```haskell
Type ABList a b = Nil | A a (ABList a b) | B b (ABList a b)
```

If you'd like more of a challenge, why not try coming up with a data structure corresponding to the Fibbonacci sequence:

\[
  F = 1 + a + 2a^2 + 3a^3 + 5a^4 + 8 a^5 + \cdots
\]

## Further Reading

There's more to be said about these expansions, but I find the fact that they exist cool enough as it is. If you're interested in this, maybe check out:

- [This StackOverflow question](https://stackoverflow.com/questions/9190352/abusing-the-algebra-of-algebraic-data-types-why-does-this-work), from someone discovering these patterns themselves.
- This [blog post](https://codewords.recurse.com/issues/three/algebra-and-calculus-of-algebraic-data-types), from the 
Recurse Center.
- This [post](https://pavpanchekha.com/blog/zippers/derivative.html#sec-2), about extending this correspondance to derivatives.
