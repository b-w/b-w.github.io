---
layout: post
title: "Intercepting interpolated strings in C#"
categories: blog
---

If you use C#, you're probably familiar with [interpolated strings](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/interpolated-strings), introduced in C# 6\. In case you're not, this:

```csharp
string.Format("Hello, {0}! The year is {1}.", name, DateTime.Now.Year)
```

...is equivalent to this:

```csharp
$"Hello, {name}! The year is {DateTime.Now.Year}."
```

Interpolated strings integrate the argument expressions into the format string itself, rather than keeping them separate. It makes the whole thing quite a bit more readable.

But interpolated strings can do more than that. Consider the following program:

```csharp
static void Main(string[] args)
{
    var name = "World";
    ProcessString($"Hello, {name}! The year is {DateTime.Now.Year}.");
}

static void ProcessString(string arg)
{
    Console.WriteLine("Received the following string:");
    Console.Write('\t');
    Console.WriteLine(arg);
}
```

This program produces the following output:

```
    Received the following string:
        Hello, World! The year is 2017.
```

Not entirely unexpected. But what if we replace the ProcessString method with the following:

```csharp
static void Main(string[] args)
{
    var name = "World";
    ProcessString($"Hello, {name}! The year is {DateTime.Now.Year}.");
}

static void ProcessString(FormattableString arg)
{
    Console.WriteLine("Received the following FormattableString:");
    Console.Write('\t');
    Console.WriteLine(arg.Format);
    foreach (var item in arg.GetArguments())
    {
        Console.WriteLine("\t{0} ({1})", item, item.GetType());
    }
}
```

Note we haven't changed the Main method at all. This program outputs the following:

```
    Received the following FormattableString:
        Hello, {0}! The year is {1}.
        World (System.String)
        2017 (System.Int32)
```

Some things to note:

*   Interpolated strings look a lot like the old string formats, under the hood.
*   Interpolated strings can be used both for regular string type arguments, as well as for arguments of type FormattableString.

When passing an interpolated string as a regular string type argument, the interpolated string is first formatted, and the resulting string is then passed as the argument value. This is pretty much how we expect interpolated strings to work, as it is equivalent to using string.Format(...). In fact, if you open a compiled app with a disassembler, you'll see that your interpolated string actually compiles down to string.Format(...).

However, as a FormattableString, we can capture the different ingredients that make up the interpolated string, before the formatting takes place. We have access to the string format, as well as the individual argument values that were included.

## Why, though?

I'll agree that this probably isn't something you need on a daily basis. But it has a couple of places where it can be very useful. For instance, I use this technique in my [Blazer library](https://github.com/b-w/Blazer) as a way of constructing SQL queries. This allows users of the library to write queries in a very natural way:

```csharp
connection.Query($"SELECT * FROM Foo WHERE Bar = {arg}")
```

In this case, Blazer will intercept this interpolated string, and will use it to construct a parameterized SQL command. This combines the best of both worlds: you get to include your arguments directly in the query text for clarity, while still having them end up as SQL paramters (a best practice).

## Overloads

Things get a little more tricky when we're dealing with method overloads. Consider the following program, which combines the ProcessString methods from the two previous examples:

```csharp
static void Main(string[] args)
{
    var name = "World";
    ProcessString($"Hello, {name}! The year is {DateTime.Now.Year}.");
    ProcessString("I am a humble, regular string.");
}

static void ProcessString(string arg)
{
    Console.WriteLine("Received the following string:");
    Console.WriteLine(arg);
}

static void ProcessString(FormattableString arg)
{
    Console.WriteLine("Received the following FormattableString:");
    Console.WriteLine(arg.Format);
    foreach (var item in arg.GetArguments())
    {
        Console.WriteLine(item);
    }
}
```

Strangly, this program produces the following output:

```
    Received the following string:
        Hello, World! The year is 2017.
    Received the following string:
        I am a humble, regular string.
```

The overload with the FormattableString argument doesn't get called at all, despite the fact that we're using an interpolated string in the first call. That's not what we expect. It is, however, what the Roslyn team [decided on](https://github.com/dotnet/roslyn/issues/46). In the case of overloads, interpolated strings prefer string over FormattableString.

So how do we fix this? Eliminating the ProcessString(string) overload isn't an option, because that will prevent us from passing regular string arguments, as there is no conversion available from string to FormattableString.

However, we can "guide" the compiler to go where we want it to, by taking advantage of the arcane art of method overload resolution ordering. To do so, we must first introduce a new type:

```csharp
public sealed class NonFormattableString
{
    public NonFormattableString(string arg)
    {
        Value = arg;
    }

    public string Value { get; }

    public static implicit operator NonFormattableString(string arg)
    {
        return new NonFormattableString(arg);
    }

    public static implicit operator NonFormattableString(FormattableString arg)
    {
        throw new InvalidOperationException();
    }
}
```

We then replace the ProcessString(string) overload with one accepting a NonFormattableString, as follows:

```csharp
static void Main(string[] args)
{
    var name = "World";
    ProcessString($"Hello, {name}! The year is {DateTime.Now.Year}.");
    ProcessString("I am a humble, regular string.");
}

static void ProcessString(NonFormattableString arg)
{
    Console.WriteLine("Received the following string:");
    Console.WriteLine(arg.Value);
}

static void ProcessString(FormattableString arg)
{
    Console.WriteLine("Received the following FormattableString:");
    Console.WriteLine(arg.Format);
    foreach (var item in arg.GetArguments())
    {
        Console.WriteLine(item);
    }
}
```

This time, we get the following output:

```
    Received the following FormattableString:
            Hello, {0}! The year is {1}.
            World (System.String)
            2017 (System.Int32)
    Received the following string:
            I am a humble, regular string.
```

Great!

But... how does it work?

Looking at the NonFormattableString class, we see that we define two implicit conversions: one from string, and one from FormattableString. The use of the implicit string conversion should be obvious: it's what allows us to use plain old strings in the case where a NonFormattableString is expected.

The implicit FormattableString conversion is a bit mysterious, especially considering it simply throws an InvalidOperationException, suggesting that it should never be called. So why is it there? Well, if we _don't_ have this conversion in place, then the call to ProcessString with an interpolated string becomes ambiguous between the NonFormattableString and FormattableString overloads. The reason for this comes back to the decision by the Roslyn team to have interpolated strings prefer string over FormattableString. Let's say the conversion to string has precedence 1, and the conversion to FormattableString has precedence 2\. Then the implicit conversion to NonFormattableString also has precedence 2, because it has to go interpolated string -> string -> NonFormattableString. This conflicts with the conversion to FormattableString, which is now at the same level of precedence, thereby making the call ambiguous. Introducing the implicit conversion FormattableString -> NonFormattableString introduces a shorter path for the interpolated string to be converted to a NonFormattableString, which the compiler will prefer over first converting to a string and then to a NonFormattableString. Finally, coming back to the ProcessString method, the choice the compiler has now becomes one between an implicit conversion FormattableString -> NonFormattableString, or calling the FormattableString overload directly. In such a case, the direct overload takes precedence.

You can obviously avoid this mess by just naming your methods differently, but often times an overload will just feel cleaner. In the case of Blazer, I easily prefer `Query("...")` and `Query($"...")` over `Query("...")` and `QueryFormatted($"...")` or something like that. Both methods do the same, in which case an overload makes the most sense.
