---
layout: post
title: 'Building an 8-bit computer: some design decisions'
date: 2018-01-07
status: publish
type: post
published: true
author: Hywel Carver
---

Since teaching myself assembly language (in previous blog posts), building
a computer is my next step to understanding how my code actually gets run.

In the first post on building an 8-bit computer, I laid out my next steps:
deciding which display to use, writing a first version of the code, and working
out all the modifications I need to make to the plans I'm using as a basis.

## Which display to use?

Running the display and writing data to it is likely to be one of the most
clock cycle-consuming parts of the code, so I want to choose carefully. I'll
also have to figure out the wiring / input protocols myself - I've got no
experience with that, so the simpler the better.

* The more complicated the display, the more work (and clock cycles) it will take to use it properly, which will affect the frame rate of the game.
* Bigger screens are better, up to a point. The smaller the display's resolution, the less "work" it will be for the CPU to update it.
* A display whose interface matches my use case will make my life easier.

I had been hoping to find a display interface offering commands like "clear the display" and "turn on the pixel at (17,49)". Given that most of the screen is blank in pong, that would be an efficient way to control the graphics display. A little research showed that displays like that do exist, but they are all quite slow to update - 100s of milliseconds - much too slow for a game.

Instead, I've decided to use an OLED display designed for use with Arduinos and Raspberry Pis. They're already targeted at hobbyists like me and low-performance computers, so should match what I need. One complication is that the standard use case here involves downloading a prewritten library to simplify the process of writing the code. Because I'm not using an Arduino or an RPi, there aren't any libraries for me to use, and I haven't found any tutorials about interfacing with these displays on an electronic level. Instead, I'll have to do some reading and work it out for myself.

The [Adafruit Monochrome 1.3" 128x64 OLED graphic display](https://www.adafruit.com/product/938) looks like a good fit for my needs. Relatively big for an OLED display, each coordinate will fit in a single byte, and designed for hobbyists. I'll take it.

## How to use the display?

I have no experience digging through hardware datasheets and after a morning of diving into them, my head is slightly spinning. The Adafruit I've decided on comes with a driver chip, the SSD1306 [(datasheet!)](https://cdn-shop.adafruit.com/datasheets/SSD1306.pdf) which has a graphics buffer and abstracts away some of the communication with the display itself.

From what I can make out, a lot of the pins are simply for configuring your use case of the display, and they can be tied to constant values. The actual work of displaying pixels can be done either in serial or parallel (8 bits).

Those 8 bits can be used for both commands ("only display from column 7!", "turn the whole display off!") and also for data. So I'll need two instructions, one for sending commands and one for instructions. There's a separate pin on the chip which can be set high or low to tell the chip whether it's receiving data or a command.

8 bits in parallel is a good fit for my needs - fewer operations, and the same width as the registers / bus on the computer I'm building. Unfortunately, the parallel interface seems to have been deprecated from the display itself, so I'm left with a serial one. That means each individual bit has to be sent on a separate clock cycle.

A quick back of the envelope calculation: the display has 128x64 = 8196 pixels, and to update each pixel in serial mode will require at least one clock cycle to update. To hit 10 frames per second, I'd need ~82,000 Hz for the drawing alone. That might be possible, and would be easier if I had a load of memory to keep a buffer of the whole display. I could write the frame to that buffer, then right it out bit-by-bit to the display.

There might be a more efficient way to update the display though - most of the display will be black, and won't need updating on every cycle. The chip allows for multiple method of addressing: one row at a time, one column at a time, one page at a time (a page is a rectangular area of the display). If the chip is in column mode, I can use a command declaring which column I'm going to draw next. And instead of redrawing the whole display on each cycle, I can just redraw the areas which a ball / paddle has just left, and redraw the areas where a ball / paddle has just entered. That also means no need for my own separate graphics buffer.

To make this simple to program, I'll also constrain that the ball doesn't get drawn if it enters the same column as a paddle. I'll still want single instructions to send a command or data to the graphics chip, but there'll also need to be some cleverness to serialise the 8-bits into the graphics chip.

To be honest, it feels like a waste sending the data in serial, because the driver chip itself is probably using a shift register to deserialise the 8-bits into parallel. I'll have a look at the driver when it arrives to see if I can bypass the serial input and input straight into the register.

## RAM, Instruction Set & Code

Let's start with RAM. There are some variables I know I'll need to track.

```
Byte | Contents
---------------
0x00 | left_score
0x01 | right_score (can be combined with left_score in a single byte if needed)
0x02 | ball_x
0x03 | ball_y
0x04 | ball_v_x
0x05 | ball_v_y (can be combined with the x velocity in a single byte if needed)
0x06 | left_paddle_y
0x07 | right_paddle_y
0x08 | last_ball_position_x
0x09 | last_ball_position_y
```

I've also written a list of the CPU-level instructions I think I'll need.
Several of those are included in the plans I'm building on, but some deserve explanation.

* I've used some labels like `:label` to mark the code, and make jumps easier to read.
* Some of the instructions are directly for register manipulation. A CPU has several
small bits of memory called registers for storing the data that it's actively working on.
There are 6 instructions for putting values into each register, storing the values from
registers into RAM, and loading values from RAM into registers.
* There are arithmetic instructions for adding and subtracting between A and B registers.
* Two instructions are for outputting values to the 7-segment displays.
* Two are for getting input from the players.
* Two are for sending commands and data to the display.
* Four are for jumps: changing the flow of the program based on values.
  * A jump sets the next instruction to be executed (which would otherwise be whatever instruction comes next in memory).
  * The conditional jumps are new additions to the instruction set, which only jump if the last value on the bus was either negative / zero / positive.
  * To do this, we'll need an additional "status" or "flags" register, which records whether the last value on the bus was -ve, zero or +ve.
  * Conditional jumps are how branches (like `if` and `else` are implemented in assembly).
* One instruction to halt.

```
Instruction | Description
------------|------------
:label      | Placeholder to make it easier to read jump locations
SETA <val>  | Put a literal value into register A
STA  <loc>  | Store the value in register A to RAM
LDA  <loc>  | Load from a RAM location into register A
SETB <val>  | Put a literal value into register B
STB  <loc>  | Store the value in register B to RAM
LDB  <loc>  | Load from a RAM location into register B
AADDB       | Add the value in register B to register A, stored in A
ASUBB       | Subtract the value in register B from register A, stored in A
OUTL <loc>  | Output a value from memory to the left player's 7-seg display
OUTR <loc>  | Output a value from memory to the right player's 7-seg display
JMP  <loc>  | Jump to a new location
JMP+ <loc>  | Jump if the value is positive
JMP0 <loc>  | Jump if the value is zero
JMP- <loc>  | Jump if the value is negative
INLA        | Put the left-hand input into reg A
INRA        | Put the right-hand input into reg A
GFXC <cmd>  | Output a command byte to the graphics display
GFXD <dat>  | Output a data byte to the graphics display
HLT         | Stop the computer
```

I've used those instructions to make an outline of the code for the game, so that
I can get a sense of how much RAM will be needed to store tho code, and how many clock cycles each frame
will need.

```
// Setup: The score for each player gets set to 0
SETA 0
STA left_score
STA right_score
OUTL left_score
OUTR right_score

GFXC <commands to initialise the display>

:start_a_point
// Put ball in middle, initially moving to one player
SETA 64
STA ball_x
SETA 32
STA ball_y
SETA 2
STA ball_v_x
SETA 1
STA ball_v_y
// Initialise the paddles
SETA 32
STA left_paddle_y
STA right_paddle_y

// Do the work of one frame
:frame
// Draw the current state
GFXC <3 commands required to set a starting column address, set to col 0>
GFXC
GFXC

// TODO Iterate through the 8 bytes of the column, subtracting 8 from the paddle
// height each time.
// If the y is in a certain range (depending on paddle length), then we need
// to draw part of the paddle in this column.
// This might be best implemented with a lookup table in RAM.
GFXD <something>

// TODO Repeat for the other paddle

// TODO compare new ball position with old, and repaint the old ball's area
// if it's different to the new ball's area
// TODO display the dot for the new ball.

// Check for a point being won
LDA ball_x
// Note: the ball being out of bounds will result in a highbit of 1, either
// side, which is the same as the number being negative in 2s complement.
JMP+ :no_point_won
LDA ball_v_x
JMP- :right_point_won
:left_point_won
LDA right_score
STB 1
AADDB
STA right_score
JMP :end_point
:right_point_won
LDA left_score
STB 1
AADDB
STA left_score
JMP :end_point

:no_point_won

// move paddles
INLA
JMP0 :move_right_paddle
JMP- :a_negative
LDA left_paddle_y
SETB 120 // Assuming a height of 8 for the paddle
ASUBB
JMP- :move_right_paddle
LDA left_paddle_y
SETB 1
ASUBB
STA left_paddle_y
JUMP :move_right_paddle

:a_negative
// TODO do the same for a negative input
:move_right_paddle
// TODO do the same for the right paddle

:move_and_collide
// Wall collisions
LDA ball_y
JMP+ :top_wall_bounce
JMP :do_y_bounce
:top_wall_bounce
SETB 63
ASUBB
JMP- :move_ball
:do_y_bounce
SETA 0
LDB ball_v_y
ASUBB
STA ball_v_y

// Bounce from paddles
LDA ball_x
SETB 1
ASUBB
JMP+ :right_paddle
// Check the ball overlaps with the y values of the paddle
// Either bounce or not
// TODO

:right_paddle
// TODO - the same from the right paddle

:do_x_bounce
SETA 0
LDB ball_v_x
ASUBB
STA ball_v_x

// Move ball
:move_ball
LDA ball_x
LDB ball_v_x
AADDB
STA ball_x
LDA ball_y
LDB ball_v_y
AADDB
STA ball_y

// Do the next frame
JMP :frame

:end_point
// Output current scores
OUTL left_score
OUTR right_score

// Check whether either score is 9 or more
SETB 9
LDA left_score
ASUBB
JMP0 :done
LDA right_score
ASUBB
JMP0 :done
JMP :start_a_point
:done
HLT
```

A quick approximate calculation suggests this will be approximately 285 bytes
of code. I'm confident that with some optimisation, I can get that down to 256,
so it'll fit into an 8-bit addressable ROM. Adding a third register could also
help, to be used in higher level instruction.
For example, a new instruction `SUBA` where `SUBA 3` would use a third register for storing a
literal 3 (rather than having to do SETB 3; ASUBB), or adding an instruction
to subtract a value read from memory (to do `ASUBM 0x4` instead of
`LDB 0x4; ASUBB`).

A quick count suggests there are approximately 550 microinstructions required
on each frame, so I would need a 5.5kHz clock speed to get to 10 frames per
second. That's much lower than I was expecting, so it should be well within
the reach of all the components. Still it's something I'll bear in mind during
the build - I'll make sure I test each component at high-frequency if I can.

I don't want to have to re-input the program every time I turn the computer
off, so I'll put it on a ROM instead. In that case, it's OK for the RAM to be
very small - not a lot of data is being put there (at the moment). But it might
be that more RAM could be useful in making the program shorter or faster.
For that reason, I'm still going to expand the RAM, so that I have more
flexibility in future.

So here are all the changes I'm going to make to the original computer design:
* Add a third register
* 8-bit memory address register (instead of 4-bit)
* Full 8-bit RAM [Someone did that already](https://blog.straypaper.com/2017/12/02/replacing-74ls189-with-6116p/) (instead of 4-bits / 16 bytes)
* 8-bit program counter (instead of 4-bit)
* Use an EEPROM for the program itself
* Add a status register to allow for conditional jump instructions
* Expand the instruction set the computer recognises
* Add a display
* At some point I'll need some kind of input for the players to use, but I'm postponing that decision.

After I wrote the first draft of this post, Ben posted [a new video](https://www.youtube.com/watch?v=AqNDk_UJW4k) about making the computer Turing-complete, including a discussion on conditional jumps. So I suspect he'll be adding a status register himself before very long!

Feeling buoyed with confidence, I've now ordered everything.

Watch this space for more about the actual build.
