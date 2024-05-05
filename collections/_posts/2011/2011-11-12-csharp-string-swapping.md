---
layout: post
title: "String swapping"
categories: blog
---

It's a very basic question: given two strings _a_ and _b_, how would you swap their values?

```csharp
static void Main(string[] args)
{
    string a = "Some random string.";
    string b = "Another one.";

    // begin code

    // end code

    Console.WriteLine(a);
    Console.WriteLine(b);
    Console.ReadKey();
}
```

Fairly trivial, right? Still, there are a number of ways to do this.

The most basic one is the one everyone and their dog initially comes up with:

```csharp
static void Main(string[] args)
{
    string a = "Some random string.";
    string b = "Another one.";

    // begin code

    string c = a;
    a = b;
    b = c;

    // end code

    Console.WriteLine(a);
    Console.WriteLine(b);
    Console.ReadKey();
}
```

But is this the most efficient way to do it? You might not like that extra string variable you had to declare there, so you might think, I can do this with less:

```csharp
static void Main(string[] args)
{
    string a = "Some random string.";
    string b = "Another one.";

    // begin code

    int i = a.Length;
    a = a + b;
    b = a.Substring(0, i);
    a = a.Substring(i);

    // end code

    Console.WriteLine(a);
    Console.WriteLine(b);
    Console.ReadKey();
}
```

Or with _even_ less:

```csharp
static void Main(string[] args)
{
    string a = "Some random string.";
    string b = "Another one.";

    // begin code

    a = a + b;
    b = a.Substring(0, a.Length - b.Length);
    a = a.Substring(b.Length);

    // end code

    Console.WriteLine(a);
    Console.WriteLine(b);
    Console.ReadKey();
}
```

But which one is more efficient? Well, I don't know enough about the internals of the .NET framework to give an exact answer to this, but I wouldn't be surprised if it's the first one. The second example "only" declares an extra integer value, which on average would take less memory space than a string. The third example doesn't declare any extra variables at all. But they do both modify _a_ to hold both _a_ and _b_. Let's just count, shall we?

The first example at least requires space _O(2a + 2b)_.

The second example requires _O(i + 2a + 2b)_.

The third example again requires _O(2a + 2b)_.

Between the first and the third examples, the first one creates a new string variable, but because of the immutability of strings in .NET, so does the third when it modifies _a_! Next, the third example has some substring operations while the first simply reassigns string values. So, is the "smart" solution the better one? Probably not in this case...
