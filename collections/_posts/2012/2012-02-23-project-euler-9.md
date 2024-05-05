---
layout: post
title: "Project Euler: Problem 9"
categories: blog
---

Another Euler problem to take on:

> A Pythagorean triplet is a set of three natural numbers, a < b < c, for which,  
> a<sup>2</sup> + b<sup>2</sup> = c<sup>2</sup>  
>   
> For example, 3<sup>2</sup> + 4<sup>2</sup> = 9 + 16 = 25 = 5<sup>2</sup>.  
>   
> There exists exactly one Pythagorean triplet for which a + b + c = 1000.  
> Find the product abc.

Alright, so we are dealing with something called Pythagorean triplets. Before looking any further into the problem, we can safely assume we're probably going to have to test triplets to see whether they are Pythagorean, so we first go ahead and define a function for that:

```fsharp
let IsPythTriplet(a, b, c) =
    a ** 2.0 + b ** 2.0 = c ** 2.0
```

Now on to the actual problem. It appears we're finding triplets (a, b, c) which are not only Pythagorean, but also have a sum which is equal to 1000 and have a < b < c. This restricts our answer domain quite a bit, and we can use these restrictions to create an efficient sequence:

```fsharp
let candidates = seq { for i in 1.0..498.0 do
                        for j in (i+1.0)..499.0 do
                            let k = 1000.0 - i - j
                            if k > j then yield (i, j, k)
                        }
```

We can reason that neither a or b should ever be larger or equal to 500, because in that case the value of c (which needs to satisfy b < c) will violate the a + b + c = 1000 property. This reduces the size of our answer domain and speeds up computation. Now that we have the candidate triplets satisfying a < b < c and a + b + c = 1000, all that's left is to weed out the Pythagorean triplets:

```fsharp
let (a, b, c) = candidates |> Seq.filter IsPythTriplet |> Seq.head

let result = a * b * c
```

Note we simply take the sequence head even though there only should be one element remaining.
