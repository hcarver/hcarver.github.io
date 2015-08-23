---
layout: post
title: 'Fun with Assembly 9: Seriously though, sanitise your inputs'
date: 2015-08-24
status: publish
type: post
published: true
author: Hywel Carver
---
This is the ninth in a series. You might want to read the [previous post]({% post_url 2015-08-23-fun-with-assembly-8 %}) before reading this.

This post is based on the Addis Ababa level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# First steps

As always, I'm starting with a quick read through the code to look for anything unusual compared to other levels. Firstly, we get up to 19 characters of input, and store it at `0x2400`.

    4454:  3e40 1300      mov   #0x13, r14
    4458:  3f40 0024      mov   #0x2400, r15
    445c:  b012 8c45      call  #0x458c <getsn>

19 isn't very many. The input is then copied onto the stack, two bytes down from the top of the stack.

    4460:  0b41           mov   sp, r11
    4462:  2b53           incd  r11
    4464:  3e40 0024      mov   #0x2400, r14
    4468:  0f4b           mov   r11, r15
    446a:  b012 de46      call  #0x46de <strcpy>

At this point, the code runs `test_password_valid` to check the password's right. I'm going to ignore that, because the password we enter will be wrong.Next, the result of that is written onto the stack before the code calls `printf` with the password that was entered. 

    4476:  814f 0000      mov r15, 0x0(sp)
    447a:  0b12           push  r11
    447c:  b012 c845      call  #0x45c8 <printf>

Just like last time, there's no sanitising of the string that we can input. That means we can exploit `printf` in the same way as before. Now we just need to work out what we want to change. 

Later on, that return value is read to decide whether or not to open the door.

    448a:  8193 0000      tst   0x0(sp)
    448e:  0324           jz    #0x4496 <main+0x5e>
    4490:  b012 da44      call  #0x44da <unlock_door>

So, let's use `printf` to overwrite the return value with anything that isn't zero. By inserting a breakpoint on these lines, you can step through the code and see that the return value is being written to `0x303c` (the location of the stack pointer at that moment).

When `printf` is called with the input from the user, the stack looks like this: 

    303a: 3e30 (the address of the input string)
    303c: 0000 (the return value that we want to overwrite)
    303e: ... (the input string itself)

`printf` will see its arguments as the address of the string, the value 0, then the first two bytes of the input string. The string should start `3c30` (remember that addresses are stored with the byte order switched), because this will be the location that printf eventually writes something to. 

Then we need to add something for the function to consume the 0000, e.g. a `%i` formatter that will output an integer (`2569` in hex). Finally, we want a `%n`(`256e` in hex), which will make `printf` write out the number of characters it has written so far to the next address on the stack (which we've manipulated so that it will overwrite the return value from `check_password_valid`).

`3c302569256e`

# And the door springs open

As before, this is what happens when you take input from users then use it without sanitisation in sensitive contexts (printing functions, SQL, HTML). The best case is that they can crash your program, but [the worst case is that they can hack into your data](https://xkcd.com/327/), or open your locks.