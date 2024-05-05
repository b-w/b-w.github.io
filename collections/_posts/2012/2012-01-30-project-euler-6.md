---
layout: post
title: "Project Euler: Problem 6"
categories: blog
---

Alright, so yesterday's Euler problem didn't really count. Here's another one:

> The sum of the squares of the first ten natural numbers is,  
> 1<sup>2</sup> + 2<sup>2</sup> + ... + 10<sup>2</sup> = 385  
>   
> The square of the sum of the first ten natural numbers is,  
> (1 + 2 + ... + 10)<sup>2</sup> = 552 = 3025  
>   
> Hence the difference between the sum of the squares of the first ten natural numbers and the square of the sum is 3025 − 385 = 2640.  
>   
> Find the difference between the sum of the squares of the first one hundred natural numbers and the square of the sum.

Alright, so this one is pretty trivial as well...

```fsharp
let sumSquare = List.sumBy (fun x -> x ** 2.0) [1.0..100.0]
let squareSum = List.sum [1.0..100.0] ** 2.0

let result = squareSum - sumSquare
```

First, we compute the sum of the squares by feeding a list of the numbers 1 to 100 into the List.sumBy function, transforming each element by taking its square before summing. Next, we take the sum of numbers 1 to 100 and square that to get the square of the sum. We then subtract the two to arrive at the answer.

_(note: I had anticipated these early problems to be slightly less... trivial... but, alas, they are not)_
