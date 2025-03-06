
# 1 Board

We'll start off this tictactoe machine learning course by implementing the game itself.

If you're mostly interested in the machine learning aspect, you can speed through this chapter and just copy the codeblocks into the correct places. The knowledge in this chaper is irrelevant to machine learning specifically.

## 1.1 Game rules

Very quick introduction to tictactoe. 2 players take turns placing pieces on a 3-by-3 board. One player is cross, the other is circle. The first player to make a 3-in-a-row pattern is the winner. If the board i filled up before that happens, the game is a tie.[1]

## 1.2 Board and pieces

The game consists of a game board. The 3-by-3 board has 9 fields. Each field is at any point in the game either empty, a cross or a circle. Let's define those 3 states as integer values.

```py
EMPTY = 0
CROSS = 1
CIRCLE = 2
```

Each field can also be represented by an integer, if we assign a unique integer to each field. We could assign integers like this:

```
+---+---+---+
| 0 | 1 | 2 |
|---+---+---|
| 3 | 4 | 5 |
|---+---+---|
| 6 | 7 | 8 |
+---+---+---+
```

## 1.3 Board as a bit field

A piece can be stored in 2 bits, as can be seen by the following table.

Piece|Decimal|Binary
---|---|---
`EMPTY` | `0` | `0b00`
`CROSS` | `1` | `0b01`
`CIRCLE` | `2` | `0b10`

The last value (`3`, `0b11`) will not be used.

A bit field is just an integer value, where the individual bits are used. Since each piece can be stored in 2 bits, we can use a bit field to store the board. We need 2 bit * 9 fields = 18 bits, meaning the `int` in Python will suffice. We'll place each piece's value into the bit field at the positions offset. See the following example.

```
... 00 00 00 00 00 00 00 00 00
                   ^^       ^^-- position 0
                   --- position 3       
```

To encapsulate this logic, we'll make a `Board`-class with a bit field data-member.

```py
class Board:
    def __init__(self) -> None:
        self.bitfield = 0
```

The board should start empty. Since `EMPTY` is `0`, we can assign zero like this.

## 1.4 Get a piece at a position

To manipulate the board, we need 2 methods. One to insert a piece, and one to inspect the piece at a position. We'll implement these using bitshifting and bitwise operations. They might be unfamiliar, especially in Python, but hang in there.

To insert and inspect pieces we need the size of a piece, which we know to be `2`. We also need what's called a bit mask. For a 2 bit value, we need a bit mask where the first 2 bits are set. We can define these as constants.

```py
PIECE_SIZE = 2
PIECE_MASK = 0b11
```

We'll implement inspect first, as a method `piece_at` on the board class we just defined.

```py
class Board:
    # ...
    def piece_at(self, pos: int) -> int:
        return self.bitfield >> pos * PIECE_SIZE & PIECE_MASK
```

Now, to explain what the right-shift `>>`-operator and bitwise AND `&`-operator does, I'll lay out the bits. Imagine the following board.

```
+---+---+---+
|   |   | O |
|---+---+---|
| X |   |   |
|---+---+---|
|   |   |   |
+---+---+---+
```

This board would be represented in the bit field like the following.

```
    8  7  6  5  4  3  2  1  0
... 00 00 00 00 00 01 10 00 00
                   ^^ ^^-- 0b10 = CIRCLE
                   --- 0b01 = CROSS
```

Since we're working with an integer bigger than 18 bits, the bits after the 9th piece are all zero.

We want to inspect the piece at position 2 (same as calling `.at(pos=2)`). To do this we'll have to apply 2 bit operations. The first is a bitshifting operation, that moves the interesting into the correct position. The amount we need to shift, is enough to move the piece's bits into the first spot. `pos` tells us which piece, and `PIECE_SIZE` tells us how many bits a piece occupies, therefore `pos * PIECE_SIZE` will tell us, how many bits to shift (4 in this case).

```
bitfield:
8  7  6  5  4  3  2  1  0
00 00 00 00 00 01 10 00 00

bitfield >> 4

bitfield:
      8  7  6  5  4  3  2
00 00 00 00 00 00 00 01 10
```

Notice that the piece's bits are at the first spot. The next operation seeks to extract the piece's 2 bits from the board. We do this using the bit mask. Remember the boolean AND operation

A | B | A and B
--|--|--
False | False | False
**True** | False | False
False | **True** | False
**True** | **True** | **True**

The same principle can be applied to bits, eg. `1 and 1 = 1` and `0 and 1 = 0`. The bitwise AND `&`-operator applies this to every bit in the integer.

```
bitfield:
      8  7  6  5  4  3  2
00 00 00 00 00 00 00 01 10

bitfield & 0b11


  00 00 00 00 00 00 00 01 10
& 00 00 00 00 00 00 00 00 11
----------------------------
= 00 00 00 00 00 00 00 00 10

bitfield:
                        2
00 00 00 00 00 00 00 00 10
```

Now we have essentially erased the other pieces in the produced value. Now, if we assign the value to a variable, we have a piece.

> [!IMPORTANT]  
> Bitwise and bitshit operations are generally hard to understand just looking at them. If you experience problems following along, make sure that the operations are done like shown here.

## 1.5 Place a piece on a position

In addition to getting a piece, we also need a method to place pieces on the board. We'll also define the `place_piece`-method using bit-operators.

```py
class Board:
    # ...
    def place_piece(self, piece: int, pos: int):
        self.bitfield |= piece << pos * PIECE_SIZE
```

To explain the first part `piece << pos * PIECE_SIZE`, we'll again use visual representation of the board's bits.

```
piece = CIRCLE = 0b01
pos = 2

piece:
00 00 00 00 00 00 00 00 01

pos * PIECE_SIZE = 2 * 2 = 4

piece << 4:
00 00 00 00 00 00 01 00 00
```

Notice that the `1` in this case moves 4 bits to the right.

The `|=` is 2 operators combined. The expression `a |= b` is the same as `a = a | b`. This is just like addition, where `a += b` is the same as `a = a + b`. This means that `self.bitfield |= piece << pos * PIECE_SIZE` is the same as `self.bitfield = self.bitfield | piece << pos * PIECE_SIZE`. We'll focus on `self.bitfield | piece << pos * PIECE_SIZE`.

The OR `|`-operator is similar to the AND `&`-operator, but using OR-logic, instead of AND-logic. Compare the following table with the table for AND above.

A | B | A and B
--|--|--
False | False | False
**True** | False | **True**
False | **True** | **True**
**True** | **True** | **True**

We can visualize the bitwise operation like this:

```
bitfield:
8  7  6  5  4  3  2  1  0
00 00 00 00 00 00 00 00 00


piece << 4:
00 00 00 00 00 00 01 00 00


  00 00 00 00 00 00 00 00 00
| 00 00 00 00 00 00 01 00 00
----------------------------
= 00 00 00 00 00 00 01 00 00
```

By this, it can be seen that the piece will be inserted correctly.

## 1.6 

[1]: There are unlimited pieces, but pieces can't be moved. This is in contrast to rules regarding the same game, inwhich there are only 3 of each piece, and then the pieces can be moved. We'll use the former ruleset.


