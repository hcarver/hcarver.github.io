---
layout: post
title: 'Fun with Assembly 13: Overflows and Underflows'
date: 2015-08-30
status: publish
type: post
published: true
author: Hywel Carver
---
This is the thirteenth in a series. You might want to read the [previous post]({% post_url 2015-08-26-fun-with-assembly-12 %}) before reading this.

This post is based on the Jakarta level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# First steps

You've seen a lot of assembly by now, and this level is quite similar to the previous one, so I'm going to give less detailed commentary on the code. Try to follow the instructions for yourself and work out what's going on.

Let's dive in to the `login` function. Here's the first and last few instructions of it.

    4560:  0b12           push  r11
    4562:  3150 deff      add   #0xffde, sp
    ...
    ...
    462e:  3150 2200      add   #0x22, sp
    4632:  3b41           pop   r11
    4634:  3041           ret

So there's `0x22` = 34 bytes allocated on the stack.

    4566:  3f40 8244      mov   #0x4482 "Authentication requires a username and password.", r15
    456a:  b012 c846      call  #0x46c8 <puts>
    456e:  3f40 b344      mov   #0x44b3 "Your username and password together may be no more than 32 characters.", r15
    4572:  b012 c846      call  #0x46c8 <puts>
    4576:  3f40 fa44      mov   #0x44fa "Please enter your username:", r15
    457a:  b012 c846      call  #0x46c8 <puts>

Output some stuff.

    457e:  3e40 ff00      mov   #0xff, r14
    4582:  3f40 0224      mov   #0x2402, r15
    4586:  b012 b846      call  #0x46b8 <getsn>
    458a:  3f40 0224      mov   #0x2402, r15
    458e:  b012 c846      call  #0x46c8 <puts>

Get 255 bytes of username, store it at `0x2402`, and then write it out.

    4592:  3f40 0124      mov   #0x2401, r15
    4596:  1f53           inc   r15
    4598:  cf93 0000      tst.b 0x0(r15)
    459c:  fc23           jnz   #0x4596 <login+0x36>

Increment `r15` until it points to the null byte at the end of the username.

    459e:  0b4f           mov   r15, r11
    45a0:  3b80 0224      sub   #0x2402, r11

Calculate the length of the username in bytes, in `r11`.

    45a4:  3e40 0224      mov   #0x2402, r14
    45a8:  0f41           mov   sp, r15
    45aa:  b012 f446      call  #0x46f4 <strcpy>

Copy the username onto the stack.

    45ae:  7b90 2100      cmp.b #0x21, r11
    45b2:  0628           jnc   #0x45c0 <login+0x60>
    45b4:  1f42 0024      mov   &0x2400, r15
    45b8:  b012 c846      call  #0x46c8 <puts>
    45bc:  3040 4244      br    #0x4442 <__stop_progExec__>

Compare the length with `0x21` = 33. If the length is greater than 33, print something out and end the program.

    45c0:  3f40 1645      mov   #0x4516 "Please enter your password:", r15
    45c4:  b012 c846      call  #0x46c8 <puts>

Write out some text.

    45c8:  3e40 1f00      mov   #0x1f, r14
    45cc:  0e8b           sub   r11, r14

Work out `0x1f` (=31) minus the length of the username, in `r14`

    45ce:  3ef0 ff01      and   #0x1ff, r14

Ensure that `r14` only has its bottom 9 bits set. That means its value will be no bigger than 511.

    45d2:  3f40 0224      mov   #0x2402, r15
    45d6:  b012 b846      call  #0x46b8 <getsn>
    45da:  3f40 0224      mov   #0x2402, r15
    45de:  b012 c846      call  #0x46c8 <puts>
    45e2:  3e40 0224      mov   #0x2402, r14
    45e6:  0f41           mov   sp, r15
    45e8:  0f5b           add   r11, r15
    45ea:  b012 f446      call  #0x46f4 <strcpy>

Get a password of that length, print it out, then copy it onto the stack, right after the password.

    45ee:  3f40 0124      mov   #0x2401, r15
    45f2:  1f53           inc   r15
    45f4:  cf93 0000      tst.b 0x0(r15)
    45f8:  fc23           jnz   #0x45f2 <login+0x92>
    45fa:  3f80 0224      sub   #0x2402, r15

Get the length of the password entered, stored in `r15`.

    45fe:  0f5b           add   r11, r15
    4600:  7f90 2100      cmp.b #0x21, r15
    4604:  0628           jnc   #0x4612 <login+0xb2>
    4606:  1f42 0024      mov   &0x2400, r15
    460a:  b012 c846      call  #0x46c8 <puts>
    460e:  3040 4244      br    #0x4442 <__stop_progExec__>

If more than 33 characters were entered for the username and password together, it prints something out then ends the program.

    4612:  0f41           mov   sp, r15
    4614:  b012 5844      call  #0x4458 <test_username_and_password_valid>
    4618:  0f93           tst   r15
    461a:  0524           jz    #0x4626 <login+0xc6>
    461c:  b012 4c44      call  #0x444c <unlock_door>
    4620:  3f40 3245      mov   #0x4532 "Access granted.", r15
    4624:  023c           jmp   #0x462a <login+0xca>
    4626:  3f40 4245      mov   #0x4542 "That password is not correct.", r15
    462a:  b012 c846      call  #0x46c8 <puts>

Test the username and password for correctness. If they're right, unlock the door. Otherwise, tell the user that the password is wrong.

# An idea

There are a few similar numbers in this code - maybe some of them are slightly wrong. There are **34** bytes on the stack, we're told our username and password must be at most **32** bytes in length. The username is then tested for being at most **33 bytes** long. The password must be below (**31** - username) bytes long. Then we check that the username and password together are less than **33** bytes long.

### Let's talk about overflow and underflow

A single byte can only hold values between 0 and 255. So how does a CPU processor cope when you tell it to do 255 + 1 and store the result in a single byte? Simple! It resets back to 0. So 255 + 5 = 4 (in a single byte).

You can imagine that the CPU adds the numbers together then ignores the bits that it doesn't have space to hold. E.g. in binary, 255 + 5 looks like this: `0b11111111 + 0b00000101`. That comes to `0b100000100`. But the CPU can only store the bottom 8 bits! So we get `0b00000100` which is 4.

The same happens with subtractions, except here it's called underflow. `0 - 1` will be computed as `0b00000000 - 0b00000001` which, in the same way as above, will come to `0b11111111`.

Why would you ever want this? Well sometimes it's useful! For example in this code, when we want to have 34 more bytes on the stack, we do `add   #0xffde, sp`. In a 2-byte register, this sum will overflow. The amount it will overflow can be found by `0x10000` - `0xffde` = `0x22` which is 34 bytes.

You can always check whether the previous sum has overflowed, as the carry bit will be set in the status register. The instructions `jc` and `jnc` will let you jump based on whether an operation has overflowed or not.

### Using overflows and underflows

What would happen if our username was 32 bytes long here? Its length would be less than 33 bytes, so that test would pass. The password would have to be 31 - 32 bytes, which will underflow to `0xffff`. The next line ensures it has a max of 511, so that's the limit of the password that we can enter.

When the length of the password is added to the length of the username, it needs to be less than 33 in the lowest byte. If the username is 32 bytes, and the password is 256 bytes, then the sum will be `0x120`. The code only looks in the lowest byte, which holds `0x20` which is indeed less than `0x21`.

To hack this lock, the username needs to be 32 characters (e.g. 'A' 32 times). The password needs 4 character of anything, to fill up the allocated stack space, and the 2 bytes for the stored copy of `r11`. The next 2 bytes will overwrite the return address for the function (I'm using `4c44`, so that the function returns into `unlock_door`), followed by 250 bytes of 'A' so that the total is 256.)

# The door springs open

This is a complicated problem. Hacking this lock requires a good understanding of how overflow and underflow work, in order to spot ways to exploit them. The key here was noticing that a few similar numbers were coming up in the code: 32, 33, 34. That suggested that the person writing the code had made a small error somewhere - and they had. Allowing the username to be up to 33 characters, and letting the password be 31 characters minus the username's length was the key to hacking the lock.