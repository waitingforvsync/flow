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

This sets the first requirement: that sprites may never overlap with non-blank background tiles (although we will try to support moving behind background tiles - more on that later).

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
