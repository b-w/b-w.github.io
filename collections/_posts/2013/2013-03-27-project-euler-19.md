---
layout: post
title: "Project Euler: Problem 19"
categories: blog
---

Let's do another Euler problem, shall we?

> You are given the following information, but you may prefer to do some research for yourself.  
>   
>     1 Jan 1900 was a Monday.  
>   
>     Thirty days has September,  
>     April, June and November.  
>     All the rest have thirty-one,  
>     Saving February alone,  
>     Which has twenty-eight, rain or shine.  
>     And on leap years, twenty-nine.  
>   
>     A leap year occurs on any year evenly divisible by 4, but not on a century unless it is divisible by 400.  
>   
> How many Sundays fell on the first of the month during the twentieth century (1 Jan 1901 to 31 Dec 2000)?

I'm afraid it's another trivial one today. Modern languages and frameworks such as .NET have no problem dealing with dates and times. After all, it's such a common problem that it would be strange if there wouldn't be a solution for it baked right into the framework. But I digress.

Today's problem will be solved in F#. Ignoring all the superfluous information we are given by the problem statement, we start off by defining a sequence which generates all first-of-the-month days between January 1, 1901 and December 31, 2000:

```fsharp
open System

let Days = (new DateTime(1901, 1, 1)) |> Seq.unfold (fun state ->
    if state.Year = 2001 then None
    else Some(state, state.AddMonths(1)))
```

We can then filter this sequence by only allowing those days that also happen to fall on a Sunday:

```fsharp
let result = Days |>
                Seq.filter (fun x -> x.DayOfWeek = DayOfWeek.Sunday) |>
                Seq.length
```

Taking the length of the resulting sequence gives us the answer. Note that we were able to translate the question almost 1:1 into F# expressions, without the need for any math at all.
