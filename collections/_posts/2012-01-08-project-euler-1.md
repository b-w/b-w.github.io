---
layout: post
title: "Project Euler: Problem 1"
categories: blog
---

As a way of challenging myself, I though a good project which will keep me occupied for the next year (and probably beyond that) would be to take a crack at [Project Euler](http://projecteuler.net/). Project Euler provides a series of problems which can be solved through a combination of math and programming skills. The problems range from difficult to extremely difficult and they tend to lean more towards math than towards programming, so most of them will probably be out of my league, but I'll find that out as I go along. I'll also be exclusively using F# and C# to solve the problems, as these are my favorite languages and I would like to learn new things in them.

Alright, on to the first problem:

> If we list all the natural numbers below 10 that are multiples of 3 or 5, we get 3, 5, 6 and 9\. The sum of these multiples is 23.
> 
> Find the sum of all the multiples of 3 or 5 below 1000.

For most of the problems I've looked at so far, F# is going to suit me just fine. For this problem, we can start of by defining a list which contains all the numbers that are multiples of 3 or 5, using [F#'s List.filter function](http://msdn.microsoft.com/en-us/library/ee370294.aspx):

```fsharp
let numbers = List.filter (fun x -> (x % 3 = 0) || (x % 5 = 0)) [1..999]
```

We can then easily sum the result:

```fsharp
let result = List.sum numbers
```

Which gives us the desired answer of 233168.

Alright, that was easy, but the next problems will not be so trivial...
