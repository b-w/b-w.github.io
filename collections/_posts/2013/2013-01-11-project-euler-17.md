---
layout: post
title: "Project Euler: Problem 17"
categories: blog
---

Last year I've managed to complete 16 Euler problems. And even though I've been quite busy over the last 6 months, I'm still a bit disappointed in myself. Anyhow, let's see how far we can get this year. Here's the next problem:

> If the numbers 1 to 5 are written out in words: one, two, three, four, five, then there are 3 + 3 + 5 + 4 + 4 = 19 letters used in total.  
>   
> If all the numbers from 1 to 1000 (one thousand) inclusive were written out in words, how many letters would be used?  
>   
> NOTE: Do not count spaces or hyphens. For example, 342 (three hundred and forty-two) contains 23 letters and 115 (one hundred and fifteen) contains 20 letters. The use of "and" when writing out numbers is in compliance with British usage.

We'll be solving this one in F#. The obvious starting point is a function that will give us the number of letters that some number uses when written in words. It should be possible to make this function recursive, minimizing the amount of information we need to hard-code in there. For instance, the number 384 is written as three hundred and eighty-four, which can be split into writing 3, 100, 80, and 4.

In the general case, numbers of shape _xy_ can be written as _x0-y_ (e.g. 84 is 80-4, or eighty-four). Numbers of the shape _xyz_ can be written as _x_ hundred and _y0-z_ (e.g. 384 is 3 hundred and 80-4, or three hundred and eighty-four). We write a function that exploits this recursion:

```fsharp
let rec NumLetters n =
    match n with
    | 0 -> 0
    | 1 -> "one".Length
    | 2 -> "two".Length
    | 3 -> "three".Length
    | 4 -> "four".Length
    | 5 -> "five".Length
    | 6 -> "six".Length
    | 7 -> "seven".Length
    | 8 -> "eight".Length
    | 9 -> "nine".Length
    | 10 -> "ten".Length
    | 11 -> "eleven".Length
    | 12 -> "twelve".Length
    | 13 -> "thirteen".Length
    | 14 -> "fourteen".Length
    | 15 -> "fifteen".Length
    | 16 -> "sixteen".Length
    | 17 -> "seventeen".Length
    | 18 -> "eighteen".Length
    | 19 -> "nineteen".Length
    | 20 -> "twenty".Length
    | 30 -> "thirty".Length
    | 40 -> "forty".Length
    | 50 -> "fifty".Length
    | 60 -> "sixty".Length
    | 70 -> "seventy".Length
    | 80 -> "eighty".Length
    | 90 -> "ninety".Length
    | 1000 -> "onethousand".Length
    | _ when n < 100 -> (NthDigit(n, 2) * 10 |> NumLetters) +
                        (NthDigit(n, 1) |> NumLetters)
    | _ when n >= 100 -> (NthDigit(n, 3) |> NumLetters) +
                            "hundred".Length +
                            (n % 100 |> NumLetters) +
                            if n % 100 <> 0 then "and".Length else 0
    | _ -> 0
```

The _NthDigit_ function is defined as follows, and is used for extracting the _n_-th digit in a multi-digit number:

```fsharp
let NthDigit(n, d) = (n / int(10.0 ** float(d - 1))) % 10
```

In the main function there are a bunch of numbers that need to be hard-coded. It gets more interesting when a number doesn't exactly fit in one of these cases. For example, 84 is handled by

```fsharp
| _ when n < 100 -> (NthDigit(n, 2) * 10 |> NumLetters) +
                    (NthDigit(n, 1) |> NumLetters)
```

where its digits get split into 80 and 4, which then get looked up again. The number 384 is handled by

```fsharp
| _ when n >= 100 -> (NthDigit(n, 3) |> NumLetters) +
                        "hundred".Length +
                        (n % 100 |> NumLetters) +
                        if n % 100 <> 0 then "and".Length else 0
```

where the 3 gets split off and looked up, the word "hundred" gets included, and the 84 part is handled recursively. There's also a case for including the word "and", which is used whenever the number is not an exact multiple of 100.

After finishing this convoluted function, the final answer is obtained rather easily:

```fsharp
let result = [1..1000] |> List.sumBy (fun x -> NumLetters x)
```

Kind of an odd Euler problem this was, but oh well...
