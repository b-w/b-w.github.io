---
layout: post
title: "Project Euler: Problem 4"
categories: blog
---

Time for another Project Euler problem:

> A palindromic number reads the same both ways. The largest palindrome made from the product of two 2-digit numbers is 9009 = 91 × 99.  
>   
> Find the largest palindrome made from the product of two 3-digit numbers.

Alright, an obvious function we are going to need is something for determining whether a number is a palindrome. We'll go ahead and use the following function:

```fsharp
let IsPalindrome n =
    let s = n.ToString()
    let sr = new string(Array.rev(s.ToCharArray()))
    s = sr
```

It might not be the most efficient implementation, but that's not an issue yet at this point (it will be in later Euler problems). Now that we have this function, the rest is trivial:

```fsharp
let palindromes = [ for x in 100..999 do
                    for y in 100..999 do
                        if IsPalindrome(x * y) then yield x * y ]

let result = List.max palindromes
```

We first define a list of all possible products of 3-digit numbers (again not in the most efficient fashion, but ignore this for now) and filter out any non-palindromes. We then simply take the maximum of this list to arrive at the correct answer.

_(note: starting with this post, I will no longer be posting the final answers at the bottom of each post)_
