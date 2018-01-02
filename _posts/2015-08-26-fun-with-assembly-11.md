---
layout: post
title: 'Fun with Assembly 11: More smashing through the stack'
date: 2015-08-26
status: publish
type: post
published: true
author: Hywel Carver
---

This is the eleventh in a series. You might want to read the [previous post]({% post_url 2015-08-23-fun-with-assembly-10 %}) before reading this.

This post is based on the Montevideo level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# A suggestion

**Stop reading**. This one works very similarly to the previous level (see link above) - go and re-read that, then try really hard to solve this level yourself, before you read anything below. The one thing you might need to know is that `strcpy` will stop copying when it reaches the first null byte (`0x00`).

# First steps

This looks so similar to the last level (see link above). What's different? Well this time, the input isn't directly taken onto the stack. It's stored at `0x2400`.

    4508:  3e40 3000      mov   #0x30, r14
    450c:  3f40 0024      mov   #0x2400, r15
    4510:  b012 a045      call  #0x45a0 <getsn>

It's then copied to the stack (which again has only 16 bytes added for 48 potential bytes of input).

    4514:  3e40 0024      mov   #0x2400, r14
    4518:  0f41           mov   sp, r15
    451a:  b012 dc45      call  #0x45dc <strcpy>

Then the area where the input was written is reset.

    451e:  3d40 6400      mov   #0x64, r13
    4522:  0e43           clr   r14
    4524:  3f40 0024      mov   #0x2400, r15

The difference this makes is all in the string copying. The `strcpy` will copy up to the first null byte (`0x00`) and then stop. That means we can't have any null bytes in our input string if we want it to work. The address of the `INT` function has also changed, to `0x454c`.

Our answer before was `00000000000000000000000000000000324500007f`. Changing for the new address of `INT` gives us `000000000000000000000000000000004c4500007f`. If we now change every 0 to a 4 (to avoid the problematic null bytes), we end up with `444444444444444444444444444444444c4544447f`, and...

# The door springs open

This is really a variation on a theme. We're applying ideas you've already seen before, with the extra constraint that we can't have a null byte in the middle of our input.
