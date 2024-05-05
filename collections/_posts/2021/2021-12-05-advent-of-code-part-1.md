---
layout: post
title: "Advent of Code 2021, days 1 to 5"
categories: blog
---

I figured I'd write about my coding adventures in the [Advent of Code](https://adventofcode.com/) this year, and see how far I'll make it this time. In case you're not familiar, it's an advent calendar containing a daily coding puzzle, starting from December 1st and continuing until the 25th. The puzzles tend to get more difficult over time, so I might not solve every single day.

I'll also be solving them in Python (version 3.10), because it's a fun language, but I haven't use it in a while. I don't do a lot of coding puzzles, and I'm not very experienced with Python either, so my solutions will probably be neither efficient nor idiomatic, but whatever. The important thing is solving them by any means necessary, because that's what Christmas is all about.

These will just be small logs and notes regarding each puzzle, rather than full explanations or walkthroughs. For the puzzle descriptions, check the [AoC calendar for this year](https://adventofcode.com/2021).

Enough rambling, on with the puzzles.

## Day 1

Unsurprisingly, we start of fairly easily. The core of this problem is moving a sliding window over a list of elements. I've created a simple helper function for this using a queue:

```python
from queue import Queue

def window(seq, n=2):
    w = Queue()
    for x in seq:
        w.put(x)
        if w.qsize() == n:
            yield tuple(w.queue)
            w.get()
```

For part 1, we walk a window of size 2 over the input, and count the times where the first element is smaller than the second, thus indicating an increase:

```python
def solution_part_1():
    count = 0
    with open('input.txt', 'r') as f_input:
        for w in window(map(lambda x: int(x), f_input)):
            if w[0] < w[1]:
                count += 1
    print(count)
```

For part 2, it nicely turned out that my window helper function could be reused, as we now have to essentially compare two windows of length 3, thus creating windows of windows:

```python
def solution_part_2():
    count = 0
    with open('input.txt', 'r') as f_input:
        for w in window(window(map(lambda x: int(x), f_input), 3)):
            if sum(w[0]) < sum(w[1]):
                count += 1
    print(count)
```

Easy enough.

## Day 2

Here we have to read in a bunch of instructions and simulate the movement of a submarine. Because I'm used to object-oriented programming as opposed to code golfing, I naturally created a class to encapsulate the submarine logic:

```python
class SubPart1:
    def __init__(self):
        self.depth = 0
        self.position = 0

    def forward(self, n):
        self.position += n

    def down(self, n):
        self.depth += n

    def up(self, n):
        self.depth -= n
```

By matching the method names to the instructions from the file, I can take advantage of Python's [getattr](https://docs.python.org/3/library/functions.html#getattr) to dynamically call the instruction method without having to perform any mapping:

```python
sub1 = SubPart1()

with open('input.txt', 'r') as f_input:
    for line in f_input:
        command = line.split(sep=' ')
        getattr(sub1, command[0])(int(command[1]))

print(sub1.depth * sub1.position)
```

Easy. For part 2, all I have to do is create a slightly different submarine class:

```python
class SubPart2:
    def __init__(self):
        self.depth = 0
        self.position = 0
        self.aim = 0

    def forward(self, n):
        self.position += n
        self.depth += (self.aim * n)

    def down(self, n):
        self.aim += n

    def up(self, n):
        self.aim -= n
```

We can use this class in exactly the same way, and we're already finished with day 2.

## Day 3

Some weird binary number counting/manipulating here. First, just fetch the input:

```python
with open('input.txt', 'r') as f_input:
    lines = f_input.readlines()

data_len = len(lines)
digits = len(lines[0]) - 1  # ignore newline
```

For part 1, we simply go over each bit from most to least significant, and compute its decimal value (128, 64, 32, ...). We then perform a count of the 1s and 0s in that position. If there are more 1s, we add the decimal value to gamma; otherwise we add it to sigma:

```python
def solution_part_1():
    gamma = 0
    epsilon = 0

    for i in range(digits):
        one_count = sum(map(lambda x: x[i] == '1', lines))
        decimal_value = 2 ** (digits - (i + 1))
        if one_count > (data_len / 2):
            gamma += decimal_value
        else:
            epsilon += decimal_value

    print(gamma * epsilon)
```

Part to gets a little more convoluted. We now have to progressively eliminate numbers based on bit counts. I make two copies of the input set (for the oxygen reading and the CO2 reading), loop over the digit positions, do the count to see whether 1s or 0s should be eliminated, and then update the sets.

```python
def solution_part_2():
    lines_oxy = lines[:]
    lines_co2 = lines[:]

    for i in range(digits):
        if len(lines_oxy) > 1:
            one_count_oxy = sum(map(lambda x: x[i] == '1', lines_oxy))
            if one_count_oxy >= len(lines_oxy) - one_count_oxy:
                lines_oxy = [x for x in lines_oxy if x[i] == '1']
            else:
                lines_oxy = [x for x in lines_oxy if x[i] == '0']

        if len(lines_co2) > 1:
            zero_count_co2 = sum(map(lambda x: x[i] == '0', lines_co2))
            if zero_count_co2 <= len(lines_co2) - zero_count_co2:
                lines_co2 = [x for x in lines_co2 if x[i] == '0']
            else:
                lines_co2 = [x for x in lines_co2 if x[i] == '1']

    print(int(lines_oxy[0], 2) * int(lines_co2[0], 2))
```

So far, the biggest challenge has been properly reading the puzzle descriptions.

## Day 4

This was a pretty fun one: we have to play bingo with a giant squid (because why not). I created a bingo-board class to encapsulate all the logic needed:

```python
class BingoBoard:
    def __init__(self, board_values: List[List[int]]):
        self.board = board_values
        self.board_marks = [[False for _ in x] for x in board_values]
        self.width = len(board_values[0])
        self.height = len(board_values)
        self.is_bingo = False

    def mark(self, value: int):
        for x in range(0, self.height):
            for y in range(0, self.width):
                if self.board[x][y] == value:
                    self.mark_pos(x, y)

    def mark_pos(self, x: int, y: int):
        self.board_marks[x][y] = True
        if not self.is_bingo:
            self.is_bingo = all(self.board_marks[x]) or all(row[y] for row in self.board_marks)

    def score(self) -> int:
        s = 0
        for x in range(0, self.height):
            for y in range(0, self.width):
                if not self.board_marks[x][y]:
                    s += self.board[x][y]
        return s
```

The board keeps two list-of-lists: the values of the squares, and whether each square has been checked or not. The _mark_ method takes a number, marks any squares where that number is found, and checks if we have a bingo. The board also implements this particular flavor of bingo's strange scoring function.

With this in place, the rest is just bookkeeping. We parse the input:

```python
bingo_boards = []

with open('input.txt', 'r') as f_input:
    bingo_nrs = list(map(lambda x: int(x), f_input.readline().split(',')))
    f_input.readline()  # blank line

    while True:
        board = []

        while True:
            board_line = f_input.readline()

            if not board_line or board_line == '\n':    # EOF returns empty string; empty line returns newline
                break

            board_nrs = list(map(lambda x: int(x), [board_line[i:i + 2] for i in range(0, len(board_line), 3)]))
            board.append(board_nrs)

        bingo_boards.append(BingoBoard(board))

        if not board_line:
            break
```

...and then run the game, keeping track of each board's score:

```python
bingo_board_winners = []

for nr in bingo_nrs:
    for b in [x for x in bingo_boards if not x.is_bingo]:
        b.mark(nr)

        if b.is_bingo:
            bingo_board_winners.append(nr * b.score())
```

For part 1, we need the first score; for part 2, the last one:

```python
print(bingo_board_winners[0])
print(bingo_board_winners[-1])
```

Bingo!

## Day 5

Today brings us lines on a grid, and finding the places where they overlap. The grid will be represented as a [defaultdict](https://docs.python.org/3/library/collections.html#collections.defaultdict) of x-y coordinate tuples, with the value representing the number of lines at each point and default value being zero. We'll then add a simple function for adding a line to this grid between two points, which entails incrementing the count for each coordinate between the points:

```python
import re
from collections import defaultdict

sea_grid = defaultdict(lambda: 0)

def add_line(x1: int, y1: int, x2: int, y2: int):
    dx = max(-1, min(1, x2 - x1))
    dy = max(-1, min(1, y2 - y1))

    x_curr = x1
    y_curr = y1

    while x_curr != x2 or y_curr != y2:
        sea_grid[(x_curr, y_curr)] += 1
        x_curr += dx
        y_curr += dy

    sea_grid[(x_curr, y_curr)] += 1     # don't forget (x2, y2)
```

For the first part, we ignore the diagonal lines, add everything else, and then just count the number of values in our dictionary that are greater than 1:

```python
def solution_part_1():
    with open('input.txt', 'r') as f_input:
        for line in f_input:
            x1, y1, x2, y2 = map(lambda x: int(x), re.match(r'(\d+),(\d+) -> (\d+),(\d+)', line).groups())

            if x1 == x2 or y1 == y2:
                add_line(x1, y1, x2, y2)

    print(len([x for x in sea_grid.values() if x > 1]))
```

Then, for the second part, we can add the diagonals and repeat:

```python
def solution_part_2():
    with open('input.txt', 'r') as f_input:
        for line in f_input:
            x1, y1, x2, y2 = map(lambda x: int(x), re.match(r'(\d+),(\d+) -> (\d+),(\d+)', line).groups())

            # assume solution_part_1 ran before this, so only consider diagonals
            if x1 != x2 and y1 != y2:
                add_line(x1, y1, x2, y2)

    print(len([x for x in sea_grid.values() if x > 1]))
```

And that's the first 5 puzzles already done!
