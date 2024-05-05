---
layout: post
title: "Project Euler: Problem 3"
categories: blog
---

Taking on the next Project Euler problem:

> The prime factors of 13195 are 5, 7, 13 and 29.  
>   
> What is the largest prime factor of the number 600851475143 ?

This problem is about [integer factorization](http://en.wikipedia.org/wiki/Integer_factorization), which decomposes a number into a series of prime numbers that together multiply to get the original number back. Although apparently no efficient algorithm is known, smaller numbers can be factored quite quickly.

The basic approach of my solution is to start at the smallest relevant prime number (2), divide the input number by it as often as possible, increment the number by 1, and repeat.

```fsharp
let factorize(n : uint64) =
    let rec fac(x : uint64, p : uint64, f : uint64 list) =
        if x = 1UL then f
        elif x % p = 0UL then fac(x / p, p, p :: f)
        else fac(x, p + 1UL, f)
    fac(n, 2UL, [])
```

Note that I'm not actually explicitly doing anything with prime numbers. Because I start dividing by 2 and work my way up from there, I won't have to worry about my divisors being prime numbers or not: the number will never be divided by 4, because it has already been divided by 2.

To find the answer, I simply take the latest addition to the list of factors:

```fsharp
let result = List.head(factorize(600851475143UL))
```

This leads to the correct answer of 6857.
