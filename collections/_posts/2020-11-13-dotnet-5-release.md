---
layout: post
title: "One .NET to rule them all"
categories: blog
---

So .NET 5 was released earlier this week. This is the next .NET Core release after 3.1, and drops the "core" part from its name to signify that this is the singular .NET implementation going forward.

Microsoft continues the tradition of total confusion when naming things. We move from .NET Core 3.1 to .NET 5, which is not a continuation of .NET 4.8 (also known as the Full Framework and now in support-only) but represents the future of .NET nonetheless and is the only version in active development moving forward. ASP.NET Core 5 and Entity Framework Core 5 continue to use "core" in their names to avoid confusing them with the old ASP.NET 5 and EF 5\. You know, to keep it simple. Then there's .NET Standard, which, like the Full Framework, will continue to be supported, but won't be developed further, again effectively replacing it with .NET 5.

Right. Now that no one's confused, let's look at what's actually new.

## New in .NET 5

Feature-wise it's actually a smaller release than you might expect from its major version bump. The main goal with .NET 5 has been the unification of the SDKs and BCLs. There's also some new features in System.Text.Json, some additional stuff relating to app deployment, and a bunch of performance improvements. And, of course, support for the next version of C#.

## New in C# 9

Here's mainly where it gets interesting for developers.

### Record types

The most significant new feature of C# 9 is the addition of record types. A record is essentially some syntactic sugar that produces an immutable reference type along with some boilerplate code for things like equality checking and deconstruction.

For example, this record:

```csharp
public record Foo(int fizz, int buzz);
```

...is equivalent to the following class:

```csharp
public class Foo : IEquatable<Foo>
{
    public Foo(int fizz, int buzz)
    {
        Fizz = fizz;
        Buzz = buzz;
    }

    public int Fizz { get; init; }

    public int Buzz { get; init; }

    public override string ToString() => // impl

    public bool Equals(Foo other) => // impl

    public virtual bool Equals(Foo other) => // impl

    public static bool operator !=(Foo x, Foo y) => // impl

    public static bool operator ==(Foo x, Foo y) => // impl

    public void Deconstruct(out int fizz, out int buzz)
    {
        fizz = Fizz;
        buzz = Buzz;
    }
}
```

Of course, you don't have to keep your records limited to oneliners. The following is perfectly valid:

```csharp
public record Foo(int Fizz, int Buzz)
{
    public void Magic(int input) => SomethingHappened?.Invoke(input * Fizz);

    public string Answer { get; set; }

    public event Action<int> SomethingHappened;
}
```

This combines the boiler plate that comes with a record type with additional members you define yourself.

Inheritance is also supported:

```csharp
public record Foo(int Fizz, int Buzz);

public record Bar(string answer) : Foo(3, 5);

public record Fooz(int Fizz, int Buzz, int FizzBuzz) : Foo(Fizz, Buzz);
```

However, records can only be used in inheritance chains with other records; you can't combine them with classes or structs.

Even though records are immutable, you can create and modify a copy of one using the new "with" keyword:

```csharp
public record Foo(int Fizz, int Buzz);

var x = new Foo(3, 5);
var y = x with { Buzz = 10 };  // y.Fizz = 3
```

In case the record contains reference type fields, only the reference is copied, leaving both records with a reference to the same object. It's also possible to override the default copy behaviour:

```csharp
public record Foo(int Fizz, int Buzz)
{
    protected Foo(Foo original)
    {
        Fizz = original.Fizz * 2;
        Buzz = original.Buzz;
    }
};

var x = new Foo(3, 5);
var y = x with { Buzz = 10 };  // y.Fizz = 6
```

### Init setters

There's now a new way of creating a property setter: the "init" keyword. It denotes a property that can only be set during object initialization, which is subtly different from setting it in the object constructor. Let's examine how using the following piece of code:

```csharp
public class FooBar
{
    public FooBar()
    {
    }

    public FooBar(int magicNumber)
    {
        MagicNumber = magicNumber;  // A
    }

    public int MagicNumber { get; set; }

    public void DoSomeEvil()
    {
        MagicNumber -= 10;  // B
    }
}

var x = new FooBar(42);
var y = new FooBar
{
    MagicNumber = 42    // C
};
x.MagicNumber = 21;    // D
```

There's four different places where we try to set the MagicNumber property, and whether each one is allowed depends on how we define the property setter:

<table>
    <thead>
        <tr>
            <th scope="col">Definition</th>
            <th scope="col">A</th>
            <th scope="col">B</th>
            <th scope="col">C</th>
            <th scope="col">D</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>public int MagicNumber { get; set; }</td>
            <td>Allowed</td>
            <td>Allowed</td>
            <td>Allowed</td>
            <td>Allowed</td>
        </tr>
        <tr>
            <td>public int MagicNumber { get; private set; }</td>
            <td>Allowed</td>
            <td>Allowed</td>
            <td>Not allowed</td>
            <td>Not allowed</td>
        </tr>
        <tr>
            <td>public int MagicNumber { get; }</td>
            <td>Allowed</td>
            <td>Not allowed</td>
            <td>Not allowed</td>
            <td>Not allowed</td>
        </tr>
        <tr>
            <td>public int MagicNumber { get; init; }</td>
            <td>Allowed</td>
            <td>Not allowed</td>
            <td>Allowed</td>
            <td>Not allowed</td>
        </tr>
    </tbody>
</table>

As you can see, this allows us to set the property _once_ using the property initializer syntax, while preventing it from being set after initialization, including by methods in the class itself.

The properties a record type autogenerates use init setters.

### Pattern matching

C# 9 ramps up C#'s pattern matching capabilities yet again, this time by introducing boolean and relational patterns, allowing you to write switch statements like so:

```csharp
var rating = x.Fizz switch
{
    < 6 => "boo",
    >= 6 and < 12 => "meh",
    >= 12 => "sugoi dekai!"
};
```

It also lets you write regular if statements in different ways. For instance these two are equivalent:

```csharp
if (x.Fizz > 24 || x.Fizz < 6)

if (x.Fizz is > 24 or < 6)
```

I'm not sure yet how I feel about that last one, but there we are.

## Wrapping up

.NET 5 and C# 9 feel like nice, but small, incremental releases. Record types seem like the only new things that might have an impact on my day-to-day work, so I'm curious about trying those out in practice once I've got a .NET 5 project to play with. I've highlighted a few of the features that were most interesting to me, but this isn't the only new stuff. For a more complete overview, I suggest you read the changenotes on MSDN for [.NET 5](https://docs.microsoft.com/en-us/dotnet/core/dotnet-five) and [C# 9](https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-9).
