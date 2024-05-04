---
layout: post
title: "A simple sparse set implementation in C#"
categories: blog
---

I'm not working on any projects right now (at least, not any that would have me do some C# programming), so instead I occasionally try to make small exercises to keep myself sharp. Nothing special or big or complicated; just an hour or so a day of applying my skills to a little puzzle or problem, like [this one](http://programmingpraxis.com/2012/03/09/sparse-sets/).

The sparse set is a data structure which represents a set of natural numbers, and sacrifices space for speed. The integers it stores can come from domain _[0..u]_, and the data structure takes _O(2u)_ space to do so, whether it's empty or completely full. Not terribly efficient. It's also fixed size. On the other hand, it can do insertions, deletions, member checks, and emptying of the set all in _O(1)_ time. To do this, it maintains two arrays _D_ and _S_ of size _u_, and the current size of the set _n_. The arrays are linked such that at any time, for a given value _i_ in the set, _D[S[i]] = i_.

Uses for this data structure are scenario's where storage space is not much of an issue, but computational time is. Standard set operations are extremely fast, and this includes set emptying and cardinality checking. The former is an _O(n)_ operation on a HashSet.

On to the implementation. The following constructor initializes a new sparse set:

```csharp
private readonly int _max;
private int _n;
private readonly int[] _d;
private readonly int[] _s;

public SparseSet(int maxValue)
{
    _max = maxValue + 1;
    _n = 0;
    _d = new int[_max];
    _s = new int[_max];
}
```

The size of the set is fixed and cannot be changed. The following code adds a value to the set:

```csharp
public void Add(int value)
{
    if (value >= 0 && value < _max && !Contains(value))
    {
        _d[_n] = value;
        _s[value] = _n;
        _n++;
    }
}
```

The code for the contains check is not important yet. To add a value, we simply place it at the next open slot in _D_, link it to _S_, and increment _n_. Removal of an element is slightly more complicated:

```csharp
public void Remove(int value)
{
    if (Contains(value))
    {
        _d[_s[value]] = _d[_n - 1];
        _s[_d[_n - 1]] = _s[value];
        _n--;
    }
}
```

Here, we first replace the value to be removed in _D_ (looking it up through _S_) with a new value: the last value of _D_. We then fix the link between the two arrays by looking up this new value in _S_ and pointing it to its new location, which is the same as the value that is removed and thus in that position in _S_. We then decrement _n_. Note that we don't "clean up" the arrays; the value of _n_ is used as a cut-off to determine what elements in _D_ are actually part of the set.

Now for the contains check:

```csharp
public bool Contains(int value)
{
    if (value >= _max || value < 0)
        return false;
    else
        return _s[value] < _n && _d[_s[value]] == value;
}
```

Here, we check for the condition mentioned earlier: if an element _i_ is in the set then there is always a valid link between the two arrays such that _D[S[i]] = i_. Secondly, the lookup _S[i]_ must point to a location _D[j]_ such that _j < n_.

We have one more function to implement:

```csharp
public void Clear()
{
    _n = 0;
}
```

Emptying the set is a fairly trivial operation with this data structure: all we need to do is set _n_ to _0_. Even though the arrays _D_ and _S_ are potentially still full of information, no value has _S[i] < 0_ so the set will appear empty. When new values are added, _D_ and _S_ are simply overwritten along the way.

Full source code listing:

```csharp
using System.Collections;
using System.Collections.Generic;

namespace Com.BartWolff.Datastructures
{
    /// <summary>
    /// Represents an unordered sparse set of natural numbers, and provides constant-time operations on it.
    /// </summary>
    public sealed class SparseSet : IEnumerable<int>
    {
        private readonly int _max;      // maximal value the set can contain
                                        // _max = 100; implies a range of [0..99]
        private int _n;                 // current size of the set
        private readonly int[] _d;      // dense array
        private readonly int[] _s;      // sparse array

        /// <summary>
        /// Initializes a new instance of the <see cref="SparseSet"/> class.
        /// </summary>
        /// <param name="maxValue">The maximal value the set can contain.</param>
        public SparseSet(int maxValue)
        {
            _max = maxValue + 1;
            _n = 0;
            _d = new int[_max];
            _s = new int[_max];
        }

        /// <summary>
        /// Adds the given value.
        /// If the value already exists in the set it will be ignored.
        /// </summary>
        /// <param name="value">The value.</param>
        public void Add(int value)
        {
            if (value >= 0 && value < _max && !Contains(value))
            {
                _d[_n] = value;     // insert new value in the dense array...
                _s[value] = _n;     // ...and link it to the sparse array
                _n++;
            }
        }

        /// <summary>
        /// Removes the given value in case it exists.
        /// </summary>
        /// <param name="value">The value.</param>
        public void Remove(int value)
        {
            if (Contains(value))
            {
                _d[_s[value]] = _d[_n - 1];     // put the value at the end of the dense array
                                                // into the slot of the removed value
                _s[_d[_n - 1]] = _s[value];     // put the link to the removed value in the slot
                                                // of the replaced value
                _n--;
            }
        }

        /// <summary>
        /// Determines whether the set contains the given value.
        /// </summary>
        /// <param name="value">The value.</param>
        /// <returns>
        ///   <c>true</c> if the set contains the given value; otherwise, <c>false</c>.
        /// </returns>
        public bool Contains(int value)
        {
            if (value >= _max || value < 0)
                return false;
            else
                return _s[value] < _n && _d[_s[value]] == value;    // value must meet two conditions:
                                                                    // 1\. link value from the sparse array
                                                                    // must point to the current used range
                                                                    // in the dense array
                                                                    // 2\. there must be a valid two-way link
        }

        /// <summary>
        /// Removes all elements from the set.
        /// </summary>
        public void Clear()
        {
            _n = 0;     // simply set n to 0 to clear the set; no re-initialization is required
        }

        /// <summary>
        /// Gets the number of elements in the set.
        /// </summary>
        public int Count
        {
            get { return _n; }
        }

        /// <summary>
        /// Returns an enumerator that iterates through all elements in the set.
        /// </summary>
        /// <returns>
        /// An <see cref="T:System.Collections.IEnumerator"/> object that can be used to iterate through the collection.
        /// </returns>
        public IEnumerator<int> GetEnumerator()
        {
            var i = 0;
            while (i < _n)
            {
                yield return _d[i];
                i++;
            }
        }

        /// <summary>
        /// Returns an enumerator that iterates through all elements in the set.
        /// </summary>
        /// <returns>
        /// An <see cref="T:System.Collections.IEnumerator"/> object that can be used to iterate through the collection.
        /// </returns>
        IEnumerator IEnumerable.GetEnumerator()
        {
            return GetEnumerator();
        }
    }
}
```

Happy coding!
