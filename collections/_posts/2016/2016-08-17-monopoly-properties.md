---
layout: post
title: "Finding the most valuable Monopoly properties"
categories: blog
---

I think everyone has at least heard of the game of [Monopoly](https://en.wikipedia.org/wiki/Monopoly_(game)). It's been years since I last played it, but the game still facinates me. I've always wondered if it was possible to build a monopoly AI who could play the game flawlessly, and what the most optimal strategy was. I won't be going that far in this post, but I will attempt to find the most valuable property spaces. I'll do this by simulating a whole bunch of monopoly games, and keeping track of how many times each property space is visited. I'll be using Python 3 for this project.

We'll start by defining a Monopoly space:

```python
class Space(object):
    def __init__(self, name):
        self.landing_count = 0
        self.start_count = 0
        self.name = name

    def player_landing(self, player):
        self.landing_count += 1

    def player_starting(self, player):
        self.start_count += 1
```

A space will keep track of how many times a player lands on it, and how many times a player starts his turn on it. If it's not immediately obvious that these values can be different, consider this: a player can land on the "go to jail" space, but he can never start his turn there.

Next, we'll define two special spaces: the "go to jail" space, and the "Chance" and "Community Chest" spaces (grouped in a single "card" space):

```python
class GoToJailSpace(Space):
    def __init__(self):
        super().__init__("Go to Jail")

    def player_landing(self, player):
        super().player_landing(player)
        player.goto_jail()

class CardSpace(Space):
    def __init__(self, name, deck):
        super().__init__(name)
        self.deck = deck

    def player_landing(self, player):
        super().player_landing(player)
        self.deck.player_draw(player)
```

The "go to jail" space moves the player to the jail space and sets his "in jail" flag. The card space directs the player to draw a card from its deck. Next we'll take a look at how these cards are implemented.

First of all we define what a card is. A card is a simple object containing a single "draw" function that gets called whenever a player draws it.

```python
class Card(object):
    def player_draw(self, player):
        return
```

For the basic card object, this draw function does nothing. In our simulation we're only interested in cards that affect player movement and board position. We won't actually implement the other types of cards, but we still have to account for their presence in the deck. So, they will simply be added to the deck as cards that do nothing.

As for the movement cards, there's essentially four different types:

1.  Cards that move the player to an absolute position (e.g. "advance to Go");
2.  Cards that move the player relative to his current position (e.g. "go back 3 spaces");
3.  Cards that move the player to the nearest space in a given set (e.g. "advance to the nearest Railroad");
4.  Cards that move the player to jail.

We implement them as follows:

```python
class CardMoveTo(Card):
    def __init__(self, position):
        self.position = position

    def player_draw(self, player):
        player.position = self.position
        player.handle_landing()

class CardMoveRelative(Card):
    def __init__(self, count):
        self.count = count

    def player_draw(self, player):
        player.move(self.count)
        player.handle_landing()

class CardMoveToNearest(Card):
    def __init__(self, positions):
        self.positions = positions
        self.positions.sort()

    def player_draw(self, player):
        target_pos = -1

        for i in self.positions:
            if i > player.position:
                target_pos = i
                break

        if target_pos == -1:
            target_pos = self.positions[0]

        player.position = target_pos
        player.handle_landing()

class CardGoToJail(Card):
    def player_draw(self, player):
        player.goto_jail()
```

Note we don't consider the "get out of jail free" card in our simulation, even though it's technically the fifth type of card that can affect player movement.

Cards are kept in a deck, which is responsible for handling a player drawing a card from it, as well as re-shuffling the deck after all cards have been drawn. The deck is implemented as follows:

```python
class CardDeck(object):
    def __init__(self, cards):
        self.cards = cards
        self.shuffle_cards()

    def shuffle_cards(self):
        self.card_index = 0
        random.shuffle(self.cards)

    def player_draw(self, player):
        card = self.cards[self.card_index]
        card.player_draw(player)

        self.card_index += 1
        if self.card_index == len(self.cards):
            self.shuffle_cards()
```

Now that we have the spaces and the cards in place, we can start working on the player. We also create a helper function to handle dice rolls.

```python
class Dice(object):
    @staticmethod
    def roll():
        return random.randint(1, 6), random.randint(1, 6)

class Player(object):
    def __init__(self, board):
        self.board = board
        self.position = 0
        self.my_turn = False
        self.in_jail = False
        self.in_jail_count = 0

    def move(self, count):
        self.position = (self.position + count) % board_size

    def goto_jail(self):
        self.position = 10
        self.my_turn = False
        self.in_jail = True

    def leave_jail(self):
        self.in_jail = False
        self.in_jail_count = 0

    def handle_landing(self):
        self.board.spaces[self.position].player_landing(self)

    def handle_starting(self):
        self.my_turn = True
        self.board.spaces[self.position].player_starting(self)
```

The player maintains a few variables (like his position, whether it's his turn, and whether he's in jail) and is responsible for moving around the board. He also keeps a reference to the game board itself. Landing and starting events are simply delegated to the space he's currently on.

Next, we'll have the pleasant task of initializing the Monopoly game board. We'll also define a few global variables.

```python
board_size = 40
card_chance_count = 16
card_chest_count = 16
max_jail_stays = 3
doubles_to_jail = 3

class Board(object):
    def __init__(self):
        cards_chance = [Card() for _ in range(card_chance_count)]
        cards_chance[0] = CardMoveTo(0)
        cards_chance[1] = CardMoveTo(24)
        cards_chance[2] = CardMoveTo(11)
        cards_chance[3] = CardMoveToNearest([12, 28])
        cards_chance[4] = CardMoveToNearest([5, 15, 25, 35])
        cards_chance[5] = CardMoveToNearest([5, 15, 25, 35])
        cards_chance[6] = CardMoveRelative(-3)
        cards_chance[7] = CardGoToJail()
        cards_chance[8] = CardMoveTo(5)
        cards_chance[9] = CardMoveTo(39)
        self.deck_chance = CardDeck(cards_chance)

        cards_chest = [Card() for _ in range(card_chest_count)]
        cards_chest[0] = CardMoveTo(0)
        cards_chest[1] = CardGoToJail()
        self.deck_chest = CardDeck(cards_chest)

        self.spaces = [None] * board_size
        self.spaces[0] = Space("Go")
        self.spaces[1] = Space("Brown 1")
        self.spaces[2] = CardSpace("Chest 1", self.deck_chest)
        self.spaces[3] = Space("Brown 2")
        self.spaces[4] = Space("Income Tax")
        self.spaces[5] = Space("Railroad 1")
        self.spaces[6] = Space("Blue 1")
        self.spaces[7] = CardSpace("Chance 1", self.deck_chance)
        self.spaces[8] = Space("Blue 2")
        self.spaces[9] = Space("Blue 3")
        self.spaces[10] = Space("Jail")
        self.spaces[11] = Space("Pink 1")
        self.spaces[12] = Space("Electric")
        self.spaces[13] = Space("Pink 2")
        self.spaces[14] = Space("Pink 3")
        self.spaces[15] = Space("Railroad 2")
        self.spaces[16] = Space("Orange 1")
        self.spaces[17] = CardSpace("Chest 2", self.deck_chest)
        self.spaces[18] = Space("Orange 2")
        self.spaces[19] = Space("Orange 3")
        self.spaces[20] = Space("Parking")
        self.spaces[21] = Space("Red 1")
        self.spaces[22] = CardSpace("Chance 2", self.deck_chance)
        self.spaces[23] = Space("Red 2")
        self.spaces[24] = Space("Red 3")
        self.spaces[25] = Space("Railroad 3")
        self.spaces[26] = Space("Yellow 1")
        self.spaces[27] = Space("Yellow 2")
        self.spaces[28] = Space("Water")
        self.spaces[29] = Space("Yellow 3")
        self.spaces[30] = GoToJailSpace()
        self.spaces[31] = Space("Green 1")
        self.spaces[32] = Space("Green 2")
        self.spaces[33] = CardSpace("Chest 3", self.deck_chest)
        self.spaces[34] = Space("Green 3")
        self.spaces[35] = Space("Railroad 4")
        self.spaces[36] = CardSpace("Chance 3", self.deck_chance)
        self.spaces[37] = Space("Dark Blue 1")
        self.spaces[38] = Space("Luxury Tax")
        self.spaces[39] = Space("Dark Blue 2")

        self.players = []
```

This initializes all of the board spaces and card decks. We'll also define a function for starting a new game:

```python
class Board(object):
    # more

    def new_game(self, player_count):
        self.players = [Player(self) for _ in range(player_count)]
```

Note that starting a new game means only resetting the players. We don't actually reset any of the spaces, because we want to keep their player landing and starting tally between games.

Moving on, we'll implement the logic for handling a single turn:

```python
class Board(object):
    # more

        def single_turn(self):
        for player in self.players:
            roll_count = 0
            player.handle_starting()
            while player.my_turn:
                (d1, d2) = Dice.roll()
                roll_count += 1
                is_doubles = (d1 == d2)

                if player.in_jail:
                    if is_doubles:
                        player.leave_jail()
                    else:
                        player.in_jail_count += 1
                        if player.in_jail_count == max_jail_stays:
                            player.leave_jail()
                            player.my_turn = False
                            break

                if roll_count == doubles_to_jail and is_doubles:
                    player.goto_jail()
                    break
                else:
                    player.move(d1 + d2)
                    player.handle_landing()

                if not is_doubles:
                    player.my_turn = False
```

We consider a single "game turn" to consist of a single turn for each player. During his turn, a player can keep rolling the dice indefinitely, until he either

1.  fails to roll a double, or
2.  rolls too many doubles, and is subsequently sent to jail.

You'll note that most lines in that function are just there to deal with the jail: when to send the player to jail, what to do if he is in jail, etc. Regular player movement is captured in two lines:

```python
player.move(d1 + d2)
player.handle_landing()
```

We now have pretty much all we need to simulate a game of Monopoly. But before we can do this, we'll still need a way of viewing the results we're interested in. We'll add a simple helper function to have the game board print the percentages of how many times each of its spaces was landed on:

```python
class Board(object):
    # more

    def print_stats(self):
        total_landing = 0
        total_start = 0
        for x in self.spaces:
            total_landing += x.landing_count
            total_start += x.start_count

        for x in self.spaces:
            print("{:>5f}%\t{:>5f}%\t{:s}".format(
                (x.landing_count / total_landing) * 100,
                (x.start_count / total_start) * 100,
                x.name))
```

And that's it! We can now start simulating games, and figuring out what the most attractive property spaces are. Here's how we simulate 100.000 4-player games consisting of 50 turns per game:

```python
test_board = Board()

for _ in range(100000):
    test_board.new_game(4)
    for _ in range(50):
        test_board.single_turn()

test_board.print_stats()
```

This can take a while, depending on what kind of hardware you have. After a minute or so, we'll get something like this:

    2.875629%	4.941465%	Go
    1.946312%	1.883700%	Brown 1
    2.007422%	1.754095%	Chest 1
    2.064407%	2.047730%	Brown 2
    2.270286%	2.247235%	Income Tax
    2.912282%	2.947920%	Railroad 1
    2.292530%	2.307950%	Blue 1
    2.384518%	0.912430%	Chance 1
    2.369728%	2.400700%	Blue 2
    2.322159%	2.395645%	Blue 3
    2.272191%	6.115145%	Jail
    2.674042%	2.722810%	Pink 1
    2.561125%	2.389360%	Electric
    2.300931%	2.349675%	Pink 2
    2.400827%	2.322135%	Pink 3
    2.871279%	2.966770%	Railroad 2
    2.714201%	2.688495%	Orange 1
    2.875629%	2.644790%	Chest 2
    2.841810%	2.864690%	Orange 2
    2.973502%	3.187150%	Orange 3
    2.774197%	2.858615%	Parking
    2.726091%	2.944895%	Red 1
    2.675631%	1.059430%	Chance 2
    2.617196%	2.871095%	Red 2
    3.048636%	3.404870%	Red 3
    2.926532%	3.189710%	Railroad 3
    2.566232%	2.830390%	Yellow 1
    2.539405%	2.665725%	Yellow 2
    2.652990%	2.796600%	Water
    2.430510%	2.442005%	Yellow 3
    2.469227%	0.000000%	Go to Jail
    2.498570%	2.459680%	Green 1
    2.447098%	2.526085%	Green 2
    2.518340%	2.146245%	Chest 3
    2.315299%	2.371375%	Green 3
    2.256061%	2.188015%	Railroad 4
    2.140715%	0.795585%	Chance 3
    2.021052%	1.927990%	Dark Blue 1
    2.010489%	2.047130%	Luxury Tax
    2.434917%	2.384670%	Dark Blue 2

The first column contains the landed-on percentages, the second column has the turn-starting percentages for each space.

A cursory glance at these numbers reveals what every expert Monopoly player already knows: the 3rd red property is the most landed-on in the game. The first three railroads and the entire orange property set are also very attractive to have. The brown properties seem to be one of the worst sets in the game. An interesting follow-up might be to use these values to compute ROIs for each property, but we'll leave that one as an exercise for the reader...
