---
layout: post
title: "FastArg: quick and extendable command-line argument handling in C#"
categories: blog
---

Having an application handle or parse command-line arguments is a very common task. Today I present a generic solution which greatly simplifies doing so, and which can be reused in any project. _FastArg_ allows you to tie switches to delegates that handle them, and relieves you of the task of parsing the arguments yourself. It also has a few neat features which improve usability.

Let's look at the most basic example of how to use it:

```csharp
var testArgs = new string[] { "-print", "Hello World" };

var argHandler = new FastArg("-", "[", "]");

argHandler.Add("print", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine(x);
}));

argHandler.Handle(testArgs);
```

It's a "Hello World" example, of course. I use the `testArgs` variable to simulate the arguments that would normally be passed to your application. We first create a new `FastArg` object, which serves as our argument parser and handler. The `Add` function on our argument handler registers a new delegate under the key "print". The delegate itself takes a string and prints that to the console. We'll discus what the other parameters do in a minute. When we run this code, "Hello World" gets printed to the console, as expected. But wait, there's more...

## Partial switch inferring

Rather than forcing you to use the full switch name, or be restricted to a limited set of aliases, FastArg requires only the minimal amount of information required to disambiguate between switches. For example, the following sets of arguments all neatly print "Hello World":

    > FastArg.exe -print "Hello World"
    Hello World
    > FastArg.exe -pri "Hello World"
    Hello World
    > FastArg.exe -p "Hello World"
    Hello World

Easy, right? So what about the following:

```csharp
var testArgs = new string[] { "-p", "Hello World" };

var argHandler = new FastArg("-", "[", "]");

argHandler.Add("print", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine(x);
}));

argHandler.Add("process", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine("Processing: " + x);
}));

argHandler.Handle(testArgs);
```

When we run this code, FastArgs throws an exception saying the switch "p" is ambiguous. It's obvious why: there's two different handlers which start with the letter "p" and which both take one argument. In this case, FastArg can't decide which handler to use. We'll have to provide at least the first three letters of the switch in order for things to disambiguate.

Things get a little more interesting when we try the following:

```csharp
var testArgs = new string[] { "-p", "Hello World" };

var argHandler = new FastArg("-", "[", "]");

argHandler.Add("print", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine(x);
}));

argHandler.Add("process", 1, 1, false, new Action(() => {
    Console.WriteLine("Now processing...");
}));

argHandler.Handle(testArgs);
```

Here all we've done is change the "process" delegate to an action which does not take an argument. Perhaps surprisingly, this code will run just fine and print "Hello World" to the console. The reason behind this lies with FastArg's greedy parsing algorithm. Formally speaking, when given a switch _s_ followed by _n_ arguments, FastArg will seek out the handler with a name starting with _s_ and with _k <= n_ arguments such that _k_ is maximal. In the above case there is only one such handler, and thus no ambiguity. This is a key part of how FastArg works, and important to understand if you start using it.

As a second example of this, consider the following:

```csharp
var testArgs = new string[] { "-p", "Hello World", "Greedy!" };

var argHandler = new FastArg("-", "[", "]");

argHandler.Add("print", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine(x);
}));

argHandler.Add("process", 1, 1, false, new Action<string, string>((string x, string y) => {
    Console.WriteLine("Now processing " + x + " and " + y);
}));

argHandler.Handle(testArgs);
```

This code will end up printing "Now processing Hello World and Greedy!".

## Anonymous arguments

Let's get back to our basic example:

```csharp
var testArgs = new string[] { "Hello World" };

var argHandler = new FastArg("-", "[", "]");

argHandler.Add("print", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine(x);
}));

argHandler.Handle(testArgs);
```

As a twist, we have removed the switch from the argument list altogether, and now only provide the "Hello World" parameter. This code runs fine, and is a basic example of how FastArg deals with anonymous arguments. Since there is only one handler, taking one argument, it is trivial to infer what to do in this case.

Again we can think of more interesting examples:

```csharp
var testArgs = new string[] { "Hello World", "How are you doing?" };

var argHandler = new FastArg("-", "[", "]");

argHandler.Add("print", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine(x);
}));

argHandler.Add("process", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine("Processing: " + x);
}));

argHandler.Handle(testArgs);
```

Here, an exception is thrown as FastArg has detected an ambiguity. It doesn't know whether the "Hello World" argument belongs to the "print" handler or the "process" handler. Luckily, we can resolve this ambiguity quite easily:

```csharp
var testArgs = new string[] { "Hello World", "How are you doing?" };

var argHandler = new FastArg("-", "[", "]");

argHandler.Add("print", 2, 1, false, new Action<string>((string x) => {
    Console.WriteLine(x);
}));

argHandler.Add("process", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine("Processing: " + x);
}));

argHandler.Handle(testArgs);
```

Can you spot the change? Hint: it's on line 5\. The second parameter of the `Add` function will let us specify a handler's argument ordering, i.e. the order in which the handlers should deal with anonymous arguments. In the case of an ambiguity as in the previous example, the argument ordering will be used to resolve it (lower values indicting higher priority). Hence, the code above prints the following:

    How are you doing?
    Processing: Hello World

The first argument, "Hello World", was passed to the "process" handler, and the second was sent to the "print" handler. This is another key part of understanding how FastArg works.

The current limit of anonymous arguments is that they only work for handlers which take a single argument. For instance, the following will throw an exception as FastArg is unsure what to do here:

```csharp
var testArgs = new string[] { "Hello World", "Greedy?" };

var argHandler = new FastArg("-", "[", "]");

argHandler.Add("process", 1, 1, false, new Action<string, string>((string x, string y) => {
    Console.WriteLine("Now processing " + x + " and " + y);
}));

argHandler.Handle(testArgs);
```

The only way of currently getting around this limitation is by explicitly grouping arguments together. Which brings us to our next subject...

## Argument grouping

FastArgs supports argument grouping, and lets you define your own string values which indicate the opening and closing of argument groups. In our examples so far, we have been using "[" and "]" for this purpose. Argument grouping forces FastArg to consider the arguments inside the group as a whole, meaning they can only be passed to a handler that takes that exact number of arguments.

Here's how we can use argument grouping to fix our previous anonymous argument difficulties:

```csharp
var testArgs = new string[] { "[", "Hello World", "Greedy?", "]" };

var argHandler = new FastArg("-", "[", "]");

argHandler.Add("process", 1, 1, false, new Action<string, string>((string x, string y) => {
    Console.WriteLine("Now processing " + x + " and " + y);
}));

argHandler.Handle(testArgs);
```

Argument grouping can have a great effect on how FastArg processes your arguments. Consider the following example:

```csharp
var argHandler = new FastArg("-", "[", "]");

argHandler.Add("print", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine(x);
}));

argHandler.Add("process", 1, 1, false, new Action<string, string>((string x, string y) => {
    Console.WriteLine("Now processing " + x + " and " + y);
}));

argHandler.Handle(args);
```

Here's the output with and without argument grouping:

    > FastArg.exe -p "Hello World" "Groups!" "And Such"
    Now processing Hello World and Groups!
    And Such
    > FastArg.exe -p [ "Hello World" ] [ "Groups!" "And Such" ]
    Hello World
    Now processing Groups! and And Such

Grouping also opens the door for...

## Method overloading

FastArg includes support for method overloading, i.e. providing multiple handlers for a single switch where each handler takes a different number of arguments. Consider the following example:

```csharp
var argHandler = new FastArg("-", "[", "]");

argHandler.Add("print", 1, 1, false, new Action<string>((string x) => {
    Console.WriteLine(x);
}));

argHandler.Add("process", 1, 1, false, new Action<string, string>((string x, string y) => {
    Console.WriteLine("Now processing " + x + " and " + y);
}));

argHandler.Add("process", 1, 1, false, new Action<string, string, string>((string x, string y, string z) => {
    Console.WriteLine("Now processing " + x + ", " + y + ", and " + z);
}));

argHandler.Handle(args);
```

Here, the "process" switch has two handlers, one which takes two arguments and one which takes three arguments. Perhaps by now you'll be able to predict how FastArg will behave in this case...

    > FastArg.exe -p "Hello World" "Groups!" "And Such"
    Now processing Hello World, Groups!, and And Such
    > FastArg.exe -p [ "Hello World" "Groups!" ] "And Such"
    Now processing Hello World and Groups!
    And Such

In the first case, FastArg is greedy and thus calls the handler for a switch starting with "p" and taking the largest possible number of arguments, which is our newly defined "process" handler. In the second case, argument grouping forces FastArg to consider "Hello World" and "Groups!" together, which causes them to be passed to our old "process" hander which takes two arguments. Then, the third argument is anonymous, and gets sent to the "print" hander because that's the only handler left taking a single argument.

## Other niceties

A few miscellaneous features of FastArg we have not yet discussed include:

*   **Execution ordering**. The third parameter in the `Add` function used for registering handlers specifies a handler's execution order. After first fully parsing the arguments given to it, FastArg invokes all the required handlers. The order in which it does this might matter in your application, and can be controlled by specifying each handler's execution ordering. A lower value indicates a higher priority.
*   **Required switches**. The fourth parameter in the `Add` function is a boolean which indicates whether this particular switch or handler is required or not. If any required handler goes unused, FastArg throws an exception.
*   **Variable switch prefix**. FastArg will let you decide what prefix to use to identify switches. Common prefixes include "-", "/", or "--". Any valid string is allowed.

## Wrapping up...

FastArg is a powerful tool for quickly parsing command-line arguments in any application. It includes a number of advanced features, is easy to use, and effortlessly automates an otherwise tedious and repetitive process. In a next post, we'll discuss the code behind FastArg.
