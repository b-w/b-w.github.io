---
layout: post
title: "Project Euler: Problem 5"
categories: blog
---

Euler problem 5 asks the following:

> 2520 is the smallest number that can be divided by each of the numbers from 1 to 10 without any remainder.  
>   
> What is the smallest positive number that is evenly divisible by all of the numbers from 1 to 20?

Solving this problem actually involves no F# but a math trick instead:

```fsharp
let result = 2 * 2 * 2 * 2 * 3 * 3 * 5 * 7 * 11 * 13 * 17 * 19
```

Why? Well, we're really just finding the least common multiple of the numbers in the range 2 .. 20\. This is the same as finding the smallest set of prime numbers which (by way of multiplication) allows us to create all the numbers 2 .. 20\. Here we go:

*   The first number is 2\. We add 2 to the set: { 2 }
*   The second number is 3\. We add 3 to the set: { 2, 3 }
*   The third number is 4\. 4 is not a prime number. Instead, we add another 2 to the set: { 2, 2, 3 }
*   The fourth numbers is 5\. We add 5 to the set: { 2, 2, 3, 5 }
*   The fifth number is 6\. We add nothing, as 6 can already be made by multiplying 2 and 3.
*   The sixth number is 7\. We add 7 to the set: { 2, 2, 3, 5, 7 }
*   The seventh number is 8\. 8 is not a prime number. Instead, we add another 2 to the set: { 2, 2, 2, 3, 5, 7 }
*   The eight number is 9\. 9 is not a prime number. Instead, we add another 3 to the set: { 2, 2, 2, 3, 3, 5, 7 }
*   ...
*   The nineteenth number is 20\. We add nothing, as 20 can already be made by multiplying 2, 2, and 5.

Eventually, we end up with the set { 2, 2, 2, 2, 3, 3, 5, 7, 11, 13, 17, 19 }. When then multiple those to arrive at the least common multiple.
