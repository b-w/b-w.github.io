---
layout: post
title: "C# features: async and await"
categories: blog
---

My recent [upgrade to Visual Studio 2012]({% post_url 2013/2013-03-07-visual-studio-2012 %}) means I now also have the new features in .NET version 4.5 to play with. The one I was most exited about was the `async`/`await` pair of keywords introduced to C#. These two keywords aim to offer a relatively simple way of doing asynchronous programming, without all the headaches this sort of thing normally creates. Today I'll go over the syntax and semantics of these two new keywords.

## async

We first look at the `async` keyword. This keyword is used to mark a method as asynchronous. Consider an example method which returns a number:

```csharp
public int GetNumber()
{
    // method body
}
```

If we want to make this method asynchronous, we change its signature to the following:

```csharp
public async Task<int> GetNumber()
{
    // method body
}
```

There's two things to note. First, we've added the `async` keyword to the method signature. But we've also changed the return type, by wrapping it in a `Task` generic. However, we don't need to change what we return in the method's body; this would still be just an integer.

## await

The `async` keyword is useless without its counterpart: the `await` keyword. In fact: any method marked `async` should have at least one instance of the `await` keyword somewhere in its body. If it doesn't, the method will just execute sequentially, without any asynchronicity. The `await` keyword can only be used inside a method marked `async`. We expand our example method:

```csharp
public async Task<int> GetNumber()
{
    Task<int> xTask = SlowMethod();
    int y = FastMethod();
    int x = await xTask;

    return x + y;
}
```

Now, when `GetNumber` is called, the asynchronous method `SlowMethod` is called first, then `FastMethod` is called, and upon reaching the `await` keyword the `GetNumber` method returns control to its caller while it waits for `SlowMethod` to finish. The idea behind this all is of course that several methods can perform time-consuming operations asynchronously in separate threads, while their callers can keep doing other work and only need to wait for the slower methods to finish once their results are actually needed.

## The chicken and the egg

The thing that confused me the most about the `async`/`await` keywords is that there seems to be a sort of chicken-and-the-egg problem going on. First, to mark a method `async`, this method needs to contain at least one `await` call, which means the method needs to call at least one other method which is also `async`. Similarly, to call an `async` method you need to use the `await` keyword, which you can only do from within an `async` method. This seems to suggest that whenever you mark some method `async`, you'll be forced to spread this asynchronicity up and down this method's call stack, until your entire program is asynchronous. Right?

Well, fortunately not. On both ends it is possible to end the cycle. Using an asynchronous method from a regular method is done as follows:

```csharp
public int RegularMethod()
{
    Task<int> task = GetNumber();

    DoOtherThings();

    task.Wait();
    return task.Result;
}
```

Here, the time-consuming `GetNumber` method still runs asynchronously while `RegularMethod` is able to do other things. On the other end of the call stack, here's how you can call regular methods from an asynchronous method:

```csharp
public async Task<int> GetNumber()
{
    int y = FastMethod();
    int x = await Task.Run(() => RegularSlowMethod());

    return x + y;
}
```

Here, the `await` keyword works just the same, returning control to `GetNumber`'s caller while `RegularSlowMethod` executes asynchronously. In fact any `Task` you can come up with will work with `await` in this way.
