---
layout: post
title: 'Building an 8-bit computer for Pong'
date: 2017-01-03
status: publish
type: post
published: true
author: Hywel Carver
---

I've been inspired to build an 8-bit computer that lets you play Pong. Why? Building my own computer that I understand down to the transistor will be super-interesting and educational for me, and Pong is the minimal program I'd be likely to actually use (which would also be super-fun). If I can, I'll also make it possible to play other games.

![A homemade 8-bit computer]({{ "/assets/ben-eater.jpg" }})

This series of posts will chart my progress as I go.

## Standing on the Shoulders of Giants

[This fantastic series of videos](https://www.youtube.com/watch?v=HyznrdDSSGM&index=1&list=PLowKtXNTBypGqImE405J2565dvjafglHU) by [Ben Eater](https://twitter.com/ben_eater?lang=en) will be my starting point. Go watch those videos now - Ben walks you through the creation of a fully functional 8-bit computer (RAM, registers, program counter, ALU, control logic, bus, right down to the clock), and explains everything down to the level of individual transistors. Probably the best thing about it is listening to how Ben reasons about the operation of the computer and its individual components.

They really are *excellent* videos and I'll be working through them to build my computer. However there are a whole set of changes that'll be needed to build something capable of running Pong.

## The end and the beginning

Pong is a 2-player game, where each player controls a "paddle" which can be moved up and down, one on each side of the screen. A ball bounces around the screen between the walls at the top and bottom, and the players' paddles. If the ball hits the left or right side, a player scores a point.

The original game looks like this, and I'm going to make something similar.
![Pong]({{ "/assets/pong.png" }})

Here's what Ben's 8-bit computer project provides:
* A clock to set the pace for the whole computer (which can be advanced manually for debugging)
* A bus to connect components together
* Two 8-bit data registers
* One 8-bit instruction register
* An 8-bit ALU that can add or subtract in twos complement, between the two registers
* 16 bytes of RAM (addressed with a 4-bit register), shared between code & data
* A program counter
* A 7-segment display output
* Control logic to map instructions to control signals (using 4 bits for instruction and 4 bits for any arguments)
* Instructions for load-from-mem-into-reg-a, store-from-reg-a-to-mem, put-value-into-reg-a, add-from-mem-to-reg-a, subtract-from-reg-a-with-mem, output-to-7-seg-display, halt, jump
* A reset button.

As well as some bonus useful extras:
* An EEPROM programmer (with an Arduino Nano) for writing data efficiently
* A 4-bit address switch and an 8-bit data switch to hand-program the values in RAM
* A switch to swap memory between "programmable" and "running" modes.

## My plan

Ben's project gets us a lot of the way there, but there are still going to need to be some changes to the computer. Without planning everything, there are some design factors I can already plan.

* I'll implement the game entirely with integer arithmetic (no floating points) to keep things simple.
* I'll need a screen output as well as the 7-segment display, which I'll keep for displaying the score. [Something like this](https://uk.rs-online.com/web/p/lcd-monochrome-displays/7588721/) would make sense - given that most of the screen will be off, and only a few pixels are 'on' (the paddles, the ball). I haven't thought too hard about the responsiveness / practicalities of the screen yet - that will come later. It might be possible to use an OLED display [like this.](https://uk.rs-online.com/web/p/oled-displays/8235926/).
* The game speed will be determined by the clock, so that I don't have to deal with any real-world times. That'll mean
  * Ensuring that all paths through the code are the same number of microinstructions (so that every tick of the game is the same real-world length)
  * I'll need a faster clock speed than Ben uses, and I'll need to check all the components are capable of working at that speed.

Some unknowns that need resolving:
* Ben's computer uses 4 bits of the instruction register for instructions, and 4 for arguments / addresses. That might not be sufficient for Pong, which might need more than 16 bytes of RAM (and more than 4 bits for addressing). Or we might have data in RAM, separate from the program code in ROM (even 8-bit-addressable RAM would be 256 bytes, which might not be enough for the Pong code).
* We might also need more than 16 instructions (so more than 4 bits for the instruction register).
* Ben's ALU only does signed arithmetic (with numbers in the range of -128 to 127). If the display I use is 256 pixels, I might need to use numbers between 128 and 256, which would need some unsigned arithmetic.
* There's currently no overflow detection, which may be necessary.
* It'd be nice to have some random number generation so that not every game is the same.
* The RAM used might be too slow to support the clock speed necessary. That's worth investigating.
* The computer is programmed via hand, which I can't really imagine doing for an entire pong game, so I might use an EEPROM for the program itself, and program it via Arduino.
* We'll need some kind of input device for each user. Ideally this would be usable for other games.

So my plan is:
1. Decide on a display to use.
1. Decide the layout of RAM, and write the code for the whole game.
1. Work out the clock speed we need to get to 10 fps (say).
1. Find out the maximum clock speed supported by the existing ICs used.
1. Work out the other unknowns.
1. Work out the other changes that need making.
1. Build it.

Watch this space for more.
