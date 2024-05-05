---
layout: post
title: "Project Euler: Problem 7"
categories: blog
---

Continuing the Euler adventure, here is problem 7:

> By listing the first six prime numbers: 2, 3, 5, 7, 11, and 13, we can see that the 6th prime is 13.  
>   
> What is the 10001st prime number?

Alright, so how do we tackle this? Well, first of all, note we are working with prime numbers, so we'll probably need some way of testing whether a given number is prime or not. Let's define a function for this:

```fsharp
open System

let IsPrime n =
    [2UL..uint64(Math.Sqrt(float(n)))] |> List.forall (fun x -> n % x <> 0UL)
```

The System namespace is needed for the Math class we're using. Other than that, this function is an implementation of a technique called [trial division](http://en.wikipedia.org/wiki/Trial_division), which can be used to factor a number and therefor also to test for primeness. As Wikipedia says, it's not a very efficient technique, but it is also easy to understand (and implement) and for now it will do just fine.

The next obvious thing to do is to create some list or sequence of prime numbers to iterate over. We'll use a sequence because it has the nice property of being lazy: it only expands (is generated / evaluated) when it is needed.

```fsharp
let Primes =
    let rec gen x = seq {
        if IsPrime x then yield x
        yield! gen(x + 1UL)
    }
    gen 2UL
```

The prime sequence above checks all numbers but it only yields those which are prime. Also note the sequence is infinite. However, when we request the 10001st element (which will be the 10001st number to be yielded by the sequence, and therefor the 10001st prime number), only the first 10001 elements actually get generated to accomplish this.

```fsharp
let result = Primes |> Seq.nth 10000
```

We need element 10000 because sequences (like everything else) use a 0-based index.
