---
layout: post
title: "Project Euler: Problem 8"
categories: blog
---

Here is the next Euler problem:

> Find the greatest product of five consecutive digits in the 1000-digit number.
> 
> (a 1000-digit number I'm not going to bother copying here)

This is the first Euler problem so far I will not be using F# for. Why? Because the set of Euler problems consists of two partially overlapping subsets: those problems that are suited for F#, and those that are suited for C#. I will try to use F# for the most part, but in this case the functional programming paradigm just doesn't feel applicable.

Now, on to the problem. I have put the 1000-digit number on a single line in a text file called _problem-8.txt_. All that remains is a basic loop to iterate over all possible consecutive digits and find the greatest product.

```csharp
static void Problem8()
{
    string number;

    using (var sr = new StreamReader("problem-8.txt"))
    {
        number = sr.ReadLine();
    }

    var max = 0;

    for (int i = 0; i < number.Length - 5; i++)
    {
        var digits = number.Substring(i, 5);
        var candidate = int.Parse(digits[0].ToString()) *
                        int.Parse(digits[1].ToString()) *
                        int.Parse(digits[2].ToString()) *
                        int.Parse(digits[3].ToString()) *
                        int.Parse(digits[4].ToString());
        if (candidate > max)
            max = candidate;
    }

    Console.WriteLine(max);
}
```

There is really nothing special or interesting here, unfortunately.
