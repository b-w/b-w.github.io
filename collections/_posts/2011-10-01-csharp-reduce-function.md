---
layout: post
title: "A generic reduce function in C#"
categories: blog
---

A reduce function is a function that traverses over some input data (usually a set of objects) and produces a single output value. An example use would be finding the average age in a set of people. Here I'll show how to build a generic reduce function in C#. This is not necessarily good code, but it is a nice practice with generics and delegates.

We start by defining the following delegate:

```csharp
delegate U Op<T1, T2, U>(T1 arg1, T2 arg2);
```

This delegate is extremely generic. It can be used to define all functions that take two arguments of any type and return some value. We'll be using it in the reduce function:

```csharp
static U Reduce<T, U>(U start, IEnumerable<T> data, Op<T, U, U> accum)
{
    U res = start;
    foreach (T item in data)
    {
        res = accum(item, res);
    }
    return res;
}
```

So what does the reduce function actually do? At a first glance, not much. But this _is_ the completed function. The idea is that the reduce operation itself, _how_ it works, can be specified independent of the actual work that is being done during the traversal over the data and the subsequent reduce steps.

The reduce function takes a set of objects of some type T and produces a value of type U as an accumulated result. For it to work, it needs an initial value to start from (of type U) and an accumulation function which will do the actual work. In this case, the accumulation function can be any function that accepts some object of type T, some value of type U, and returns the result of accumulating T to U (the result again being of type U).

To see how this works, consider the following test data:

```csharp
class MyClass
{
    public string MyStringValue;
    public int MyIntValue;
}

MyClass data1 = new MyClass() { MyIntValue = 1, MyStringValue = "Abc" };
MyClass data2 = new MyClass() { MyIntValue = 3, MyStringValue = "Ijk" };
MyClass data3 = new MyClass() { MyIntValue = 5, MyStringValue = "Xyz" };

IEnumerable<MyClass> data = new MyClass[] { data1, data2, data3 };
```

A typical use of our reduce function on this data would be finding the sum of all values in `MyIntValue`. To do this, we will need to define an accumulation function to handle this for us:

```csharp
Op<MyClass, int, int> addition =
    delegate(MyClass arg1, int arg2)
    {
        return arg1.MyIntValue + arg2;
    };
```

This function takes some object of type `MyClass` and some integer value. It then adds the integer value and the `MyIntValue` field and returns the result. Here's how we apply this accumulation function to the dataset:

```csharp
int value1 = Reduce<MyClass, int>(0, data, addition);
Console.WriteLine(value1);
```

Here we reduce over type `MyClass` and expect a result of type int. We pass along the initial value of 0, the data, and the accumulation function we have defined earlier. As a result, "9" will be outputted to the console. Another possibility using strings:

```csharp
Op<MyClass, string, string> concat = delegate(MyClass arg1, string arg2)
    {
        return String.Concat(arg2, arg1.MyStringValue);
    };

string value2 = Reduce<MyClass, string>(String.Empty, data, concat);
Console.WriteLine(value2);
```

...which produces output "AbcIjkXyz".

Of course, these examples are fairly trivial, but it's easy to imagine more complicated accumulation functions, the result of which might just as easily be an enum or class rather than a value type.

The important thing to take away from this is the power of delegates in C#, which we in this case combined with generics. It's a bit of an extreme example and the resulting code is not that easy to understand, but it's nice to see how we were able to write the framework for our reduce function before any actual accumulator functions even existed. We were able to define our accumulator functions later on and then pass them to the reducer. The reducer itself doesn't know -- not even at compile time -- which accumulation function will be passed to it, yet it still nicely performs our additions and concatenations.
