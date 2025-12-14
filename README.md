# flow
Sprite engine for the BBC Micro.

This is a prototype of a (hopefully) fast and flexible sprite engine which could be used in a real game with a large number of moving sprites in a smooth scrolling world.

## Basic principles

### Write unmasked sprites where possibile

Deleting and replotting masked sprites is slow, so try to avoid doing it at all!

Ideally, we will always try to directly write sprites unmasked to the screen:

```
LDA (spritedata),Y
STA (screen),Y
INY
```

This sets the first requirement: that sprites may never overlap with non-blank background tiles (although we will try to support moving *behind* background tiles - more on that later).

Moving sprites will normally have either one or two dirty edges (they may have even more if they are also changing size). These edges will be erased just before the sprite is written unmasked in its new position. Hence, a sprite moving horizontally has its dirty side edge erased, and then overwrites the rest of itself at its new position. It doesn't get faster than this!

### What about when sprites cross each other?

Then we need to perform masking writes. But only in the places where they actually overlap. The first of the sprites can still be plotted by overwriting. Any sprites which overlap, should then be plotted with a slower masked path:

```
LDA (spritedata),Y
STA maskindex+1
.maskindex
LDA masktable    ; page aligned, self modify operand LSB
ORA maskindex+1
STA (screen),Y
INY
```

Clearly this is slower so should be avoided where possible. We will try to limit the use of this path to just bytes where sprites are overlapping.

### Avoid flicker

We don't have the luxury of double buffering, so we will need to race the beam. This means we should be plotting the sprites roughly from top to bottom, but with the important requirement that if any group of sprites is overlapping, the members of that group are plotted in a stable order (not according to their position), so that their stacking position is preserved.

We should aim to keep the dirty edge erasure as temporally close as possible to the sprite plotting, so that, even if the beam catches up, any flicker is avoided.


## Implementation overview

This is a surprisingly complicated process which can be broken into various steps. For the moment, we will consider how to make it work in a static screen; then we can see how to adapt it so it works with scrolling.

### Definitions

A **sprite** has the following data associated with it:

- Index (this is implicit)
- old x, y, width, height
- new x, y, width, height
- sprite data pointer

The old and new x, y, width, height can be seen as a bounding rectangle.

The **dirty rectangle** of a sprite is the union of the old and new bounding rectangles.

### Dirty list

We maintain a list of sprite indices, sorted by the y position of their dirty rectangle. We'll call this the **dirty list**.

We can exploit the fact that sprites move small distances each frame, by making this list persistent across frames. When a sprite changes its y position, we "bubble" its entry in the dirty list backwards or forwards, as appropriate, until it is correctly sorted.

Now we consider character rows of the screen.

We are going to update each row of the screen in isolation, from top to bottom, keeping track of the dirty rectangles which overlap that row. First we will erase the dirty edges in this row, then we will plot the segment of the sprite in its new position in this row.

The first row we update comes from the first entry in the dirty list. We can skip straight to this row as there's nothing above here. We pull this index, and any subsequent indices whose dirty rectangles also start in this character row, into a new list which we'll call the **active list**.

The active list contains the sprite indices which we have to consider in this row. It will persist across multiple rows.

We also have to remember to remove indices from the active list when the row is no longer in the dirty rectangle.

### Render list

