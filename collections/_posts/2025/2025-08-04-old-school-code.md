---
layout: post
title: "Old School Code"
categories: blog
---

While digging around some old hard drives the other day, I stumbled across an ancient software artifact. It's a small program for the TI-83 graphing calculator that I wrote over 20 years ago when I was in highschool. Some real _old school code_, if you will. Let's see what it does.

## The TI-83

The Texas Instruments TI-83 is a graphing calculator that is ubiquitous in highschool math classes all over the world. In fact, it's so ubiquitous that it effectively has a monopoly, which is why its price (as well as that of its successor, the TI-84) has barely dropped despite the hardware being almost 30 years old at this point.

I owned the TI-83 Plus.

![](/assets/img/blog/2025/08/Ti83plus.jpg)

I thought this device was awesome. Despite its simplicity and limited capabilities, it's still technically a computer, and I was especially interested in the ability to write my own programs for it. Although they were little more than simple scripts, it's one of the first "computers" that I used to "program" on. I wrote most of my code directly on the device itself, using its shitty little keyboard and its 96x64 monochrome display. Who needs IDEs, anyway?

## WORTEL.8XP

Most of my experiments with the TI-83's basic programming language were just me messing around. However, I did create a few things that were actually useful. One of those programs was called "WORTEL" (literally _root_, as in the square root of a number).

Here's the original source code listing, in TI-BASIC:

```
ClrHome
Disp "  sqrt(PROGRAMMA)","   WRITTEN BY","   BART WOLFF",""
Input "sqrt(",A
If A<0:Then
ClrHome
Disp "",""
Output(1,1,"sqrt(
Output(1,3,A
Output(1,16,"=
Output(2,1,"ONMOGELIJK"
Stop
End
If fPart(A)!=0
Then
ClrHome
Disp "","",""
Output(1,1,"sqrt(
Output(1,3,A)
Output(1,16,"=
Output(2,1,"ONMOGELIJK"
Output(2,16,"=
Output(3,1,sqrt(A
Stop
End
sqrt(A)->B
round(B,0)+1->X
0->Z
While X>0
X²->Y
A/Y->Z
If fPart(Z)=0
Goto Ø
X-1->X
End
Lbl Ø:ClrHome
If fPart(Z)!=0 or X=1
Then
Disp "","",""
Output(1,1,"sqrt(
Output(1,3,A
Output(1,16,"=
Output(2,1,"ONMOGELIJK"
Output(2,16,"=
Output(3,1,sqrt(A
Stop
Else
Disp "",""
If X<1:3->S
If X>=1:2->S
If X>=10:3->S
If X>=100:4->S
If X>=1000:5->S
If X>=10000:6->S
If X>=100000:7->S
If X>=1000000:8->S
If Z=1
Then
Output(1,1,"sqrt(
Output(1,3,A)
Output(1,16,"=
Output(2,1,X)
Else
Disp ""
Output(1,1,"sqrt(
Output(1,3,A)
Output(1,16,"=
Output(2,1,X)
Output(2,S,"sqrt(
Output(2,S+2,Z):Output(2,16,"=
Output(3,1,sqrt(A
End
End
```

If you think that's hard to read, imagine writing this code on a tiny monochrome display that can show 8 lines at a time.

To make it a little easier to understand, here's a Go version that does the same thing:

```go
package main

import (
    "fmt"
    "math"
    "strconv"
    "strings"
)

func main() {
    fmt.Println("  sqrt(PROGRAMMA)")
    fmt.Println("   WRITTEN BY")
    fmt.Println("   BART WOLFF")

    var A float64
    fmt.Print("sqrt(")
    fmt.Scanln(&A)

    if A < 0 {
        fmt.Println("---")
        fmt.Printf("sqrt(%-10s", strconv.FormatFloat(A, 'f', -1, 64))
        fmt.Println("=")
        fmt.Println("ONMOGELIJK")
        return
    }

    _, A_f := math.Modf(A)
    if A_f != 0 {
        fmt.Println("---")
        fmt.Printf("sqrt(%-10s", strconv.FormatFloat(A, 'f', -1, 64))
        fmt.Println("=")
        fmt.Println("ONMOGELIJK     =")
        fmt.Printf("%.7f\n", math.Sqrt(A))
        return
    }

    B := math.Sqrt(A)
    X := math.Round(B)
    var Z float64 = 0

    for X > 0 {
        Y := X * X
        Z = A / Y

        _, Z_f := math.Modf(Z)
        if Z_f == 0 {
            break
        }

        X = X - 1
    }

    fmt.Println("---")

    _, Z_f := math.Modf(Z)
    if Z_f != 0 || X == 1 {
        fmt.Printf("sqrt(%-10s", strconv.FormatFloat(A, 'f', -1, 64))
        fmt.Println("=")
        fmt.Println("ONMOGELIJK     =")
        fmt.Printf("%.7f\n", math.Sqrt(A))
        return
    } else {
        if Z == 1 {
            fmt.Printf("sqrt(%-10s", strconv.FormatFloat(A, 'f', -1, 64))
            fmt.Println("=")
            fmt.Println(X)
        } else {
            fmt.Printf("sqrt(%-10s", strconv.FormatFloat(A, 'f', -1, 64))
            fmt.Println("=")
            X_str := strconv.FormatFloat(X, 'f', -1, 64)
            Z_str := strconv.FormatFloat(Z, 'f', -1, 64)
            fmt.Printf("%ssqrt(%s%s", X_str, Z_str, strings.Repeat(" ", 10-len(X_str)-len(Z_str)))
            fmt.Println("=")
            fmt.Printf("%.7f\n", math.Sqrt(A))
        }
    }
}
```

It's not an exact 1-to-1 translation (mainly due to differences in I/O functions), but all of the logic is identical. Most of the verbosity comes from formatting the output; the logic at the core of the program is actually very simple.

So, what does it do?

Well, as the name implies, it has to do with square roots. Specifically, this is a program that automates simplifying square roots.

It's based on the rule that:

$$ \sqrt{a \cdot b} = \sqrt{a} \cdot \sqrt{b} $$

The idea is, when given some $$ \sqrt{n} $$, you remove the perfect squares from $$ n $$, i.e. rewriting $$ n $$ as $$ k^2 \cdot m $$ such that $$ m $$ no longer contains any perfect squares. Then $$ \sqrt{k^2 \cdot m} $$ can be rewritten as $$ \sqrt{k^2} \cdot \sqrt{m} $$ which simplifies to $$ k \sqrt{m} $$.

The core of the program is just a simple loop that finds the largest value for $$ k $$. Like I said: _very_ simple. But, for someone who was just dipping their toes into the world of programming, this was a nice little homebrew tool with real practical use.

Here's some example outputs from the Go version.

Simplifying $$ \sqrt{25} $$ simply removes the square root altogether:

```
  sqrt(PROGRAMMA)
   WRITTEN BY
   BART WOLFF
sqrt(25
---
sqrt(25        =
5
```

Simplifying $$ \sqrt{26} $$ is impossible:

```
  sqrt(PROGRAMMA)
   WRITTEN BY
   BART WOLFF
sqrt(26
---
sqrt(26        =
ONMOGELIJK     =
5.0990195
```

Simplifying $$ \sqrt{27} $$ by extracting the perfect square $$ 3^2 \cdot 3 $$:

```
  sqrt(PROGRAMMA)
   WRITTEN BY
   BART WOLFF
sqrt(27
---
sqrt(27        =
3sqrt(3        =
5.1961524
```

Simplifying $$ \sqrt{72} $$ by extracting the largest possible square $$ 6^2 \cdot 2 $$ (as opposed to the equally possible $$ 3^2 \cdot 8 $$):

```
  sqrt(PROGRAMMA)
   WRITTEN BY
   BART WOLFF
sqrt(72
---
sqrt(72        =
6sqrt(2        =
8.4852814
```

I was pretty proud of this trivial little program. Simplifying square roots was part of the math curriculum, and my program effectively rendered this small part of my homework (and exams!) moot. I also shared it with my classmates, copying it to each of their TI-83s using the link cable. Good times.
