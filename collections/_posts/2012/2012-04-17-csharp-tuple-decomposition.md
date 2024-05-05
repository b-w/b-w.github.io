---
layout: post
title: "Easy C# tuple decomposition through extension methods"
categories: blog
---

The [tuple class](http://msdn.microsoft.com/en-us/library/system.tuple.aspx) has been part of the .NET Framework since version 4\. A tuple is a basic data structure which contains a fixed, ordered number of objects or values. An example would be the 3-tuple (or triplet) containing an employee's number, name, and the date they were hired by the company: _(id, name, hire_date)_. The items in the tuple can be of different data types. They are also read-only, and cannot be changed once the tuple has been created.

Consider our example 3-tuple. Here's how it would be used in C#:

```csharp
var employeeInfo = Tuple.Create(1234, "Bart Wolff", DateTime.Now);

var id = employeeInfo.Item1;
var name = employeeInfo.Item2;
var hireDate = employeeInfo.Item3;

someObject.SomeMethod(employeeInfo);
```

Note that they are in some respects similar to anonymous types, although tuples have the added benefit that because they are not anonymous, they _can_ be used in all cases where types are needed, for example as return types for methods, as parameters, as class fields, etcetera. In a way, this allows you to have methods which return more than one type:

```csharp
public Tuple<int, string> GetEmployeeInfo()
{
    return new Tuple<int, string>(1234, "Bart Wolff");
}
```

Here I show the other way of constructing a tuple. The first way has been shown above. The following statement is equivalent to this:

```csharp
public Tuple<int, string> GetEmployeeInfo()
{
    return Tuple.Create(1234, "Bart Wolff");
}
```

Observe how this allows methods to easily return multiple values, without having to resort to custom structs or classes.

## Decomposition

Obviously, once you have a tuple, you want to be able to decompose it to get to the data. So how do you do this? Well, the only way that is offered by the tuple class itself is through the _Item1_, _Item2_, _Item3_, etc. properties, as shown above. Somehow this doesn't feel very pretty. Especially when in F# you can do the following:

```fsharp
> let SomeFunction n = (n + 1, n + 10, n + 100);;

val SomeFunction : int -> int * int * int

> SomeFunction 5;;

val it : int * int * int = (6, 15, 105)

> let (x, y, z) = SomeFunction 5;;

val z : int = 105
val y : int = 15
val x : int = 6

>
```

Unfortunately, the C# syntax simply doesn't offer this kind of simplistic beauty. The best way I've been able to approach it is by using [extension methods](http://msdn.microsoft.com/en-us/library/bb383977.aspx), like the following:

```csharp
public static void Decompose<T1, T2>(this Tuple<T1, T2> tuple, out T1 item1, out T2 item2)
{
    item1 = tuple.Item1;
    item2 = tuple.Item2;
}
```

This way, you can decompose any tuple in a single line:

```csharp
var carInfo = Tuple.Create("Mazda", new DateTime(1989, 1, 1));

string brand;
DateTime manDate;

carInfo.Decompose(out brand, out manDate);
```

It still requires all variables to be present before decomposition, of course, but once you have those set up your code will be far more efficient:

```csharp
public void ProcessCars(List<Tuple<string, DateTime>> carList)
{
    string make;
    DateTime manDate;

    foreach (var tuple in carList)
    {
        tuple.Decompose(out make, out manDate);

        // do something
    }
}
```

You'll never get the ease tuples offer in F#, but it can make your life a little easier in the right cases. So go ahead, download the source, or view the full code listing after the break. I've already done all the boring copy-paste work for you.

```csharp
using System;

namespace Com.BartWolff.Extensions
{
    /// <summary>
    /// Extension methods for System.Tuple
    /// </summary>
    public static class TupleExtensions
    {
        /// <summary>
        /// Decomposes the specified tuple.
        /// </summary>
        /// <typeparam name="T1">The type of the first item in the tuple.</typeparam>
        /// <param name="tuple">The tuple.</param>
        /// <param name="item1">Will contain the first item in the tuple.</param>
        public static void Decompose<T1>(this Tuple<T1> tuple, out T1 item1)
        {
            item1 = tuple.Item1;
        }

        /// <summary>
        /// Decomposes the specified tuple.
        /// </summary>
        /// <typeparam name="T1">The type of the first item in the tuple.</typeparam>
        /// <typeparam name="T2">The type of the second item in the tuple.</typeparam>
        /// <param name="tuple">The tuple.</param>
        /// <param name="item1">Will contain the first item in the tuple.</param>
        /// <param name="item2">Will contain the second item in the tuple.</param>
        public static void Decompose<T1, T2>(this Tuple<T1, T2> tuple, out T1 item1, out T2 item2)
        {
            item1 = tuple.Item1;
            item2 = tuple.Item2;
        }

        /// <summary>
        /// Decomposes the specified tuple.
        /// </summary>
        /// <typeparam name="T1">The type of the first item in the tuple.</typeparam>
        /// <typeparam name="T2">The type of the second item in the tuple.</typeparam>
        /// <typeparam name="T3">The type of the third item in the tuple.</typeparam>
        /// <param name="tuple">The tuple.</param>
        /// <param name="item1">Will contain the first item in the tuple.</param>
        /// <param name="item2">Will contain the second item in the tuple.</param>
        /// <param name="item3">Will contain the third item in the tuple.</param>
        public static void Decompose<T1, T2, T3>(this Tuple<T1, T2, T3> tuple, out T1 item1, out T2 item2, out T3 item3)
        {
            item1 = tuple.Item1;
            item2 = tuple.Item2;
            item3 = tuple.Item3;
        }

        /// <summary>
        /// Decomposes the specified tuple.
        /// </summary>
        /// <typeparam name="T1">The type of the first item in the tuple.</typeparam>
        /// <typeparam name="T2">The type of the second item in the tuple.</typeparam>
        /// <typeparam name="T3">The type of the third item in the tuple.</typeparam>
        /// <typeparam name="T4">The type of the fourth item in the tuple.</typeparam>
        /// <param name="tuple">The tuple.</param>
        /// <param name="item1">Will contain the first item in the tuple.</param>
        /// <param name="item2">Will contain the second item in the tuple.</param>
        /// <param name="item3">Will contain the third item in the tuple.</param>
        /// <param name="item4">Will contain the fouth item in the tuple.</param>
        public static void Decompose<T1, T2, T3, T4>(this Tuple<T1, T2, T3, T4> tuple, out T1 item1, out T2 item2, out T3 item3, out T4 item4)
        {
            item1 = tuple.Item1;
            item2 = tuple.Item2;
            item3 = tuple.Item3;
            item4 = tuple.Item4;
        }

        /// <summary>
        /// Decomposes the specified tuple.
        /// </summary>
        /// <typeparam name="T1">The type of the first item in the tuple.</typeparam>
        /// <typeparam name="T2">The type of the second item in the tuple.</typeparam>
        /// <typeparam name="T3">The type of the third item in the tuple.</typeparam>
        /// <typeparam name="T4">The type of the fourth item in the tuple.</typeparam>
        /// <typeparam name="T5">The type of the fifth item in the tuple.</typeparam>
        /// <param name="tuple">The tuple.</param>
        /// <param name="item1">Will contain the first item in the tuple.</param>
        /// <param name="item2">Will contain the second item in the tuple.</param>
        /// <param name="item3">Will contain the third item in the tuple.</param>
        /// <param name="item4">Will contain the fouth item in the tuple.</param>
        /// <param name="item5">Will contain the fifth item in the tuple.</param>
        public static void Decompose<T1, T2, T3, T4, T5>(this Tuple<T1, T2, T3, T4, T5> tuple, out T1 item1, out T2 item2, out T3 item3, out T4 item4, out T5 item5)
        {
            item1 = tuple.Item1;
            item2 = tuple.Item2;
            item3 = tuple.Item3;
            item4 = tuple.Item4;
            item5 = tuple.Item5;
        }

        /// <summary>
        /// Decomposes the specified tuple.
        /// </summary>
        /// <typeparam name="T1">The type of the first item in the tuple.</typeparam>
        /// <typeparam name="T2">The type of the second item in the tuple.</typeparam>
        /// <typeparam name="T3">The type of the third item in the tuple.</typeparam>
        /// <typeparam name="T4">The type of the fourth item in the tuple.</typeparam>
        /// <typeparam name="T5">The type of the fifth item in the tuple.</typeparam>
        /// <typeparam name="T6">The type of the sixth item in the tuple.</typeparam>
        /// <param name="tuple">The tuple.</param>
        /// <param name="item1">Will contain the first item in the tuple.</param>
        /// <param name="item2">Will contain the second item in the tuple.</param>
        /// <param name="item3">Will contain the third item in the tuple.</param>
        /// <param name="item4">Will contain the fouth item in the tuple.</param>
        /// <param name="item5">Will contain the fifth item in the tuple.</param>
        /// <param name="item6">Will contain the sixth item in the tuple.</param>
        public static void Decompose<T1, T2, T3, T4, T5, T6>(this Tuple<T1, T2, T3, T4, T5, T6> tuple, out T1 item1, out T2 item2, out T3 item3, out T4 item4, out T5 item5, out T6 item6)
        {
            item1 = tuple.Item1;
            item2 = tuple.Item2;
            item3 = tuple.Item3;
            item4 = tuple.Item4;
            item5 = tuple.Item5;
            item6 = tuple.Item6;
        }

        /// <summary>
        /// Decomposes the specified tuple.
        /// </summary>
        /// <typeparam name="T1">The type of the first item in the tuple.</typeparam>
        /// <typeparam name="T2">The type of the second item in the tuple.</typeparam>
        /// <typeparam name="T3">The type of the third item in the tuple.</typeparam>
        /// <typeparam name="T4">The type of the fourth item in the tuple.</typeparam>
        /// <typeparam name="T5">The type of the fifth item in the tuple.</typeparam>
        /// <typeparam name="T6">The type of the sixth item in the tuple.</typeparam>
        /// <typeparam name="T7">The type of the seventh item in the tuple.</typeparam>
        /// <param name="tuple">The tuple.</param>
        /// <param name="item1">Will contain the first item in the tuple.</param>
        /// <param name="item2">Will contain the second item in the tuple.</param>
        /// <param name="item3">Will contain the third item in the tuple.</param>
        /// <param name="item4">Will contain the fouth item in the tuple.</param>
        /// <param name="item5">Will contain the fifth item in the tuple.</param>
        /// <param name="item6">Will contain the sixth item in the tuple.</param>
        /// <param name="item7">Will contain the seventh item in the tuple.</param>
        public static void Decompose<T1, T2, T3, T4, T5, T6, T7>(this Tuple<T1, T2, T3, T4, T5, T6, T7> tuple, out T1 item1, out T2 item2, out T3 item3, out T4 item4, out T5 item5, out T6 item6, out T7 item7)
        {
            item1 = tuple.Item1;
            item2 = tuple.Item2;
            item3 = tuple.Item3;
            item4 = tuple.Item4;
            item5 = tuple.Item5;
            item6 = tuple.Item6;
            item7 = tuple.Item7;
        }

        /// <summary>
        /// Decomposes the specified tuple.
        /// </summary>
        /// <typeparam name="T1">The type of the first item in the tuple.</typeparam>
        /// <typeparam name="T2">The type of the second item in the tuple.</typeparam>
        /// <typeparam name="T3">The type of the third item in the tuple.</typeparam>
        /// <typeparam name="T4">The type of the fourth item in the tuple.</typeparam>
        /// <typeparam name="T5">The type of the fifth item in the tuple.</typeparam>
        /// <typeparam name="T6">The type of the sixth item in the tuple.</typeparam>
        /// <typeparam name="T7">The type of the seventh item in the tuple.</typeparam>
        /// <typeparam name="TRest">The type of the remaining components in the tuple.</typeparam>
        /// <param name="tuple">The tuple.</param>
        /// <param name="item1">Will contain the first item in the tuple.</param>
        /// <param name="item2">Will contain the second item in the tuple.</param>
        /// <param name="item3">Will contain the third item in the tuple.</param>
        /// <param name="item4">Will contain the fouth item in the tuple.</param>
        /// <param name="item5">Will contain the fifth item in the tuple.</param>
        /// <param name="item6">Will contain the sixth item in the tuple.</param>
        /// <param name="item7">Will contain the seventh item in the tuple.</param>
        /// <param name="rest">Will contain the remaining components in the tuple.</param>
        public static void Decompose<T1, T2, T3, T4, T5, T6, T7, TRest>(this Tuple<T1, T2, T3, T4, T5, T6, T7, TRest> tuple, out T1 item1, out T2 item2, out T3 item3, out T4 item4, out T5 item5, out T6 item6, out T7 item7, out TRest rest)
        {
            item1 = tuple.Item1;
            item2 = tuple.Item2;
            item3 = tuple.Item3;
            item4 = tuple.Item4;
            item5 = tuple.Item5;
            item6 = tuple.Item6;
            item7 = tuple.Item7;
            rest = tuple.Rest;
        }
    }
}
```

Happy coding!
