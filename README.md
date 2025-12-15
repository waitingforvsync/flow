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

![Dirty sprite](/docs/sprite.svg)

### What about when sprites cross each other?

Then we need to perform masking writes. But only in the places where they actually overlap. The first of the sprites can still be plotted by overwriting. Any sprites which overlap, should then be plotted with a slower masked path:

```
LDA (spritedata),Y
STA maskindex+1
LDA (screen),Y
.maskindex
AND masktable    ; page aligned, self modify operand LSB
ORA maskindex+1
STA (screen),Y
INY
```

This is 15 cycles per byte more than the unmasked write, so should be avoided where possible. We will try to limit the use of this path to just bytes where sprites are overlapping.

### Avoid flicker

We don't have the luxury of double buffering, so we will need to race the beam. This means we should be plotting the sprites roughly from top to bottom, but with the important requirement that if any group of sprites is overlapping, the members of that group are plotted in a stable order (not according to their position), so that their stacking position is preserved.

We should aim to keep the dirty edge erasure as temporally close as possible to the sprite plotting, so that, even if the beam catches up, at worst we get tearing rather than flicker.

### Keep it small!

It would be so easy to detect overlaps if we just kept a high-level mirror of the screen character blocks in memory with a bitfield representing which sprites appear where (this could also be reused for collision detection!). But a 80x24x16 bits buffer is probably more than we can afford. So we will need to find an alternative way.


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

We can exploit the fact that sprites move small distances each frame by making this list persistent across frames. When a sprite changes its y position, we "bubble" its entry in the dirty list backwards or forwards, as appropriate, until it is correctly sorted.

Now we consider character rows of the screen.

We are going to update each row of the screen in isolation, from top to bottom, keeping track of the dirty rectangles which overlap that row. First we will erase the dirty edges in this row, then we will plot the segment of the sprite in its new position in this row.

Essentially we are breaking sprites up into row-wise chunks. Although that might sound inefficient, it is contiguous screen memory on the BBC Micro. Also, any data we set up for the first sprite row can be kept persistent, so we can pick up where we left off over subsequent rows. 

### Active list

The first row we update comes from the first entry in the dirty list. We can skip straight to this row as there's nothing above here. We pull this index, and any subsequent indices whose dirty rectangles also start in this character row, into a new list which we'll call the **active list**.

The active list contains the sprite indices which we have to consider in this row. It will persist across multiple rows.

We also have to remember to remove indices from the active list when the row is no longer in the dirty rectangle.

### Render list

Sprites in the active list don't necessarily have any data to render; we might just need to erase their dirty edge. For example, if a sprite is moving downwards, perhaps it just left a dirty edge in the first row which needs erasing.

So we will maintain a separate list, the **render list** which contains just sprites which need to be plotted on this row. While the active list is unordered, we will make a point of ordering the render list by sprite index. When we walk this, this will be the basis of the stacking order for when sprites overlap.

We need to observe changes to the render list (indices being added or removed) as the next part of the overlap detection.

### Overlaps

We are already detecting overlaps in the y direction, by virtue of maintaining a sorted list of dirty rectangles and walking them from top to bottom. At a given moment, all the entries in the active list have dirty rectangle overlaps in y.

To detect overlaps in the x direction, we will take a different approach.

Let's suppose we are using a BBC Micro screen mode with 80 characters (bytes) across. All we want to know is whether a given byte has already had a sprite plotted in it; if so, we should enable masked plotting. So we just need a bitfield of 80 bits, initialized to all zeroes, which we will call the **occupancy mask**.

As we walk the render list (in sprite index order), get the bounds of the sprite in x, and look in the occupancy mask for those x positions. A clear bit means that character column will be overwritten; a set bit means it will be written masked. Let's build a **render type bitmask** per sprite containing a copy of this data, up to a maximum width of 8 characters. And finally set the bit in the occupancy mask for subsequent entries in the active list.

For a further optimisation, let's only initialise the occupancy mask when the render list *changes*. While the same sprites span consecutive rows, the occupancy data doesn't change, and we save ourselves some processing!

Now, for the current row, we have a list of sprites to plot, ordered by index (via the render list), and for each character column of each sprite, a bitmask saying whether to plot that byte column with the quick path (overwrite) or the slow path (masked).

### Render

Finally we're in a position to render this row. First erase the dirty edges for this row. Then render the sprite rows from the render list, in index order, according to the render type bitmask.

Where a sprite covers multiple rows, much of the time, the render type bitmask and dirty edges will persist from one row to the next, making it quick to render, while also strictly respecting the raster racing.

