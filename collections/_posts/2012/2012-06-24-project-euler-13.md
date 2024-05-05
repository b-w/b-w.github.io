---
layout: post
title: "Project Euler: Problem 13"
categories: blog
---

It's been over a month since I did any Euler problems, so I'll be posting some of those the coming week. Here's problem 13:

> Work out the first ten digits of the sum of the following one-hundred 50-digit numbers.  
>   
> 37107287533902102798797998220837590246510135740250  
> 46376937677490009712648124896970078050417018260538  
> 74324986199524741059474233309513058123726617309629  
> 91942213363574161572522430563301811072406154908250  
> 23067588207539346171171980310421047513778063246676  
> Etc...

 We'll be solving this one in C# again. First, I've saved the numbers in a file called _problem-13.txt_ for easy access. This problem would be non-trivial if I had to solve it using integers or longs, but since I have access to the BigInteger class from System.Numerics, well...

```csharp
var numbers = new BigInteger[100];

using (var sr = new StreamReader("problem-13.txt"))
{
    for (int i = 0; i < 100; i++)
    {
        numbers[i] = BigInteger.Parse(sr.ReadLine());
    }
}

BigInteger total = 0;

for (int i = 0; i < 100; i++)
{
    total += numbers[i];
}

Console.WriteLine(total.ToString().Substring(0, 10));
```

There's really nothing more than reading and parsing the numbers from the file, adding them together, and printing the first ten digits. By the way, they apparently mean the first ten most-significant digits, as opposed to my original guess of the first ten least-significant digits.
