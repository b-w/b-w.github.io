---
layout: post
title: "Building an array from functions"
categories: blog
---

I've been working with lambda expressions lately for a project, and one of the things I needed to do was use them to build a basic array data structure. Out of sheer boredom, I wondered if I could translate this to C#, because hey, why not? Turns out I can, very easily actually. Here's how it goes:

```csharp
delegate T FArray<T>(int index);
```

My "array" will actually be a function which takes an integer as index and spits out some value which is at that index. The first thing we need to do is declare an empty array.

```csharp
static FArray<T> NewArray<T>()
{
    return delegate(int index) { return default(T); };
}
```

As you can see, the empty array is a function which takes an integer as index, ignores it completely, and just returns the default value of whatever type the array is of (null for reference types, 0 for value types). Splendid. Now we need to put stuff in the array. This can get slightly more complex.

```csharp
static FArray<T> SetValue<T>(FArray<T> array, int atIndex, T value)
{
    return delegate(int index)
    {
        if (index == atIndex)
            return value;
        else
            return array(index);
    };
}
```

Not to mention ugly. The SetValue function eats an array, an integer indicating where to store the new value, and the value itself. It spits out a function which again takes an integer as index and checks whether it's equal to the index where the new value has been stored. If it is, it returns the new value. If not, it applies the index to the array value that was given to it initially.

And that's all. Use it like so:

```csharp
var arr = NewArray<int>();
arr = SetValue<int>(arr, 4, 42);
arr = SetValue<int>(arr, 27, 3);

int fortytwo = arr(4);
```

You don't even need to initialize it to some fixed size, and you can write at any position. In this way it's essentially a key-value store. I used integers for an index but you could use any data type for this purpose, as long as you can compare whether two values of it are equal or not.

Now, as fun as this example might look, it is also hideous beyond words. Because when values are being stored in this array, essentially a giant chain of functions is being constructed, which only gets bigger each time you store a new value. So yeah, you probably shouldn't use this for anything...
