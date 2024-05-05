---
layout: post
title: "A classic example of fine craftsmanship"
categories: blog
---

Of all the bad code I've seen (some of which I've written myself), this classical example is still one of the finer and more hilarious examples of a truly creative mind. From memory it looked something like this:

```csharp
bool IsNumberNegative(int number)
{
    try
    {
        Math.Sqrt(number);
        return true;
    }
    catch (MathException)
    {
        return false;
    }
}
```

Not sure what language it was originally in. I came across it a few years back, I believe on a codeproject.com thread about bad code, and it's been hard to top ever since. I don't know if it was posted as a joke or if a human being somewhere actually wrote this, but I sincerely hope it's the former.
