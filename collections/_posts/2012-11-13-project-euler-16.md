---
layout: post
title: "Project Euler: Problem 16"
categories: blog
---

It's been a while since we last did any Project Euler problems, so why not have a go?

> 2<sup>15</sup> = 32768 and the sum of its digits is 3 + 2 + 7 + 6 + 8 = 26.  
>   
> What is the sum of the digits of the number 2<sup>1000</sup>?

Unfortunately, this is another one of those Euler problems that shows its age. While _2<sup>1000</sup>_ far exceeds the domain of 64-bit integers, it is nothing a modern numerics library can't handle. In our case, we use F#'s `BigInteger`:

```fsharp
open System

let num = 2I ** 1000
```

We then convert this number to a string, and make a list of all the digits by parsing them one at the time:

```fsharp
let str = num.ToString()

let numList = [ for i in 0..str.Length - 1 do yield UInt64.Parse(str.[i].ToString()) ]
```

Now we simply take the sum of this list to arrive at the answer:

```fsharp
let result = numList |> List.sum
```

There are, unfortunately, no tricks involved this time.
