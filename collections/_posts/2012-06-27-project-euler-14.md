---
layout: post
title: "Project Euler: Problem 14"
categories: blog
---

Here's problem 14 of Project Euler:

> The following iterative sequence is defined for the set of positive integers:  
>   
> n → n/2 (n is even)  
> n → 3n + 1 (n is odd)  
>   
> Using the rule above and starting with 13, we generate the following sequence:  
> 13 → 40 → 20 → 10 → 5 → 16 → 8 → 4 → 2 → 1  
>   
> It can be seen that this sequence (starting at 13 and finishing at 1) contains 10 terms. Although it has not been proved yet (Collatz Problem), it is thought that all starting numbers finish at 1.  
>   
> Which starting number, under one million, produces the longest chain?  
>   
> NOTE: Once the chain starts the terms are allowed to go above one million.

This problem deals with the [Collatz conjecture](https://en.wikipedia.org/wiki/Collatz_conjecture), more specifically with the length of sequences created by a number. The following is a simple imperative implementation which generates Collatz sequences for a given number and counts the number of steps taken:

```fsharp
let CollatzLength n =
    let mutable x = n
    let mutable s = 1
    while x > 1UL do
        if x % 2UL = 0UL then x <- x / 2UL
        else x <- 3UL * x + 1UL
        s <- s + 1
    (n, s)
```

I initially created a functional declaration of this function, but the imperative version turned out to be much faster. All that remains now is finding the longest sequence:

```fsharp
let result = [ for i in 1UL..999999UL do yield CollatzLength i ] |>
    List.maxBy (fun (a, b) -> b)
```

CollatzLength returns tuples, and we need to find the maximum only by sequence length. The longest sequence turns out to be 525 steps long.
