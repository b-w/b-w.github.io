---
layout: post
title: "TIS-100, final part"
categories: blog
---

It's time to bite the bullet and grind our way through the final puzzles of TIS-100.

## Segment 60099: signal window filter

This is the only puzzle in the game where the histograms tell me I have a genuinely poor performing solution (in terms of cycles). I have taken the straightforward approach here, which in this case is apparently naive. I have no interest in improving it, though, as this final batch of puzzles is already pushing the boundary of what I consider "fun".

![](/assets/img/blog/TIS-100/60099.png)

Anyway, on to the solution. The 4 values before the most recent one (the "window") live on the top stack memory. The node between the stacks bounces the values back and forth, passing them to the accumulating node and discarding the oldest one on the way back up. The most recent value gets passed to the accumulating node directly by node 2, which also takes care of adding it to the top stack memory once the other values have been bounced back there. The accumulating node simply adds the values that are given to it, sending the accumulated value down after 3 and 5 additions.

## Segment 61212: signal divider

A compact solution, though a little heavy on the cycles. The division takes place in just two nodes, with node 1 doing the heavy lifting. Node 2 is used as memory, to keep sending the current value of B to node 1 until it is instructed to fetch the next B. Node 1 performs division by counting how many times it can subtract B from A before running into the negatives. This is the value for Q. Once this is found, it adds B to this negative value once, in order to get R. After that the values are simply passed down.

![](/assets/img/blog/TIS-100/61212.png)

## Segment 62711: sequence indexer

Let's see, what haven't we done with sequences yet...? That's right! Indexing!

This puzzle was interesting. One of the few ones I actually spent some time optimizing, because I felt it was reasonably possible to do so.

![](/assets/img/blog/TIS-100/62711.png)

Node 1 reads the sequence to the top stack memory node, and passes the length down to node 3\. Node 3 will act as a memory node, passing the sequence length to node 4 whenever needed.

Node 4 is where the interesting stuff happens. We read the sequence length from node 3 and subtract the requested index value. This tells us how many sequence values to read from the top stack in order to find the requested value. We read however many values from the top stack into the bottom one. The first value will always be the terminating zero. After this, the requested value is the last value placed into the bottom stack. We then read the bottom stack back into the top stack, outputting the first value to node 5, and stopping when we read the terminating zero.

## Segment 63534: sequence sorter

THIS PUZZLE. THIS GODDAMN PUZZLE.

SON OF A BITCH.

I've tried. Believe me, I have tried. But I have not been able to find a working solution for this puzzle, and after spending a few hours going absolutely nowhere, I've given up.

This puzzle reminds me of the harder puzzles of SHENZHEN I/O, near the end of the main storyline and in the entire bonus campaign. Eventually, the difficulty in these games seems to ramp up beyond the point where the puzzles are still fun for me to do. I'll usually have multiple ideas on how to solve them, but none of them will fit in the limited space the game gives you. At this point, the game stops being a fun challenge and starts to feel unfair and difficult for the sake of being difficult. As I'm grinding away, getting more frustrated every minute, I'm reminded of a quote by Reggie Fils-Aimé, president of Nintendo of America:

> If it's not fun, why bother?

I _could_ spend several frustrating hours on these puzzles in order to solve them, for the sake of completion, but why would I? I could also spend that time on things that are actually fun. I've long since given up on SHENZHEN I/O's bonus campaign, as I have now also given up on this particular puzzle.

There. Rant over.

We are moving on.

## Segment 70601: stored image decoder

After the previous puzzle, of which we shall no longer speak, this one felt a bit underwhelming. Maybe it's because I find these image-type puzzles relatively easy, or maybe it's because this one is not supposed to be hard. I don't know. Regardless, here's my solution:

![](/assets/img/blog/TIS-100/70601.png)

Node 1 reads the length L and the color value C, and simply outputs C, L times. After this it reads the next L and C and repeats.

Node 2 keeps track of which X and Y we are on. Rather than counting up from zero for X, we count down from 30, because it's just easier to check for zero than it is to check for 30\. Node 3 then simply takes 30 - UP to get X. After reaching X = 30 (zero for node 2), we increment Y by 1 to move on to the next line.

After that it's just a matter of outputting the values. Not much more to it, I'm afraid. After watching the creepy ending animation (seriously, what?) we have finally repaired the TIS-100 and we are **done**.

## ILLEGAL_EAGLE

There's a hidden puzzle in this game, and solving it will get you the "ILLEGAL_EAGLE" achievement. Finding the puzzle is easy: when in the segment list screen, simply press F2 to view the "anti-tamper certification" popup, and then click on the eagle.

The puzzle itself contains garbled instructions, so you'll have to study the expected output values to guess what it'll do. It's actually pretty simple: OUT.R is IN divided by 25, and OUT.E is a [run-length encoding](https://en.wikipedia.org/wiki/Run-length_encoding) of OUT.R.

Here's my solution for this one:

![](/assets/img/blog/TIS-100/ILLEGAL_EAGLE.png)

OUT.E is definitely the most tricky part here. Starting at the beginning, node 1 computes OUT.R using a simple division-by-subtraction loop. Then there are two nodes (3 and 5) dedicated to computing OUT.E. Node 3 is responsible for detecting the start of a new sequence, and instructing node 5 accordingly. Node 5 keeps the length of the current sequence, which it increments by 1 every time it is instructed to do so by node 3\. Once node 3 detects a the start of a new sequence, it instructs node 5 to output its current values: the length of the current sequence and the sequence value.
