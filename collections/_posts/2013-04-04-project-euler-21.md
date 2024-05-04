---
layout: post
title: "Project Euler: Problem 21"
categories: blog
---

The last two Euler problems were rather anticlimactic, so let's just keep going.

> Let d(n) be defined as the sum of proper divisors of n (numbers less than n which divide evenly into n).  
> If d(a) = b and d(b) = a, where a ≠ b, then a and b are an amicable pair and each of a and b are called amicable numbers.  
>   
> For example, the proper divisors of 220 are 1, 2, 4, 5, 10, 11, 20, 22, 44, 55 and 110; therefore d(220) = 284\. The proper divisors of 284 are 1, 2, 4, 71 and 142; so d(284) = 220.  
>   
> Evaluate the sum of all the amicable numbers under 10000.

We'll solve this problem in F#. The goal is to identify amicable numbers, but we start off with a few simple helper functions we'll need:

```fsharp
let Divisors n = [1..n / 2] |> List.filter (fun x -> n % x = 0)

let SumDiv n = Divisors n |> List.sum
```

These two provide us with a simple implementation of _d(n)_. The next step is using this to identify the amicable numbers. The definition provided in the problem description depends on amicable pairs. From it, we'll derive the definition of an amicable number. First, we have _d(a) = b_ and _d(b) = a_. Rewriting, we get _d(d(a)) = a_. We also have _a ≠ b_. Using the previous definition of _d(a) = b_, we can rewrite this to _a ≠ d(a)_.

Hence, a number _n_ is an amicable number if _d(d(n)) = n_ and _d(n) ≠ n_. Using this definition, we can go through all the numbers under 10000 and filter out the amicable pairs. We then take the sum to arrive at the answer.

```fsharp
let result = [1..9999] |>
                List.filter (fun x -> SumDiv x <> x && SumDiv(SumDiv x) = x) |>
                List.sum
```

This solution is relatively straightforward, but also inefficient. It takes several seconds to return the answer, which is acceptable but also clearly non-optimal. Indeed, our _Divisors_ function uses an inefficient brute-force technique which is slowing down response times. Let's see if we can optimize this.

First, rather than filtering an integer list to get our divisors, we use a bitmap to represent them: a boolean array _m_ for integer _n_ where each position _m[i]_ indicates whether or not _i_ is a proper divisor for _n_. We compute the bitmap as follows:

```fsharp
let DivMap n =
    if n = 1 then
        [| false; true |]
    else
        let m = Array.create((n / 2) + 1) false;
        m.[1] <- true
        for i in 2 .. m.Length - 1 do
            if m.[i] = false && n % i = 0 then
                m.[i] <- true
                m.[n / i] <- true
        m
```

Although more complicated, this function computes the divisors of a number significantly faster than our previous naive approach. Next, we modify our _SumDiv_ function to use it:

```fsharp
let SumDiv n =
    let mutable s = 0
    let m = DivMap n
    for i in 1 .. m.Length - 1 do
        if m.[i] = true then
            s <- s + i
    s
```

Overall these two changes get our response time down to less than a second. Still not instant, but good enough for me.
