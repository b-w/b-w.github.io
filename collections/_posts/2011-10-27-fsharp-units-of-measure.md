---
layout: post
title: "F# Units of Measure"
categories: blog
---

I stumbled upon [Units of Measure](http://msdn.microsoft.com/en-us/library/dd233243.aspx) while reading a book on F#. They allow you to add some metadata to a value (can be of any type, but is aimed at integer and floating-point values) with some additional information describing the unit this value denotes, such as centimeters, kilograms, etc.

```fsharp
> [<Measure>] type cm;;

[<Measure>]
type cm

> [<Measure>] type inch;;

[<Measure>]
type inch

>
```

After declaring a unit of measure, they can be applied to any value:

```fsharp
> let x = 25.0<cm>;;

val x : float<cm> = 25.0

>
```

Value x is now a float of measure unit cm.

So what use is this? Well, it's mostly a link that F# has to the physical world, where it's sometimes imperative that calculations are done within a certain system of measurements. A famous example of how important this can be is the [Mars Climate Orbiter](http://en.wikipedia.org/wiki/Mars_Climate_Orbiter#Communications_loss), which was a $327.6 million NASA mission that was lost because parts of the internal systems software used the Metric system for calculations while another part used Imperial measurements.

With units of measure you can enforce that specific measures are used:

```fsharp
> let complicatedFunction(n : float<cm>) = n / 2.0;;

val complicatedFunction : float<cm> -> float<cm>

> let floatValue = 14.0;;

val floatValue : float = 14.0

> let inchValue = 6.0<inch>;;

val inchValue : float<inch> = 6.0

> complicatedFunction(inchValue);;

    complicatedFunction(inchValue);;
    --------------------^^^^^^^^^

stdin(6,21): error FS0001: Type mismatch. Expecting a
    float<cm>
but given a
    float<inch>
The unit of measure 'cm' does not match the unit of measure 'inch'

> complicatedFunction(floatValue);;

    complicatedFunction(floatValue);;
    --------------------^^^^^^^^^^

stdin(7,21): error FS0001: This expression was expected to have type
    float<cm>
but here has type
    float

>
```

The function above accepts only values that have been explicitly defined as unit of measure <cm>.

Of course, seeing as we are working with floating point values it is also possible to do arithmetic as usual, though it can get a bit confusing at this point. For example, consider the following:

```fsharp
> 2.0<cm> * 3.0<cm>;;

val it : float<cm ^ 2> = 6.0

>
```

What is this? Multiplying two cm values results in a cm<sup>2</sup> unit of measure? Well, yes. When performing arithmetic with unit of measure values, two things actually happen. First, the operation is performed on the actual values, and secondly the same operation is also performed on the units of measure of each value. In the example above, we get 2.0 * 3.0 = 6.0 as a value and cm * cm = cm<sup>2</sup> as a unit of measure.

The same holds for division:

```fsharp
> 7.0<cm> / 2.0<inch>;;

val it : float<cm/inch> = 3.5

>
```

Values without a unit of measure implicitly have unit of measure <1>:

```fsharp
> 2.0<cm> * 3.0;;

val it : float<cm> = 6.0

>
```

Alright, then what about addition and subtraction? Well, this works slightly differently. These operations have no effect on the units of measure, but unlike multiplication and division they must be applied on values of the same unit of measure.

```fsharp
> 3.0<cm> + 2.0;;

    3.0<cm> + 2.0;;
    ----------^^^

stdin(8,11): error FS0001: The type 'float' does not match the type 'float<cm>'

> 3.0<cm> + 2.0<cm>;;

val it : float<cm> = 5.0

>
```

Are you confused yet? It's actually not that complicated:

1.  Multiplication and division: units of measure do not need to be equal, apply operation also to units of measure
2.  Addition and subtraction: units of measure need to be equal, do not apply operation to units of measure

Some more examples:

```fsharp
> 7.0<cm> / 2.54<cm/inch>;;

val it : float<inch> = 2.755905512

> 5.0<cm> * 3.0<inch>;;

val it : float<cm inch> = 15.0

> 5.0<cm> * 3.0<inch> / 3.0<cm>;;

val it : float<inch> = 5.0

>
```

So, is this useful? Probably. Will I ever use it? Nope! But it's still fun to figure out.
