---
layout: post
title: "Writing a decimal number as a fraction"
categories: blog
---

The decimal number _2.5_ can also be written as the fraction _5 / 2_. _19.2_ becomes _96 / 5_. _0.24_ becomes _6 / 25_. Etcetera. So how can you do this programmatically? Well, it turns out a naive solution can be obtained pretty quickly.

The idea is pretty straightforward. It comes from the observation that for any fraction _f = x / y_, increasing the value of _x_ will increase the value of _f_ and increasing the value of _y_ will decrease the value of _f_. Both _x_ and _y_ must be at least _1_. So now, if we want to find the fraction for some number _n_, we start with fraction _f = x / y_ with _x = 1_ and _y = 1_. If _f_ is smaller than _n_, we increase _x_ by one. If _f_ is larger than _n_, we increase _y_ by one. We do this until _f = n_, at which point we have rewritten _n_ as a fraction _x / y_.

The code is as simple as the description of the algorithm:

```csharp
public static Tuple<int, int> GetFraction(double number)
{
    var numerator = 1.0;
    var denominator = 1.0;
    var tmp = numerator / denominator;
    while (Math.Abs(tmp - number) > 0.0000001)
    {
        if (tmp < number)
        {
            numerator++;
        }
        else
        {
            denominator++;
            numerator = Math.Floor(number * denominator);
        }
        tmp = numerator / denominator;
    }
    return Tuple.Create(Convert.ToInt32(numerator), Convert.ToInt32(denominator));
}
```

The only difference in the code above is it does not check for equality of _f_ and _n_, but instead aims to approach the value of _n_ within a certain precision. This is done to counter inaccuracies in floating point arithmetic. Note the current precision is selected arbitrarily and is probably rather conservative. The code also has an optimization step where after incrementing _y_ it sets _x_ to be the highest value it can be under this _y_ without _f_ becoming more than _n_. After all, if we have some _n_ and some _y_ then we know that because _f = x / y = n_ we have _x = n * y_. But because _n * y_ can be a decimal number, we take the floor, which is a lower bound for _x_.
