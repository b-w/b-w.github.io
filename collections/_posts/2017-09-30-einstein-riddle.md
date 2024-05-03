---
layout: post
title: "Solving Einstein's Riddle the lazy way"
categories: blog
---

There's a logic puzzle called **Einstein's Riddle** (originally known as the [Zebra Puzzle](https://en.wikipedia.org/wiki/Zebra_Puzzle)) which is supposedly pretty well-known, as far as logic puzzles are concerned. It's also claimed to be quite difficult, with only 2% of people capable of solving it, although that sounds pretty sensationalist and I can't find a source for that figure.

Anyway, here's the puzzle as it is commonly found today:

> Setup:
> 
> *   There are 5 houses, each painted in a different color
> *   The houses sit next to each other on the same road
> *   Each house has 1 person living in it
> *   Each person has a different nationality, drinks a different beverage, plays on a different gaming system, and owns a different phone
> 
> Hints:
> 
> 1.  The Brit lives in a Blue house
> 2.  The Frenchman owns a Galaxy S5
> 3.  The German drinks Coca Cola
> 4.  The Red house is next to, and on the left of, the White house
> 5.  The owner of the Red house drinks Coffee
> 6.  The person who plays on a PC owns a Nokia Lumia
> 7.  The owner of the Green house plays on an Xbox One
> 8.  The person living in the Center house drinks Water
> 9.  The American lives in the First house
> 10.  The person who plays on a PS4 lives next to the person who owns an iPhone 6
> 11.  The person who owns a Galaxy S6 lives next to the person who plays on the Xbox One
> 12.  The person who plays on the PS3 drinks Beer
> 13.  The Swede plays on the Xbox 360
> 14.  The American lives next to the Yellow house
> 15.  The person who plays on the PS4 has a neighbor who drinks Pepsi
> 
> The question: **who owns the iPhone 5**?

It's been "updated" for the 21st century, by swapping out the Zebra for an iPhone 5.

The puzzle contains no tricks, wordplay, or other such nonsense. It's purely based on logical reasoning. You know, the kind of logic that computers are so darned good at these days. Which is exactly why I will not be solving this puzzle myself. After all, I can just delegate this stuff to my computer, so why bother? All I'll have to do is encode it into a format the computer can understand.

_**Spoiler warning: this post contains the solution to the riddle.**_

We'll be encoding the puzzle as an [SMT problem](https://en.wikipedia.org/wiki/Satisfiability_modulo_theories), and we'll let [Yices](http://yices.csl.sri.com/) solve it for us.

We'll start by setting up our SMT file and declaring the variables we'll be using.

```
(benchmark einstein.smt
:logic QF_UFLIA
:extrafuns (
                    ; 1 = Blue
                    ; 2 = Red
                    ; 3 = Green
                    ; 4 = Yellow
                    ; 5 = White
    (C1 Int) (C2 Int) (C3 Int) (C4 Int) (C5 Int)

                    ; 1 = American
                    ; 2 = German
                    ; 3 = Brit
                    ; 4 = Swedish
                    ; 5 = French
    (N1 Int) (N2 Int) (N3 Int) (N4 Int) (N5 Int)

                    ; 1 = Pepsi
                    ; 2 = Coca Cola
                    ; 3 = Water
                    ; 4 = Coffee
                    ; 5 = Beer
    (B1 Int) (B2 Int) (B3 Int) (B4 Int) (B5 Int)

                    ; 1 = 360
                    ; 2 = XBone
                    ; 3 = PS3
                    ; 4 = PS4
                    ; 5 = PC
    (G1 Int) (G2 Int) (G3 Int) (G4 Int) (G5 Int)

                    ; 1 = S5
                    ; 2 = S6
                    ; 3 = iPhone 5
                    ; 4 = iPhone 6
                    ; 5 = Lumia
    (P1 Int) (P2 Int) (P3 Int) (P4 Int) (P5 Int)
)
        ; formula goes here
)
```

We'll use integer variables for house **c**olor, **n**ationality, **b**everage, **g**aming system, and **p**hone. The number in the variable name denotes the house position. It's basically a 5x5 matrix:

<table border="1" cellpadding="1" cellspacing="1">
	<tbody>
		<tr>
			<td>&nbsp;</td>
			<td>C</td>
			<td><strong>N</strong></td>
			<td>B</td>
			<td>G</td>
			<td>P</td>
		</tr>
		<tr>
			<td>1</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
		</tr>
		<tr>
			<td>2</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
		</tr>
		<tr>
			<td><strong>3</strong></td>
			<td>&nbsp;</td>
			<td>x</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
		</tr>
		<tr>
			<td>4</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
		</tr>
		<tr>
			<td>5</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
			<td>&nbsp;</td>
		</tr>
	</tbody>
</table>

Here _x_ is encoded in variable N3 and its value denotes the nationality of the person in the 3rd house. For example, N3 = 1 would mean the American lives in the 3rd house.

Now that we have our variables we can start entering our formula, i.e. the predicate that needs to be satisfied. At its root it's a giant conjunction of sub-predicates. The first set of sub-predicates involves constraining the values of our variables. Specifically, we need to encode that:

1.  Each variable must be between 1 and 5, inclusive.
2.  There can be no duplicate values among a group of variables (e.g. if the first house is Blue, then none of the other houses can be Blue).

We end up with the following predicates:

```
(benchmark einstein.smt
:logic QF_UFLIA
:extrafuns (
        ; variables go here
)
:formula(and
        ; uniqueness and range
    (and
            (> C1 0) (< C1 6)
            (> C2 0) (< C2 6)
            (> C3 0) (< C3 6)
            (> C4 0) (< C4 6)
            (> C5 0) (< C5 6)
    )
    (and
            (not (= C1 C2))
            (not (= C1 C3))
            (not (= C1 C4))
            (not (= C1 C5))
            (not (= C2 C3))
            (not (= C2 C4))
            (not (= C2 C5))
            (not (= C3 C4))
            (not (= C3 C5))
            (not (= C4 C5))
    )

        ; uniqueness and range for the remaining variables goes here

        ; hints go here
)
)
```

This encodes these requirements for the C variable group.

These predicates are then repeated exactly like this for each of the 4 other variable groups, but I've omitted that code here for the sake of brevity. It's just some copy+paste work with the C replaced by N, B, G, or P.

Next, we can finally get into the meat of the puzzle: encoding each of the 15 hints. We'll just go through them one at the time.

_Hint #1: the Brit lives in a Blue house_. This becomes:

```
(and
    (iff (= N1 3) (= C1 1))
    (iff (= N2 3) (= C2 1))
    (iff (= N3 3) (= C3 1))
    (iff (= N4 3) (= C4 1))
    (iff (= N5 3) (= C5 1))
)
```

This basically says: if the Brit happens to live in house #1, then the color of house #1 must be Blue. And, reversely, if the color of house #1 happens to be Blue, then the Brit must live there. Same for the remaining 4 houses. This hint doesn't say anything about _where_ the Brit lives; just that _wherever_ he lives, the color of that house must be Blue.

_Hint #2: the Frenchman owns a Galaxy S5_. This becomes:

```
(and
    (iff (= N1 5) (= P1 1))
    (iff (= N2 5) (= P2 1))
    (iff (= N3 5) (= P3 1))
    (iff (= N4 5) (= P4 1))
    (iff (= N5 5) (= P5 1))
)
```

Exactly the same kind of reasoning as hint #1.

_Hint #3: the German drinks Coca Cola_. This becomes:

```
(and
    (iff (= N1 2) (= B1 2))
    (iff (= N2 2) (= B2 2))
    (iff (= N3 2) (= B3 2))
    (iff (= N4 2) (= B4 2))
    (iff (= N5 2) (= B5 2))
)
```

Same reasoning again.

_Hint #4: the Red house is next to, and on the left of, the White house_. This becomes:

```
(and
    (not (= C1 5))
    (not (= C5 2))
    (iff (= C5 5) (= C4 2))
    (iff (= C4 5) (= C3 2))
    (iff (= C3 5) (= C2 2))
    (iff (= C2 5) (= C1 2))
)
```

There's two things going on here. First, the fact that the Red house is located left of the White house implies that the color of the leftmost house cannot be White, and the color of the rightmost house cannot be Red. The first two lines of the predicate capture that. The remaining lines are again similar to the previous hints: _if and only if_ house #5 is White, _then_ house #4 must be Red. _If and only if_ house #4 is White, _then_ house #3 must be Red. Etc.

_Hint #5: the owner of the Red house drinks Coffee_. This becomes:

```
(and
    (iff (= C1 2) (= B1 4))
    (iff (= C2 2) (= B2 4))
    (iff (= C3 2) (= B3 4))
    (iff (= C4 2) (= B4 4))
    (iff (= C5 2) (= B5 4))
)
```

_Hint #6: the person who plays on a PC owns a Nokia Lumia_. This becomes:

```
(and
    (iff (= G1 5) (= P1 5))
    (iff (= G2 5) (= P2 5))
    (iff (= G3 5) (= P3 5))
    (iff (= G4 5) (= P4 5))
    (iff (= G5 5) (= P5 5))
)
```

_Hint #7: the owner of the Green house plays on an Xbox One_. This becomes:

```
(and
    (iff (= C1 3) (= G1 2))
    (iff (= C2 3) (= G2 2))
    (iff (= C3 3) (= G3 2))
    (iff (= C4 3) (= G4 2))
    (iff (= C5 3) (= G5 2))
)
```

_Hint #8: the person living in the Center house drinks Water_. This becomes:

```
(= B3 3)
```

This particular hint outright gives us the value for one of the variables.

_Hint #9: the American lives in the First house_. This becomes:

```
(= N1 1)
```

Same here. Hints #8 and #9 are the only two hints like this.

_Hint #10: the person who plays on a PS4 lives next to the person who owns an iPhone 6_. This becomes:

```
(and
    (implies (= G1 4) (= P2 4))
    (implies (= G2 4) (or (= P1 4) (= P3 4)))
    (implies (= G3 4) (or (= P2 4) (= P4 4)))
    (implies (= G4 4) (or (= P3 4) (= P5 4)))
    (implies (= G5 4) (= P4 4))
)

(and
    (implies (= P1 4) (= G2 4))
    (implies (= P2 4) (or (= G1 4) (= G3 4)))
    (implies (= P3 4) (or (= G2 4) (= G4 4)))
    (implies (= P4 4) (or (= G3 4) (= G5 4)))
    (implies (= P5 4) (= G4 4))
)
```

This one's a little more interesting. The way we encode this is by first having the position of the PS4 player imply the (possible) position(s) of the iPhone 6 owner. Of course, the reverse is also true: the position of the iPhone 6 owner implies the (possible) position(s) of the PS4 player.

The remaining hints are all stuff we've seen before.

_Hint #11: the person who owns a Galaxy S6 lives next to the person who plays on the Xbox One_. This becomes:

```
(and
    (implies (= P1 2) (= G2 2))
    (implies (= P2 2) (or (= G1 2) (= G3 2)))
    (implies (= P3 2) (or (= G2 2) (= G4 2)))
    (implies (= P4 2) (or (= G3 2) (= G5 2)))
    (implies (= P5 2) (= G4 2))
)

(and
    (implies (= G1 2) (= P2 2))
    (implies (= G2 2) (or (= P1 2) (= P3 2)))
    (implies (= G3 2) (or (= P2 2) (= P4 2)))
    (implies (= G4 2) (or (= P3 2) (= P5 2)))
    (implies (= G5 2) (= P4 2))
)
```

_Hint #12: the person who plays on the PS3 drinks Beer_. This becomes:

```
(and
    (iff (= G1 3) (= B1 5))
    (iff (= G2 3) (= B2 5))
    (iff (= G3 3) (= B3 5))
    (iff (= G4 3) (= B4 5))
    (iff (= G5 3) (= B5 5))
)
```

_Hint #13: the Swede plays on the Xbox 360_. This becomes:

```
(and
    (iff (= N1 4) (= G1 1))
    (iff (= N2 4) (= G2 1))
    (iff (= N3 4) (= G3 1))
    (iff (= N4 4) (= G4 1))
    (iff (= N5 4) (= G5 1))
)
```

_Hint #14: the American lives next to the Yellow house_. This becomes:

```
(and
    (implies (= N1 1) (= C2 4))
    (implies (= N2 1) (or (= C1 4) (= C3 4)))
    (implies (= N3 1) (or (= C2 4) (= C4 4)))
    (implies (= N4 1) (or (= C3 4) (= C5 4)))
    (implies (= N5 1) (= C4 4))
)

(and
    (implies (= C1 4) (= N2 1))
    (implies (= C2 4) (or (= N1 1) (= N3 1)))
    (implies (= C3 4) (or (= N2 1) (= N4 1)))
    (implies (= C4 4) (or (= N3 1) (= N5 1)))
    (implies (= C5 4) (= N4 1))
)
```

_Hint #15: the person who plays on the PS4 has a neighbor who drinks Pepsi_. This becomes:

```
(and
    (implies (= G1 4) (= B2 1))
    (implies (= G2 4) (or (= B1 1) (= B3 1)))
    (implies (= G3 4) (or (= B2 1) (= B4 1)))
    (implies (= G4 4) (or (= B3 1) (= B5 1)))
    (implies (= G5 4) (= B4 1))
)
(and
    (implies (= B1 1) (= G2 4))
    (implies (= B2 1) (or (= G1 4) (= G3 4)))
    (implies (= B3 1) (or (= G2 4) (= G4 4)))
    (implies (= B4 1) (or (= G3 4) (= G5 4)))
    (implies (= B5 1) (= G4 4))
)
```

And that's it! We've encoded the entire puzzle as an SMT problem. It might seem like that was a lot of work, but it's not. There's only 4 different kinds of hints, and they're all pretty easy to translate into a logical predicate. It's actually mostly just copy+paste work, and making sure to use the correct variables and values.

All that remains now is to let Yices loose on our problem. After running:

```
yices-smt.exe -m .\einstein.smt
```

...we instantly get the following result:

```
sat

(= P2 2)
(= C1 3)
(= C4 2)
(= B4 4)
(= N1 1)
(= C5 5)
(= G4 1)
(= G5 3)
(= P4 3)
(= B3 3)
(= B5 5)
(= P1 4)
(= N3 3)
(= P3 5)
(= C3 1)
(= N5 5)
(= B2 2)
(= G1 2)
(= G3 5)
(= G2 4)
(= C2 4)
(= P5 1)
(= N2 2)
(= N4 4)
(= B1 1)
```

This tells us two things:

1.  Our problem is satisfiable (i.e. there exists a solution for which our predicate holds).
2.  The values for the variables for one possible solution (which, in this case, is the only solution).

Now it's just a matter of finding the owner of the iPhone 5, which comes down to first spotting the P variable with a value of 3\. In our case, we find P4 = 3\. We then find N4 to get the nationality of the iPhone 5 owner. Since we have N4 = 4, it is indeed the Swede who owns the iPhone 5.

In fact, since we have values for all of our variables, we can actually fill out the entire matrix:

<table border="1" cellpadding="1" cellspacing="1">
	<tbody>
		<tr>
			<td>&nbsp;</td>
			<td><strong>House color</strong></td>
			<td><strong>Nationality</strong></td>
			<td><strong>Beverage</strong></td>
			<td><strong>Gaming system</strong></td>
			<td><strong>Phone</strong></td>
		</tr>
		<tr>
			<td><strong>House #1</strong></td>
			<td>Green</td>
			<td>American</td>
			<td>Pepsi</td>
			<td>Xbox One</td>
			<td>iPhone 6</td>
		</tr>
		<tr>
			<td><strong>House #2</strong></td>
			<td>Yellow</td>
			<td>German</td>
			<td>Coca Cola</td>
			<td>PS4</td>
			<td>Galaxy S6</td>
		</tr>
		<tr>
			<td><strong>House #3</strong></td>
			<td>Blue</td>
			<td>British</td>
			<td>Water</td>
			<td>PC</td>
			<td>Nokia Lumia</td>
		</tr>
		<tr>
			<td><strong>House #4</strong></td>
			<td>Red</td>
			<td>Swedish</td>
			<td>Coffee</td>
			<td>Xbox 360</td>
			<td>iPhone 5</td>
		</tr>
		<tr>
			<td><strong>House #5</strong></td>
			<td>White</td>
			<td>French</td>
			<td>Beer</td>
			<td>PS3</td>
			<td>Galaxy S5</td>
		</tr>
	</tbody>
</table>

Pretty neat. All this, and we didn't even have to think about it. After all, why use your own brain when you can just let the computer do the work for you?
