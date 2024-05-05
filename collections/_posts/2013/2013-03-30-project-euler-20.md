---
layout: post
title: "Project Euler: Problem 20"
categories: blog
---

The last Euler problem was pretty trivial, so let's hope problem number 20 is more interesting...

> n! means n x (n − 1) x ... x 3 x 2 x 1  
>   
> For example, 10! = 10 x 9 x ... x 3 x 2 x 1 = 3628800,  
> and the sum of the digits in the number 10! is 3 + 6 + 2 + 8 + 8 + 0 + 0 = 27.  
>   
> Find the sum of the digits in the number 100!

Sadly, no. Again the presence of a modern numerics library will save us from having to do any real math. We will solve this problem using C#. We first compute _100!_:

```csharp
BigInteger number = 100;

for (int i = 1; i < 100; i++) {
    number *= i;
}
```

Trivial, when you've got [BigInteger](http://msdn.microsoft.com/en-us/library/system.numerics.biginteger.aspx) to help you out. The second step is simply converting _number_ to a string, iterating the digits, and taking the sum:

```csharp
var str = number.ToString();
var result = 0;

for (int i = 0; i < str.Length; i++) {
    result += Int32.Parse(str[i].ToString());
}

Console.WriteLine(result);
```

Nothing fancy this time either.
