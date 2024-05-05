---
layout: post
title: "Project Euler: Problem 10"
categories: blog
---

Here's another Project Euler problem for us:

> The sum of the primes below 10 is 2 + 3 + 5 + 7 = 17.  
>   
> Find the sum of all the primes below two million.

This didn't strike me as a particularly complicated problem, and neither was my initial attempt at solving it:

```fsharp
open System

let Candidates = seq { for i in 2UL..1999999UL -> i }

let IsPrime n =
    [2UL..uint64(Math.Sqrt(float(n)))] |> List.forall (fun x -> n % x <> 0UL)

let PrimeNumbers = seq { yield! Candidates |> Seq.filter (fun x -> IsPrime x) }

let result = PrimeNumbers |> Seq.sum
```

This solution is simple, straightforward, correct, and painfully slow. Depending on how much you know about F#, programming, or math, you'll realize my method of computing the prime numbers required is very naive and inefficient. Indeed, this code will run for several minutes on my laptop before returning the answer.

So, can we do better? Well, I'm not a mathematician myself, so I don't know a lot about prime numbers, but surely such an elemental problem as generating a sequence of them has been solved by smarter people a long time ago? I took to the Googles, and pretty soon I learned about prime sieves, and more specifically the [sieve of Eratosthenes](http://en.wikipedia.org/wiki/Sieve_of_Eratosthenes), which turns out to be an ancient method of computing all prime numbers up to a given point. Exactly what we need, and more importantly, the sieve can be implemented as an _O(n log log n)_ algorithm. Here's my version in F#:

```fsharp
let PrimeSieve n =
    let s = Array.create (n+1) true
    s.[0] <- false
    s.[1] <- false
    for i in [2..n/2] do
        if s.[i] = true then
            for j in (2 |> Seq.unfold (fun x -> if i * x <= n then Some(i * x, x + 1) else None)) do
                s.[j] <- false
    s
```

This function returns an array of booleans of size _n_, where each element at position _x_ is true if _x_ is prime, and false if _x_ is not. Now if we want to compute the primes below two million, we can simply say:

```fsharp
let Primes = PrimeSieve 1999999
```

On my machine, this computation takes about a single second. Now the remaining steps are trivial:

```fsharp
let result = [for i in 2..1999999 do
                if Primes.[i] then yield i] |> List.sumBy (fun x -> uint64(x))
```

Except this time, we get the solution in a few seconds rather than a few minutes. Quite an improvement!
