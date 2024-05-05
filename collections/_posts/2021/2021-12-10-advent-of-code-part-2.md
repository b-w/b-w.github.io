---
layout: post
title: "Advent of Code 2021, days 6 to 10"
categories: blog
---

Continuing our Advent of Code adventure from [last time]({% post_url 2021/2021-12-05-advent-of-code-part-1 %}). Let's see what the next batch of puzzles has in store.

## Day 6

Simulating the life cycles of lanternfish... My implementation for part 1 was naive: I just keep a list of fish, and update/extend the list at every step:

```python
def simulate(days: int) -> int:
    with open('input.txt', 'r') as f_input:
        fish = list(map(lambda x: int(x), f_input.readline().split(',')))

    for _ in range(days):
        new_fish_count = 0
        for i, f in enumerate(fish):
            if f == 0:
                fish[i] = 6
                new_fish_count += 1
            else:
                fish[i] -= 1

        if new_fish_count > 0:
            fish.extend([8 for _ in range(new_fish_count)])

    return len(fish)
```

This does work, but after completing part 1 we are hit with AoC's infamous "part 2 is just part 1, but more steps". We don't have to do anything different for part 2; we just have to simulate 256 days instead of 80.

The original naive implementation doesn't scale to this size, as even assuming 1 byte per fish, the final list would require twice as much RAM as I have in this computer. Not to mention constantly enumerating and updating this giant list becomes undoable long before that.

After some _deep_ pondering, I realized that we don't have to keep the list of all individual fish at all; we just have to keep track of how many fish there are of each age, and update these numbers at each step. The new simulation function then becomes:

```python
def simulate(days: int) -> int:
    with open('input.txt', 'r') as f_input:
        fish = list(map(lambda x: int(x), f_input.readline().split(',')))

    # map of the number of fishes for each age (age can be 0..8)
    fish_age_count = defaultdict(lambda: 0)

    for f in fish:
        fish_age_count[f] += 1

    for _ in range(days):
        fish_age_count_new = {}

        # aging for fishes 1..8 -> simply age by 1 day
        for i in range(1, 9):
            fish_age_count_new[i - 1] = fish_age_count[i]

        # aging for fishes 0 -> create new fishes with age 8, then add to population of age 6
        fish_age_count_new[6] += fish_age_count[0]
        fish_age_count_new[8] = fish_age_count[0]

        for i in range(9):
            fish_age_count[i] = fish_age_count_new[i]

    return sum([fish_age_count[i] for i in range(9)])
```

This runs pretty much instantly for both parts.

## Day 7

This one wasn't very interesting to me. It's more a math/statistics question as opposed to a programming puzzle.

For part 1, I guessed that we needed either the mean or the median. I tried both on the example, and it turned out we needed the median. I applied that to my input, and apparently got the right answer. Great.

```python
import statistics

with open('input.txt', 'r') as f_input:
    positions = list(map(lambda x: int(x), f_input.readline().split(',')))

def solution_part_1():
    target = int(statistics.median(positions))

    print(sum([abs(p - target) for p in positions]))
```

Part 2 was more guesswork on my part.

First, the new movement cost. I suspected there would be some easy way of computing it, so I literally googled "1+2+3+4" and to my surprise found [a Wikipedia article of the same name](https://en.wikipedia.org/wiki/1_%2B_2_%2B_3_%2B_4_%2B_%E2%8B%AF), which gave me the movement cost formula.

```python
def movement_cost(x: int) -> int:
    return (x * (x + 1)) // 2
```

Bruteforcing a whole bunch of positions wouldn't be very efficient, so I again made an assumption and guessed that the optimal position would be somewhere close to the mean. So I did a kind of guided brute-force around that guess, and again got the correct answer.

```python
def solution_part_2():
    mean_pos = int(statistics.mean(positions))

    print(min(
        [sum(
            [movement_cost(abs(p - t)) for p in positions]
        ) for t in range(mean_pos - 10, mean_pos + 10)]     # guess that the best pos is somewhere close to mean
    ))
```

I leaned nothing today.

## Day 8

Again, it wasn't really about programming today. This one is more of a logic puzzle.

I spent a decent amount of time up front to figure out how to decode the numbers, and programmed that deduction logic into a class:

```python
class InputLine:
    def __init__(self, l: str):
        split = l.split(' ')
        self.patterns = split[:10]
        self.output = split[-4:]
        self.digit_map = {}

        # 1 is the only 2-segment number
        enc_1 = next(x for x in self.patterns if len(x) == 2)
        self.digit_map[''.join(sorted(enc_1))] = 1

        # 7 is the only 3-segment number
        enc_7 = next(x for x in self.patterns if len(x) == 3)
        self.digit_map[''.join(sorted(enc_7))] = 7

        # 4 is the only 4-segment number
        enc_4 = next(x for x in self.patterns if len(x) == 4)
        self.digit_map[''.join(sorted(enc_4))] = 4

        # 8 is the only 7-segment number
        enc_8 = next(x for x in self.patterns if len(x) == 7)
        self.digit_map[''.join(sorted(enc_8))] = 8

        # 5-segment numbers: 2, 3, 5
        # 6-segment numbers: 0, 6, 9

        # 6 is the only 6-segment number that is not a superset of 7
        enc_6 = next(x for x in self.patterns if len(x) == 6 and not set(x) > set(enc_7))
        self.digit_map[''.join(sorted(enc_6))] = 6

        # 5 is the only 5-segment number that is a subset of 6
        enc_5 = next(x for x in self.patterns if len(x) == 5 and set(x) < set(enc_6))
        self.digit_map[''.join(sorted(enc_5))] = 5

        # 0 is the only 6-segment number that is not a superset of 5
        enc_0 = next(x for x in self.patterns if len(x) == 6 and not set(x) > set(enc_5))
        self.digit_map[''.join(sorted(enc_0))] = 0

        # 3 is the only 5-segment number that is a superset of 7
        enc_3 = next(x for x in self.patterns if len(x) == 5 and set(x) > set(enc_7))
        self.digit_map[''.join(sorted(enc_3))] = 3

        # 9 is the only 6-segment number that is a superset of 3
        enc_9 = next(x for x in self.patterns if len(x) == 6 and set(x) > set(enc_3))
        self.digit_map[''.join(sorted(enc_9))] = 9

        # 2 is the only 5-segment number that is not a subset of 9
        enc_2 = next(x for x in self.patterns if len(x) == 5 and not set(x) < set(enc_9))
        self.digit_map[''.join(sorted(enc_2))] = 2

    def value(self) -> int:
        v = 0
        v += self.digit_map[''.join(sorted(self.output[0]))] * 1000
        v += self.digit_map[''.join(sorted(self.output[1]))] * 100
        v += self.digit_map[''.join(sorted(self.output[2]))] * 10
        v += self.digit_map[''.join(sorted(self.output[3]))]
        return v

input_lines = []

with open('input.txt', 'r') as f_input:
    for line in f_input:
        input_lines.append(InputLine(line.rstrip()))
```

And that's basically it.

For part 1, we just count the number of times the unique-length digits occur:

```python
def solution_part_1():
    print(sum([len([o for o in l.output if len(o) in [2, 3, 4, 7]]) for l in input_lines]))
```

Then for part 2, we sum the actual values:

```python
def solution_part_2():
    print(sum([l.value() for l in input_lines]))
```

gg, ez.

## Day 9

This one is a bit more interesting. We have a hightmap (basically a grid), and we have to find out some of its properties.

First, parse the input:

```python
# (0,0) is top-left
heightmap = []

with open('input.txt', 'r') as f_input:
    for line in f_input:
        heightmap.append(list(map(lambda x: int(x), line.rstrip())))

height = len(heightmap)
width = len(heightmap[0])
```

To make things slightly less confusing, I also add a helper function to get the value for a given x-y position:

```python
def value(x: int, y: int) -> int:
    return heightmap[y][x]
```

For the first part, we need to find all local minima, which are points that are lower than all of their neighbors. We'll first create a helper function to fetch the values of a point's neighbors:

```python
def neighbor_values(x: int, y: int) -> List[int]:
    n_list = []
    # top
    if y > 0:
        n_list.append(value(x, y - 1))
    # bottom
    if y < height - 1:
        n_list.append(value(x, y + 1))
    # left
    if x > 0:
        n_list.append(value(x - 1, y))
    # right
    if x < width - 1:
        n_list.append(value(x + 1, y))
    return n_list
```

Then, the local-minimum function is pretty simple:

```python
def is_local_minimum(x: int, y: int) -> bool:
    point_value = value(x, y)
    return all([p > point_value for p in (neighbor_values(x, y))])
```

For part 1, we then just sum the values of all local minima:

```python
def solution_part_1():
    risk = 0
    for i_x in range(width):
        for i_y in range(height):
            if is_local_minimum(i_x, i_y):
                risk += 1 + value(i_x, i_y)
    print(risk)
```

That was pretty easy, but part 2 gets a bit more tricky. Now we have to find the three largest basins, which are areas on the grid enclosed by points with value 9.

For this I just loop through all points in the grid, and from each point that I haven't seen yet, I start a BFS along its neighbors, stopping when I hit the borders or nodes with value 9\. I first need a second neighbor function that returns coordinates instead of values:

```python
def neighbor_coords(x: int, y: int) -> List[Tuple[int, int]]:
    n_list = []
    # top
    if y > 0:
        n_list.append((x, y - 1))
    # bottom
    if y < height - 1:
        n_list.append((x, y + 1))
    # left
    if x > 0:
        n_list.append((x - 1, y))
    # right
    if x < width - 1:
        n_list.append((x + 1, y))
    return n_list
```

I then go through the whole grid, add all of the basins I find to a list, and then sort by length to get the top 3.

```python
def solution_part_2():
    seen = set()
    basins = []
    for i_x in range(width):
        for i_y in range(height):
            n_v = value(i_x, i_y)

            if n_v == 9 or (i_x, i_y) in seen:
                continue

            seen.add((i_x, i_y))
            basin = [(i_x, i_y)]
            to_check = Queue()

            for n in neighbor_coords(i_x, i_y):
                if n not in seen:
                    to_check.put(n)
                    seen.add(n)

            while to_check.qsize() > 0:
                n = to_check.get()
                n_x, n_y = n
                n_v = value(n_x, n_y)

                if n_v == 9:
                    continue

                basin.append(n)

                for n in neighbor_coords(n_x, n_y):
                    if n not in seen:
                        to_check.put(n)
                        seen.add(n)

            basins.append(basin)

    basins.sort(key=len)
    print(len(basins[-1]) * len(basins[-2]) * len(basins[-3]))
```

The key to this solution, besides the BFS, was using a set instead of a list to keep track of seen nodes, which drastically improved runtime.

## Day 10

This one was fun. After all, dealing with mismatched brackets is part of a programmer's daily routine. As such, it was immediately obvious to me that all we needed was a stack to create a simple scoring function for part 1:

```python
from collections import deque

bracket_pairs = {('(', ')'), ('[', ']'), ('{', '}'), ('<', '>')}

def score_illegal(line: str) -> int:
    brackets = deque()
    for c in line:
        if c in ('(', '[', '{', '<'):
            brackets.append(c)
        else:
            c_open = brackets.pop()
            c_pair = (c_open, c)
            if c_pair not in bracket_pairs:
                match c:
                    case ')':
                        return 3
                    case ']':
                        return 57
                    case '}':
                        return 1197
                    case '>':
                        return 25137
    return 0
```

Here we just find the first non-matching pair, and return the appropriate score.

We then just sum all the scores for part 1:

```python
with open('input.txt', 'r') as f_input:
    nav_prgm = [x.rstrip() for x in f_input.readlines()]

def solution_part_1():
    print(sum([score_illegal(x) for x in nav_prgm]))
```

Part 2 is very similar, only this time we have to score the missing brackets. We read through the line and push/pop all brackets, and then end by scoring the remaining open brackets:

```python
def score_incomplete(line: str) -> int:
    brackets = deque()
    for c in line:
        if c in ('(', '[', '{', '<'):
            brackets.append(c)
        else:
            brackets.pop()  # assumes that this matches, and it's not an illegal line

    score = 0
    while len(brackets) > 0:
        c = brackets.pop()
        score *= 5
        match c:
            case '(':
                score += 1
            case '[':
                score += 2
            case '{':
                score += 3
            case '<':
                score += 4
    return score
```

We score all non-illegal lines this way, and then select the middle score:

```python
def solution_part_2():
    scores = []
    for line in nav_prgm:
        if score_illegal(line) == 0:
            scores.append(score_incomplete(line))
    scores.sort()
    print(scores[len(scores) // 2])
```

And that's another 5 days done!
