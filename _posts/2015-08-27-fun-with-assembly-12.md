---
layout: post
title: 'Fun with Assembly 12: A bigger challenge'
date: 2015-08-26
status: publish
type: post
published: true
author: Hywel Carver
---

This is the twelfth in a series. You might want to read the [previous post]({% post_url 2015-08-26-fun-with-assembly-11 %}) before reading this.

This post is based on the Santa Cruz level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# First steps

Wow, this is the most complicated level yet. It also takes both a username and a password. The `main` function adds 50 bytes to the stack, then calls `login`. `login` is a huge function that contains all of the logic itself, until it eventually calls `unlock_door`.

So let's break down the whole `login` function and work out what it's doing.

### Topping and Tailing

The first and last instructions in `login` relate to each other, so it helps to look at them together.

    4550:  0b12           push  r11
    4552:  0412           push  r4
    ...
    4662:  3441           pop   r4
    4664:  3b41           pop   r11
    4666:  3041           ret

This pushes the current values of registers 11 and 4 onto the stack. At the end of the function it pops them off the stack again. That's because this function uses `r4` and `r11`, so will change their values. But other functions up the stack might be already using `r4` and `r11`, and we have to be careful not to change their values between the function being called and returning. So right now the bottom of the stack looks like this:

    From top of stack: 3  2 | 1  0  | -1 -2
                       <r4> | <r11> | <stored return address>

Then:

    4554:  0441           mov   sp, r4
    4556:  2452           add   #0x4, r4
    4558:  3150 d8ff      add   #0xffd8, sp
    455c:  c443 faff      mov.b #0x0, -0x6(r4)
    4560:  f442 e7ff      mov.b #0x8, -0x19(r4)
    4564:  f440 1000 e8ff mov.b #0x10, -0x18(r4)
    ...
    465e:  3150 2800      add   #0x28, sp

Register 4 is set to 4 bytes above the bottom of the stack, which is currently equal to the top of the stack. The stack pointer is decremented by `0x28` = 40 bytes. We then set some specific byte values in the stack. Right at the end of the `login` function, we'll then remove the extra space that was added to the stack. But for the middle part of the function we have (from the top of the stack down):

    4 bytes of stored return address (r4 points to first byte)
    --------------
    2 bytes of r11
    2 bytes of r4
    1 blank byte
    1 byte set to 0x0
    17 blank bytes
    1 byte set to 0x10
    1 byte set to 0x8
    19 blank bytes (sp points to first byte)

Hopefully we'll work out why soon.

### Getting going

We're onto the middle bit of the function now.

    456a:  3f40 8444      mov   #0x4484 "Authentication now requires a username and password.", r15
    456e:  b012 2847      call  #0x4728 <puts>
    4572:  3f40 b944      mov   #0x44b9 "Remember: both are between 8 and 16 characters.", r15
    4576:  b012 2847      call  #0x4728 <puts>
    457a:  3f40 e944      mov   #0x44e9 "Please enter your username:", r15
    457e:  b012 2847      call  #0x4728 <puts>

We print out some info for the user. Apparently we accept usernames and passwords between 8 and 16 characters long.

    4582:  3e40 6300      mov   #0x63, r14
    4586:  3f40 0424      mov   #0x2404, r15
    458a:  b012 1847      call  #0x4718 <getsn>
    458e:  3f40 0424      mov   #0x2404, r15
    4592:  b012 2847      call  #0x4728 <puts>

Nope! We actually accept `0x63` = 100 characters of input for the username (at `0x2404`. Then we write it out to the console.

    4596:  3e40 0424      mov   #0x2404, r14
    459a:  0f44           mov   r4, r15
    459c:  3f50 d6ff      add   #0xffd6, r15
    45a0:  b012 5447      call  #0x4754 <strcpy>

Then we copy it onto the stack, `0x2a` = 42 bytes behind `r4`. So, our stack now looks like this (with `*` meaning bytes that we can overwrite with our username input).

    *4 bytes of stored return address (r4 points to first byte)
    --------------
    *2 bytes of the calling function's r11
    *2 bytes of the calling function's r4
    *1 blank byte
    *1 byte set to 0x0
    *17 blank bytes
    *1 byte set to 0x10
    *1 byte set to 0x8
    *17 bytes where the username is intended to go.
    2 blank bytes (sp points to first byte)

Interesting. What's next?

    45a4:  3f40 0545      mov   #0x4505 "Please enter your password:", r15
    45a8:  b012 2847      call  #0x4728 <puts>
    45ac:  3e40 6300      mov   #0x63, r14
    45b0:  3f40 0424      mov   #0x2404, r15
    45b4:  b012 1847      call  #0x4718 <getsn>
    45b8:  3f40 0424      mov   #0x2404, r15
    45bc:  b012 2847      call  #0x4728 <puts>

We get asked for our password, which can also be `0x63` = 99 bytes. That then gets written back out for the user to see.

    45c0:  0b44           mov   r4, r11
    45c2:  3b50 e9ff      add   #0xffe9, r11
    45c6:  3e40 0424      mov   #0x2404, r14
    45ca:  0f4b           mov   r11, r15
    45cc:  b012 5447      call  #0x4754 <strcpy>

And now we copy it to the stack, at a position `0x17` = 23 bytes behind `r4`. So now the stack looks this (* = we can write to it with our username, & = we can write to it with our password):

    *&4 bytes of stored return address (r4 points to first byte)
    --------------
    *&2 bytes of the calling function's r11
    *&2 bytes of the calling function's r4
    *&1 blank byte
    *&1 byte set to 0x0
    *17 bytes where the password is intended to go
    *1 byte set to 0x10
    *1 byte set to 0x8
    *17 bytes where the username is intended to go.
    2 blank bytes (sp points to first byte)

Well that's interesting. There's lots of bytes we can write to, and some of them we can write to twice.

    45d0:  0f4b           mov   r11, r15
    45d2:  0e44           mov   r4, r14
    45d4:  3e50 e8ff      add   #0xffe8, r14
    45d8:  1e53           inc   r14
    45da:  ce93 0000      tst.b 0x0(r14)
    45de:  fc23           jnz   #0x45d8 <login+0x88>

We copy the contents of `r11` (which is pointing at the password on the stack) into `r15`, which must be useful later because it's not used here. Then we make `r14` point to `r4 - 0x18 + 1`, which will be 17 bytes behind `r4`, which is where the password is stored on the stack.

The last 3 instructions are interesting. Once `r14` points to the start of the password, we see if that byte is `0x0` (i.e. whether we've reached the end of the string). If the byte isn't NULL, we got back 2 instructions and increment `r14`, then we test again. The end result is the `r14` points to the `0x0` byte that marks the end of the string.

    45e0:  0b4e           mov   r14, r11
    45e2:  0b8f           sub   r15, r11
    45e4:  5f44 e8ff      mov.b -0x18(r4), r15
    45e8:  8f11           sxt   r15
    45ea:  0b9f           cmp   r15, r11
    45ec:  0628           jnc   #0x45fa <login+0xaa>

`r11` now points to the NULL byte at the end of the password. Then we subtract `r15` which is the first byte of the password. So now `r11` is set to the length of the password in bytes.

Then the value `0x18` = 24 bytes down from r4 (which was initialized to `0x10`) is copied into `r15`, then in sign-extended (`sxt r15` so that it fills the whole 2 bytes of the register). Now we compare that value with r11. If `r15` is bigger than `r11` (i.e. the value that was initialized to `0x10` is bigger than the length of the password), then we jump over the next section.

    45ee:  1f42 0024      mov   &0x2400, r15
    45f2:  b012 2847      call  #0x4728 <puts>
    45f6:  3040 4044      br    #0x4440 <__stop_progExec__>

And the next section writes some output then kills the program. So if we want this to be avoided, **we need to make sure that the value `0x18` bytes down from `r4` is bigger than the length of the password**.

    45fa:  5f44 e7ff      mov.b -0x19(r4), r15
    45fe:  8f11           sxt   r15
    4600:  0b9f           cmp   r15, r11
    4602:  062c           jc    #0x4610 <login+0xc0>

Now the byte `0x19` = 25 bytes down from `r4` (which was initialized to `0x8`) is copied into `r15`, then extended to fill the whole byte. That gets compared with `r11` (which is still the length of the password). If the password's length is bigger than the value, we jump the next section.

    4604:  1f42 0224      mov   &0x2402, r15
    4608:  b012 2847      call  #0x4728 <puts>
    460c:  3040 4044      br    #0x4440 <__stop_progExec__>

And the next section writes some output then stops the program. So if we want that to be avoided, **we need to make sure that the value `0x19` bytes down from `r4` is less than the length of the password**.

    4610:  c443 d4ff      mov.b #0x0, -0x2c(r4)
    4614:  3f40 d4ff      mov   #0xffd4, r15
    4618:  0f54           add   r4, r15
    461a:  0f12           push  r15
    461c:  0f44           mov   r4, r15
    461e:  3f50 e9ff      add   #0xffe9, r15
    4622:  0f12           push  r15
    4624:  3f50 edff      add   #0xffed, r15
    4628:  0f12           push  r15
    462a:  3012 7d00      push  #0x7d
    462e:  b012 c446      call  #0x46c4 <INT>
    4632:  3152           add   #0x8, sp

First of all, this sets a 0 byte, 44 bytes down from `r4`, so that our stack looks like this:

    *&4 bytes of stored return address (r4 points to first byte)
    --------------
    *&2 bytes of the calling function's r11
    *&2 bytes of the calling function's r4
    *&1 blank byte
    *&1 byte set to 0x0
    *17 bytes where the password is intended to go
    *1 byte set to 0x10
    *1 byte set to 0x8
    *17 bytes where the username is intended to go.
    1 blank bytes 
    1 byte set to 0x0 (sp points here)

Then `r15` is set to point 44 bytes below `r4`, which is the address that has just been set to `0x0`. That address is pushed to the stack, followed by the address 23 bytes below `r4` (which is where the password is stored), followed by the address 19 bytes below that (which is where the username is stored). Then `INT`, the interrupt function, is called with a first argument of `0x7d`.

Looking through the manual, this interfaces with the lock to set a flag in memory if the username and password are correct. Judging by what's been pushed to the stack, that flag is 44 bytes down the stack from `r4`.

The last instruction decreases the stack by 8 bytes, to compensate for the 4 2-byte arguments that were pushed for the call to interrupt.

    4634:  c493 d4ff      tst.b -0x2c(r4)
    4638:  0524           jz    #0x4644 <login+0xf4>
    463a:  b012 4a44      call  #0x444a <unlock_door>
    463e:  3f40 2145      mov   #0x4521 "Access granted.", r15
    4642:  023c           jmp   #0x4648 <login+0xf8>
    4644:  3f40 3145      mov   #0x4531 "That password is not correct.", r15
    4648:  b012 2847      call  #0x4728 <puts>

This part of the programme tests the magic byte at the bottom of the stack, 44 bytes below `r4`. If it's zero, we jump to printing that the password isn't correct. But if it's non-zero, we call the function to unlock the door, and print a message saying that access has been granted.

    464c:  c493 faff      tst.b -0x6(r4)
    4650:  0624           jz    #0x465e <login+0x10e>
    4652:  1f42 0024      mov   &0x2400, r15
    4656:  b012 2847      call  #0x4728 <puts>
    465a:  3040 4044      br    #0x4440 <__stop_progExec__>

The last few instructions now. Firstly, we test 6 bytes down from `r4` (the byte which was initialised to be zero). If it's zero, we jump to the return section of the function. If not, we print something out then halt the program. So **if that byte is non-zero, we won't do a normal return from the `login` function**.

### Exploit

Let's look at that final copy of the stack.

    *&4 bytes of stored return address (r4 points to first byte)
    --------------
    *&2 bytes of the calling function's r11
    *&2 bytes of the calling function's r4
    *&1 blank byte
    *&1 byte set to 0x0
    *17 bytes where the password is intended to go
    *1 byte set to 0x10
    *1 byte set to 0x8
    *17 bytes where the username is intended to go.
    1 blank bytes 
    1 byte set to 0x0 (sp points here)

One of the areas we can update is the return address. If we could rewrite that with the address `0x444a`, then when `login` returns it would drop into the `unlock_door` function (which exists at `0x444a`).

But we have to be careful - for the function to return, we have to make sure that canary 6 bytes below `r4` is still `0x0`. And we need to make sure the password's length is between the values in the bytes 25 and 24 bytes below `r4`.

Using `0x0` as the canary actually create some problems for us. Because strings are terminated with a null byte, you can't have `0x0` in the middle of a string. It has to be at the very end.

Let's say we used the password to overwrite the return address. We can't do that *and* get the canary byte to be 0. And we've already entered the username by this point, so we have no way of changing it after the password's entered.

But if we write in a long password, we can overwrite the return address, then use password to reset the canary byte to 0. To do that, we'll need a password that's exactly 17 characters long, which will then be saved with a null byte where the canary is.

Password (bytes): `1010101010101010101010101010101010`

The user name is more complicated. Look at the stack again (copied below for reference). We'll need 17 bytes to fill the space (`0x1010101010101010101010101010101010`), then one byte that's less than the length of the password (e.g. `0x01`, then one that's larger than the length of the password (e.g. `0xff`). Then 23 bytes of filler to get through the password storage, canary byte + 5 other bytes (`0x1010101010101010101010101010101010101010101010`). Remember, this is OK because we're going to write over the canary byte again when we enter the password.

Then we need the bytes we want to be our new return address. I'm going to use `0x4a44`, so that we return to the `unlock_door` function.

    *&4 bytes of stored return address (r4 points to first byte)
    --------------
    *&2 bytes of the calling function's r11
    *&2 bytes of the calling function's r4
    *&1 blank byte
    *&1 byte set to 0x0
    *17 bytes where the password is intended to go
    *1 byte set to 0x10
    *1 byte set to 0x8
    *17 bytes where the username is intended to go.
    1 blank bytes 
    1 byte set to 0x0 (sp points here)

That gives a username of (bytes) `101010101010101010101010101010101001ff10101010101010101010101010101010101010101010104a44`

# The door springs open

This is the most tricky level yet. Having two inputs complicated the exploit, but the hardest part was just working out what every instruction was doing. Once we had that, and could map out what was meant to be on the stack, it was pretty straightforward to see how it could be exploited.