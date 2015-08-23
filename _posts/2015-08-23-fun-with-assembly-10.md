---
layout: post
title: 'Fun with Assembly 10: Overflowing the stack to mess with system calls'
date: 2015-08-25
status: publish
type: post
published: true
author: Hywel Carver
---

This is the tenth in a series. You might want to read the [previous post]({% post_url 2015-08-24-fun-with-assembly-9 %}) before reading this.

This post is based on the Whitehorse level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# First steps

The `login` function here is mercifully short. Let's read through it.

    44f4:  3150 f0ff      add   #0xfff0, sp
    44f8:  3f40 7044      mov   #0x4470 "Enter the password to continue.", r15
    44fc:  b012 9645      call  #0x4596 <puts>
    4500:  3f40 9044      mov   #0x4490 "Remember: passwords are between 8 and 16 characters.", r15
    4504:  b012 9645      call  #0x4596 <puts>

Increase the size of the stack by 16 bytes, output some strings.

    4508:  3e40 3000      mov   #0x30, r14
    450c:  0f41           mov   sp, r15
    450e:  b012 8645      call  #0x4586 <getsn>

Get input directly onto the stack, up to 48 bytes. That means our input can potentially be written beyond the current stack frame. Useful to know.

    4512:  0f41           mov  sp, r15
    4514:  b012 4644      call  #0x4446 <conditional_unlock_door>

This passes the input to the function `conditional_unlock_door` which then passes it to the lock, to check if it's the right function.

    4518:  0f93           tst r15
    451a:  0324           jz    #0x4522 <login+0x2e>
    451c:  3f40 c544      mov   #0x44c5 "Access granted.", r15
    4520:  023c           jmp   #0x4526 <login+0x32>
    4522:  3f40 d544      mov   #0x44d5 "That password is not correct.", r15
    4526:  b012 9645      call  #0x4596 <puts>
    452a:  3150 1000      add   #0x10, sp
    452e:  3041           ret

Outputs some text to tell the user whether the door opened or not, then deallocate 16 bytes from the stack and return.

# What happens next?

When the function returns, what will happen? Hopefully you'll remember from before that it will return to the calling function, whose address is stored in the next 2 bytes up along the stack. And those are 2 bytes that we could overwrite with the 17th and 18th bytes of the string we input.

We don't want to go to `conditional_unlock_door` - that function just passes the password elsewhere, so it's hard to exploit. The function it calls to do that is `INT` - the hardware interrupt. It passes an argument of `0x7e`, which tells the lock to open if the password is correct.

However, reading the manual, an argument of `0x7f` will simply open the door. So we can force our function to return to `0x4532`, so that the processor drops into the `INT` function. How do we pass in the `0x7f` argument?

Reading through the `INT` function, it looks 2 bytes beyond the stack pointer for its argument. So the input we need, altogether looks like:

`<16 bytes of anything><3245 (the address 0x4532 with bytes reversed)><4 bytes of anything><7f00>`

I used `00000000000000000000000000000000324500007f00`, and...

# The door springs open

This is a more complicated stack overflow than we've seen before - we had to drop straight into the interrupt function and also pass it an argument on the stack.