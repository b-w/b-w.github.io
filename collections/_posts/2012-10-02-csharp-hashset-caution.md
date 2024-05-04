---
layout: post
title: "A word of caution when using HashSet"
categories: blog
---

The [HashSet](http://msdn.microsoft.com/en-us/library/bb359438.aspx) is a "class [which] provides high-performance set operations. A set is a collection that contains no duplicate elements, and whose elements are in no particular order." Common set operations such as additions, deletions, or containment checks can be done in _O(1)_ time. It's not difficult to come up with use cases where this may come in handy.

Still, here's some... unexpected behavior I've encountered. I call it "unexpected behavior" because it's not really a bug, even though it might feel like one when you're debugging this problem and you have no idea where it's coming from.

Consider the case where you have some class that you want to insert into lists and sets. So you implement `IEquatable` and you also decide to override `GetHashCode()`. Standard stuff.

```csharp
using System;

namespace Sandbox
{
    public class Something : IEquatable<Something>
    {
        private int m_value1;
        private int m_value2;

        public Something(int value1, int value2)
        {
            m_value1 = value1;
            m_value2 = value2;
        }

        public int Value1
        {
            get { return m_value1; }
            set { m_value1 = value; }
        }

        public int Value2
        {
            get { return m_value2; }
            set { m_value2 = value; }
        }

        public bool Equals(Something other)
        {
            return (m_value1 == other.m_value1 && m_value2 == other.m_value2);
        }

        public override bool Equals(object obj)
        {
            if (obj == null)
                return false;
            if (obj is Something)
                return this.Equals(obj as Something);
            return false;
        }

        public override int GetHashCode()
        {
            return m_value1 ^ m_value2;
        }
    }
}
```

At first, everything looks like it's working fine. Until you start getting some strange bugs that you can't directly find the origin of. The reason is, as it turns out, that HashSet doesn't play well when its contents start changing. Consider the following code snippet:

```csharp
var set = new HashSet<Something>();
var list = new List<Something>();
var item = new Something(12, 57);

set.Add(item);
list.Add(item);

Console.WriteLine("Item in set: {0}", set.Contains(item));
Console.WriteLine("Item in list: {0}", list.Contains(item));

Console.WriteLine();
Console.WriteLine("Changing item...");
item.Value1 = 86;
Console.WriteLine();

Console.WriteLine("Item in set: {0}", set.Contains(item));
Console.WriteLine("Item in list: {0}", list.Contains(item));
```

Here, we compare the behavior of a HashSet against that of a List. And in this case, you'd expect them to be the same. But they're not. Here's the output this code gives:

    Item in set: True
    Item in list: True

    Changing item...

    Item in set: False
    Item in list: True

Strange, right? Well, not if you understand how the HashSet works internally. See, to check whether some item _x_ exists in the set, the HashSet takes two steps. First, it computes the hash code of _x_ using `GetHashCode()`, which then points it to a bucket. It then checks if one of the items in this bucket has the same hash code as _x_. If one is found, the second step is done, where equality is determined through the `Equals()` function.

Here's the relevant code snippet (obtained from the [.NET Reference Source](http://referencesource.microsoft.com/netframework.aspx)):

```csharp
public bool Contains(T item) {
    if (m_buckets != null) { 
        int hashCode = InternalGetHashCode(item);
        // see note at "HashSet" level describing why "- 1" appears in for loop
        for (int i = m_buckets[hashCode % m_buckets.Length] - 1; i >= 0; i = m_slots[i].next) {
            if (m_slots[i].hashCode == hashCode && m_comparer.Equals(m_slots[i].value, item)) { 
                return true;
            } 
        } 
    }
    // either m_buckets is null or wasn't found 
    return false;
}
```

By changing our item's `Value1`, we have changed the way the hash is computed. Subsequently, when another containment check is done, the HashSet ends up looking in the wrong bucket, doesn't find the item, and thinks it's not in there. Of course, it's wrong. If we do this:

```csharp
Console.WriteLine("Item actually in set: {0}", set.ElementAt(0).Equals(item));
```

...we get this:

    Item actually in set: True

And, oddly enough, we can even insert the original item again. This:

```csharp
var addAgain = set.Add(item);
Console.WriteLine("Added again to set: {0}", addAgain);
```

...produces:

    Added again to set: True

Which means we have managed to break the definition of a set as given at the start of this post, by introducing duplicate elements:

```csharp
var itemsEqual = ReferenceEquals(set.ElementAt(0), set.ElementAt(1));
Console.WriteLine("Set failure: {0}", itemsEqual);
```

...produces:

    Set failure: True

So why does List not suffer from this same problem when doing the containment check? Simple: it doesn't use hashing, but relies solely on the `Equals()` function. So does that mean you shouldn't use HashSet? Well, no. The problem in this case is not with HashSet; it's how we're using it. In this scenario, where the hash code of objects might change while they're still in the HashSet, it's not the right collection type to use. Too bad there's no non-hash set type in .NET yet...
