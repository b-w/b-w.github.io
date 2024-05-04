---
layout: post
title: "Some F# / C# interopping"
categories: blog
---

Since F# and C# both have their uses, it shouldn't be surprising that you sometimes need both of them in one solution. So how does an F# project talk to a C# library, or vice versa? The answer is not always trivial or intuitive. In this post, I'll go over some of the basics required to get started.

## From F# to C#

We'll start with the trivial stuff. Having F# talk to a C# library is literally as easy as referencing the C# project from the F# one. Consider a C# library containing the following class:

```csharp
public class SimpleClass
{
    public SimpleClass(string someValue)
    {
        MyValue = someValue;
    }

    public string MyValue { get; set; }

    public IEnumerable<int> GetList()
    {
        int i = 1;
        while (i < 10)
        {
            yield return i;
            i++;
        }
    }

    public int ApplyFunction(Func<int, int> fn)
    {
        return fn(5);
    }
}
```

After referencing the library from our F# project, we can do the following:

```fsharp
open LibCSharp						// library containing SimpleClass

let test = new SimpleClass("henk")

test.MyValue <- "bob"

let a = test.ApplyFunction(fun x -> x * 2)		// a : int

let b = test.GetList()					// b : seq<int>
```

Simple as pie. Functions stay functions, properties are still properties. Everything works as you would expect, basically. Note that F# anonymous functions can serve as C# delegates without any conversion. The reverse is not the case, as we will see.

## From C# to F#

With the easy stuff out the way, let's find out how the other way around works. First, namespaces in F#. There are [multiple ways to do it](http://msdn.microsoft.com/en-us/library/dd233219.aspx), but I prefer using just the module keyword as it seems less confusing. Imagine we have a math library with a simple function in it:

```fsharp
module LibFSharp.Math

open System

let rec Factorial n =
    match n with
    | 0 -> 1
    | 1 -> 1
    | _ -> n * Factorial (n - 1)
```

We would like to use this library to compute the factorial of a number from our C# console application. So how do we do it? Well, this might surprise you, but again it is as simple as referencing the F# library from the C# project. After that, we can do:

```csharp
using LibFSharp;

// more

static void Main(string[] args)
{
    Console.WriteLine(LibFSharp.Math.Factorial(6));

    Console.ReadKey();
}
```

Which correctly outputs 720 to the console. Similarly, the following sequence:

```fsharp
let IsPrime n =
    [2UL..uint64(Math.Sqrt(float(n)))] |> List.forall (fun x -> n % x <> 0UL)

let Primes =
    let rec gen x = seq {
        if IsPrime x then yield x
        yield! gen(x + 1UL)
    }
    gen 2UL
```

...gets treated as an IEnumerable<long> in our C# app, allowing us to simply do this:

```csharp
foreach (var i in LibFSharp.Math.Primes.Take(10))
{
    Console.WriteLine(i);
}
```

...to output the first 10 prime numbers.

So what is the big deal then? All of this seems pretty trivial and intuitive. Where's the catch? Well, so far we've only been working with primitive data types, which are not unique to F#. Remember when I mentioned how F# lambdas can serve as C# delegates without much of a hassle? Not so for the other way around. Consider the following, which is a basic implementation of a map function in F#:

```fsharp
let rec Map fn lst =
    match lst with
    | [] -> []
    | h :: t -> fn h :: Map fn t
```

So how do we call that from C#? Probably just like so:

```csharp
foreach (var i in LibFSharp.Math.Map<int, int>(x => x * 2, Enumerable.Range(1, 5)))
{
    Console.WriteLine(i);
}
```

...right? Unfortunately, no. Here's where the fun begins. In F#, the map function has type _('a -> 'b) -> 'a list -> 'b list_, which involves F# lambdas and lists. C# cannot work with those out of the box. So how do we proceed? First, your project needs to reference FSharp.Core.dll, and you need to do some imports:

```csharp
using Microsoft.FSharp.Core;
using Microsoft.FSharp.Collections;
```

These give you access to the required conversion functions, which are needed to translate your delegate and your Enumerable into something F# understands:

```csharp
var fn = FuncConvert.ToFSharpFunc<int, int>(x => x * 2);
var lst = ListModule.OfSeq(Enumerable.Range(1, 5));

foreach (var i in LibFSharp.Math.Map<int, int>(fn, lst))
{
    Console.WriteLine(i);
}
```

...which is much less pretty. You can of course write an F# function which explicitly expects a _Func<T, TResult>_ parameter which you then invoke, but I personally think this is ugly as you're now making it a hassle for this function to be used within F# itself. It's just a matter of which hoop you prefer to jump through.

Another thing you'll notice sooner or later is that C# cannot deal with [function currying](http://en.wikipedia.org/wiki/Currying). In F#, consider the following function:

```fsharp
let TripleAddition a b c = a + b + c
```

This function has type _int -> int -> int -> int_, and in F# it's perfectly legal to do partial application like so:

```fsharp
let a = TripleAddition 5 6		// type (int -> int)

let x = a 7						// 18
let y = a 8						// 19
```

However, when calling this function from our C# application, it is represented to us as a function taking three arguments and returning an integer. The following C# code will not compile:

```csharp
LibFSharp.Math.TripleAddition(5, 6)
```

...and will throw an error stating there is no overloaded method taking 2 arguments.

## Wrapping up

So there you have it. I hope this will help you get started using F# from your C# projects. It's not that much of a hassle, but it's not trivial either and there's a few gotchas you need to be aware of. I certainely haven't touched on all of them here, but I covered some of the basics that you might be having trouble with figuring out on your own. The bottom line is if you want to talk to F#, you're going to have to do it on its own terms.
