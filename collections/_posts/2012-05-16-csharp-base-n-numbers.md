---
layout: post
title: "A class for working with base-n numbers in C#"
categories: blog
---

When talking about numbers, the **base** (also called the **radix**) represents the number of unique symbols that a particular number system uses. Everyone knows of at least one base: base-10, which is used by the decimal system. Other well known bases are base-2 (binary) and base-16 (hexadecimal).

Usually, the symbols used in these systems are 0 to 9, and then A to Z if anything more is required. Of course, the symbols themselves mean nothing and are just arbitrarily selected. Because I thought it would be fun, I've created a small C# class which can deal with numbers in any arbitrary base and with any arbitrary symbol set. It allows you to do things like:

```csharp
var base4 = new BaseNumber(4, 'A', 'B', 'C', 'D');

base4.Base10Value = 3;		// base4 = D
base4++;					// base4 = BA
base4++;					// base4 = BB
base4 /= 2;					// base4 = C

base4.BaseValue = "CA"		// base4 = 8
```

As you can see, you instantiate the class by giving it a base and a set of symbols. You can then access its value both in regular base-10 or as a representation in the base the class uses. The latter is done using strings. Arithmetical operations are also supported.

So how does it work internally? Well, the internal field keeping the actual value is a `BigInteger`, an integer of arbitrary size found in `System.Numerics`. The `BaseNumber` class knows what base it is in and which symbol set it uses. It then combines these properties with a pair of relatively straightforward conversion functions that allow for parsing base-10 numbers to base-n and vice versa:

```csharp
private string ConvertToString(BigInteger value)
{
    var result = new StringBuilder();

    do
    {
        result.Insert(0, _symbols[(int)(value % _numberBase)]);
        value /= _numberBase;
    } while (value > 0);

    return result.ToString();
}

private BigInteger ConvertFromString(string value)
{
    BigInteger result = 0;

    for (int i = 0; i < value.Length; i++)
    {
        var symbol = _symbols.IndexOf(value[i]);
        if (symbol == -1)
            throw new ArgumentException("Invalid symbol in string value.", "value");
        result += ((BigInteger)symbol) * BigInteger.Pow(_numberBase, value.Length - (i + 1));
    }

    return result;
}
```

These functions are simply an extension of the observation that for a given number _XYZ_, digit _Z_ at position _0_ represents value _Z * base<sup>0</sup> = Z_, _Y_ represents value _Y * base<sup>1</sup>_, and _X_ represents value _X * base<sup>2</sup>_. The symbol list itself is an array, and the value of any symbol is simply its position in the array.

Next, to make the class remotely user-friendly, I've overloaded a whole bunch of operators. For example, for comparing two `BaseNumber` values:

```csharp
public static bool operator >(BaseNumber left, BaseNumber right)
{
    if (left == null || right == null)
        return false;
    return left.Base10Value > right.Base10Value;
}
```

...or adding two of them together:

```csharp
public static BaseNumber operator +(BaseNumber left, BaseNumber right)
{
    left.Base10Value += right.Base10Value;
    return left;
}
```

...which provided an interesting choice to make: which base to take for the result? I've decided to make all operators where this question arises left-associative, which in this case means the base of the left operand is used for the result. I've also included operators in combination with the `BigInteger` class, which thanks to the implicit conversion from _int_ and _long_ types means the `BaseNumber` class can be used fairly painlessly in combination with those number types as well.

Here's another example of the `BaseNumber` class in action:

```csharp
var base3 = new BaseNumber(3, '_', '.', '+');
for (int i = 0; i < 11; i++)
{
    Console.WriteLine(base3);
    base3++;
}
Console.WriteLine();
base3.BaseValue = "++.";
Console.WriteLine(base3.Base10Value);
```

And here is the output this program produces:

    _
    .
    +
    ._
    ..
    .+
    +_
    +.
    ++
    .__
    ._.

    25

Not the most intuitive number system on the planet. Oh well. Enjoy this class by checking out the source code below.

```csharp
using System;
using System.Collections.Generic;
using System.Numerics;
using System.Text;

namespace Com.BartWolff.Numerics
{
    /// <summary>
    /// Represents a number in an arbitrary base using an arbitrary symbol set as representation.
    /// </summary>
    public sealed class BaseNumber : IEquatable<BaseNumber>, IComparable<BaseNumber>
    {
        private static readonly List<char> _base36symbols = new List<char>()
            {
                '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
                'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H', 'I', 'J',
                'K', 'L', 'M', 'N', 'O', 'P', 'Q', 'R', 'S', 'T',
                'U', 'V', 'W', 'X', 'Y', 'Z'
            };
        private readonly List<char> _symbols = new List<char>();
        private readonly int _numberBase;
        private BigInteger _numberValue;

        /// <summary>
        /// Initializes a new instance of the <see cref="BaseNumber"/> class, using the default [0-9A-Z] symbol set.
        /// </summary>
        /// <param name="numberBase">The number base.</param>
        /// <exception cref="ArgumentException">If the base is larger than 36.</exception>
        public BaseNumber(int numberBase)
            : this(numberBase, _base36symbols.ToArray())
        {
        }

        /// <summary>
        /// Initializes a new instance of the <see cref="BaseNumber"/> class, using the given symbol set.
        /// </summary>
        /// <param name="numberBase">The number base.</param>
        /// <param name="numberSymbols">The symbol set to use.</param>
        /// <exception cref="ArgumentException">Thrown if the number base is smaller than 2, or there are insufficient
        /// symbols provided to deal with the given base.</exception>
        public BaseNumber(int numberBase, params char[] numberSymbols)
        {
            if (numberBase < 2)
                throw new ArgumentException("Base must be at least 2.", "numberBase");
            if (numberBase > numberSymbols.Length)
                throw new ArgumentException("Number of symbols provided must be at least equal to the given number base. If no symbols are provided in the constructor, the default 36 symbols [0-9A-Z] are used.", "numberSymbols");

            _numberBase = numberBase;
            _symbols.AddRange(numberSymbols);
        }

        /// <summary>
        /// Gets the number base used.
        /// </summary>
        public int Base
        {
            get { return _numberBase; }
        }

        /// <summary>
        /// Gets the symbol set used to represent the numbers.
        /// </summary>
        public IEnumerable<char> Symbols
        {
            get
            {
                foreach (var symbol in _symbols)
                {
                    yield return symbol;
                }
            }
        }

        /// <summary>
        /// Gets or sets the current base-10 value.
        /// </summary>
        /// <value>
        /// The base-10 value.
        /// </value>
        /// <exception cref="ArgumentOutOfRangeException">Thrown if the given value is negative.</exception>
        public BigInteger Base10Value
        {
            get { return _numberValue; }
            set
            {
                if (value < 0)
                    throw new ArgumentOutOfRangeException("_numberValue", "Negative values are not supported.");
                _numberValue = value;
            }
        }

        /// <summary>
        /// Gets or sets the value in the current number base. The value is presented as a string.
        /// </summary>
        /// <value>
        /// The value represented in the current number base.
        /// </value>
        /// <exception cref="ArgumentException">Thrown if the conversion from the given string fails.</exception>
        public string BaseValue
        {
            get { return this.ToString(); }
            set
            {
                Base10Value = ConvertFromString(value);
            }
        }

        #region Conversion functions

        /// <summary>
        /// Converts the given base-10 value to a string representation in the current number base.
        /// </summary>
        /// <param name="value">The base-10 value.</param>
        /// <returns>A string representation i the current number base of the given base-10 value.</returns>
        private string ConvertToString(BigInteger value)
        {
            var result = new StringBuilder();

            do
            {
                result.Insert(0, _symbols[(int)(value % _numberBase)]);
                value /= _numberBase;
            } while (value > 0);

            return result.ToString();
        }

        /// <summary>
        /// Converts a given string representation in the current number base to a base-10 value.
        /// </summary>
        /// <param name="value">The string holding the value in the current number base.</param>
        /// <returns>The base-10 value of the given string representation in the current number base.</returns>
        private BigInteger ConvertFromString(string value)
        {
            BigInteger result = 0;

            for (int i = 0; i < value.Length; i++)
            {
                var symbol = _symbols.IndexOf(value[i]);
                if (symbol == -1)
                    throw new ArgumentException("Invalid symbol found in string value.", "value");
                result += ((BigInteger)symbol) * BigInteger.Pow(_numberBase, value.Length - (i + 1));
            }

            return result;
        }

        #endregion

        #region Comparison functions

        /// <summary>
        /// Determines whether the specified base number is equal to this instance.
        /// </summary>
        /// <param name="other">The other base number to compare with this instance.</param>
        /// <returns><c>true</c> if the specified base number is equal to this instance; otherwise, <c>false</c>.</returns>
        public bool Equals(BaseNumber other)
        {
            if ((object)other == null)
                return false;
            return this.Base10Value.Equals(other.Base10Value);
        }

        /// <summary>
        /// Determines whether the specified <see cref="System.Object"/> is equal to this instance.
        /// </summary>
        /// <param name="obj">The <see cref="System.Object"/> to compare with this instance.</param>
        /// <returns>
        ///   <c>true</c> if the specified <see cref="System.Object"/> is equal to this instance; otherwise, <c>false</c>.
        /// </returns>
        public override bool Equals(object obj)
        {
            if (obj == null)
                return false;
            if (obj is BaseNumber)
                return this.Equals(obj as BaseNumber);
            return false;
        }

        /// <summary>
        /// Compares the given base number to this instance.
        /// </summary>
        /// <param name="other">The base number to compare with this instance.</param>
        /// <returns><c>1</c> if the given base number is <c>null</c>or if this instance preceeds the given base number
        /// in the sort order, <c>0</c> if the two base numbers occur in the same position, and <c>-1</c> if this
        /// instance preceeds the given base number in the sort order.</returns>
        public int CompareTo(BaseNumber other)
        {
            if ((object)other == null)
                return 1;
            return this.Base10Value.CompareTo(other.Base10Value);
        }

        #endregion

        #region Operators

        public static bool operator ==(BaseNumber left, BaseNumber right)
        {
            if (object.ReferenceEquals(left, right))
                return true;
            if (left == null || right == null)
                return false;
            return left.Equals(right);
        }

        public static bool operator ==(BaseNumber left, BigInteger right)
        {
            if (left == null || right == null)
                return false;
            return left.Base10Value.Equals(right);
        }

        public static bool operator ==(BigInteger left, BaseNumber right)
        {
            return right == left;
        }

        public static bool operator !=(BaseNumber left, BaseNumber right)
        {
            return !(left == right);
        }

        public static bool operator !=(BaseNumber left, BigInteger right)
        {
            return !(left == right);
        }

        public static bool operator !=(BigInteger left, BaseNumber right)
        {
            return !(left == right);
        }

        public static bool operator >(BaseNumber left, BaseNumber right)
        {
            if (left == null || right == null)
                return false;
            return left.Base10Value > right.Base10Value;
        }

        public static bool operator >(BaseNumber left, BigInteger right)
        {
            if (left == null || right == null)
                return false;
            return left.Base10Value > right;
        }

        public static bool operator >(BigInteger left, BaseNumber right)
        {
            if (left == null || right == null)
                return false;
            return left > right.Base10Value;
        }

        public static bool operator <(BaseNumber left, BaseNumber right)
        {
            if (left == null || right == null)
                return false;
            return left.Base10Value < right.Base10Value;
        }

        public static bool operator <(BaseNumber left, BigInteger right)
        {
            if (left == null || right == null)
                return false;
            return left.Base10Value < right;
        }

        public static bool operator <(BigInteger left, BaseNumber right)
        {
            if (left == null || right == null)
                return false;
            return left < right.Base10Value;
        }

        public static bool operator >=(BaseNumber left, BaseNumber right)
        {
            if (left == null || right == null)
                return false;
            return left.Base10Value >= right.Base10Value;
        }

        public static bool operator >=(BaseNumber left, BigInteger right)
        {
            if (left == null || right == null)
                return false;
            return left.Base10Value >= right;
        }

        public static bool operator >=(BigInteger left, BaseNumber right)
        {
            if (left == null || right == null)
                return false;
            return left >= right.Base10Value;
        }

        public static bool operator <=(BaseNumber left, BaseNumber right)
        {
            if (left == null || right == null)
                return false;
            return left.Base10Value <= right.Base10Value;
        }

        public static bool operator <=(BaseNumber left, BigInteger right)
        {
            if (left == null || right == null)
                return false;
            return left.Base10Value <= right;
        }

        public static bool operator <=(BigInteger left, BaseNumber right)
        {
            if (left == null || right == null)
                return false;
            return left <= right.Base10Value;
        }

        public static BaseNumber operator +(BaseNumber left, BaseNumber right)
        {
            left.Base10Value += right.Base10Value;
            return left;
        }

        public static BaseNumber operator +(BaseNumber left, BigInteger right)
        {
            left.Base10Value += right;
            return left;
        }

        public static BaseNumber operator +(BigInteger left, BaseNumber right)
        {
            right.Base10Value += left;
            return right;
        }

        public static BaseNumber operator -(BaseNumber left, BaseNumber right)
        {
            left.Base10Value -= right.Base10Value;
            return left;
        }

        public static BaseNumber operator -(BaseNumber left, BigInteger right)
        {
            left.Base10Value -= right;
            return left;
        }

        public static BaseNumber operator -(BigInteger left, BaseNumber right)
        {
            right.Base10Value -= left;
            return right;
        }

        public static BaseNumber operator *(BaseNumber left, BaseNumber right)
        {
            left.Base10Value *= right.Base10Value;
            return left;
        }

        public static BaseNumber operator *(BaseNumber left, BigInteger right)
        {
            left.Base10Value *= right;
            return left;
        }

        public static BaseNumber operator *(BigInteger left, BaseNumber right)
        {
            right.Base10Value *= left;
            return right;
        }

        public static BaseNumber operator /(BaseNumber left, BaseNumber right)
        {
            left.Base10Value /= right.Base10Value;
            return left;
        }

        public static BaseNumber operator /(BaseNumber left, BigInteger right)
        {
            left.Base10Value /= right;
            return left;
        }

        public static BaseNumber operator /(BigInteger left, BaseNumber right)
        {
            right.Base10Value /= left;
            return right;
        }

        public static BaseNumber operator %(BaseNumber left, BaseNumber right)
        {
            left.Base10Value %= right.Base10Value;
            return left;
        }

        public static BaseNumber operator %(BaseNumber left, BigInteger right)
        {
            left.Base10Value %= right;
            return left;
        }

        public static BaseNumber operator %(BigInteger left, BaseNumber right)
        {
            right.Base10Value %= left;
            return right;
        }

        public static BaseNumber operator ++(BaseNumber operand)
        {
            operand.Base10Value++;
            return operand;
        }

        public static BaseNumber operator --(BaseNumber operand)
        {
            operand.Base10Value--;
            return operand;
        }

        public static BaseNumber operator ~(BaseNumber operand)
        {
            operand.Base10Value = ~operand.Base10Value;
            return operand;
        }

        public static BaseNumber operator &(BaseNumber left, BaseNumber right)
        {
            left.Base10Value &= right.Base10Value;
            return left;
        }

        public static BaseNumber operator &(BaseNumber left, BigInteger right)
        {
            left.Base10Value &= right;
            return left;
        }

        public static BaseNumber operator &(BigInteger left, BaseNumber right)
        {
            right.Base10Value &= left;
            return right;
        }

        public static BaseNumber operator |(BaseNumber left, BaseNumber right)
        {
            left.Base10Value |= right.Base10Value;
            return left;
        }

        public static BaseNumber operator |(BaseNumber left, BigInteger right)
        {
            left.Base10Value |= right;
            return left;
        }

        public static BaseNumber operator |(BigInteger left, BaseNumber right)
        {
            right.Base10Value |= left;
            return right;
        }

        public static BaseNumber operator ^(BaseNumber left, BaseNumber right)
        {
            left.Base10Value ^= right.Base10Value;
            return left;
        }

        public static BaseNumber operator ^(BaseNumber left, BigInteger right)
        {
            left.Base10Value ^= right;
            return left;
        }

        public static BaseNumber operator ^(BigInteger left, BaseNumber right)
        {
            right.Base10Value ^= left;
            return right;
        }

        public static BaseNumber operator <<(BaseNumber left, int right)
        {
            left.Base10Value <<= right;
            return left;
        }

        public static BaseNumber operator >>(BaseNumber left, int right)
        {
            left.Base10Value >>= right;
            return left;
        }

        public static explicit operator BigInteger(BaseNumber value)
        {
            return value.Base10Value;
        }

        #endregion

        /// <summary>
        /// Returns a <see cref="System.String"/> that represents this instance.
        /// </summary>
        /// <returns>
        /// A <see cref="System.String"/> that represents this instance.
        /// </returns>
        public override string ToString()
        {
            return ConvertToString(Base10Value);
        }

        /// <summary>
        /// Returns a hash code for this instance.
        /// </summary>
        /// <returns>
        /// A hash code for this instance, suitable for use in hashing algorithms and data structures like a hash table. 
        /// </returns>
        public override int GetHashCode()
        {
            return this.Base10Value.GetHashCode();
        }
    }
}
```

Happy coding!
