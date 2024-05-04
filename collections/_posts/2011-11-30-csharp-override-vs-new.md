---
layout: post
title: "Polymorphic C#: override vs. new"
categories: blog
---

An important part of polymorphism in C# is the `virtual` keyword, which allows you to mark methods, properties, or events as being overridable in derived classes. I came across this nice puzzle which helps explain the difference between the `new` and the `override` keywords:

```csharp
public class A
{
    public virtual string GetName()
    {
        return "A";
    }
}

public class B : A
{
    public override string GetName()
    {
        return "B";
    }
}

public class C : B
{
    public new string GetName()
    {
        return "C";
    }
}

void Main()
{
    A instance = new C();
    Console.WriteLine(instance.GetName());
}
```

So, what is written to the console when this program is run? Think about it for a second before reading on for the answer.

To find the answer, let's see what happens as we step through the inheritance tree. First, the base class A has a method GetName, which is marked virtual and is therefor overridable. When B inherits from A, it does exactly that using the override keyword. This means that for instances of class B (or of any class below B in the inheritance tree), GetName from class A is essentially overwritten by GetName from class B.

Next, class C inherits from B and also declares a GetName method, only this time using the new keyword. The new keyword means that class C also happens to have a GetName method, but this method is not related to the method in class B in any way. It just happens to share the same name. So, the GetName from class B is not overridden, but hidden instead. For instances of class C (or of any class below C in the inheritance tree), GetName from class C is used only if the instance is explicitly known as being of type C (or as being of an inherited type). If not, the method hiding is not in effect and the first known GetName of a class up the inheritance tree will be used instead.

Now, when creating a new instance of class C, the GetName from A is overwritten by GetName from B, and the GetName from C is used only if we know the instance to be of type C. But, because at the time of calling GetName we only know the instance to be of type A, in the end it's the overwritten GetName from B that is called.
