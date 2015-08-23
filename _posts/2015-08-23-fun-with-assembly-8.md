---
layout: post
title: "Fun with Assembly 8: Sanitising your users' input"
date: 2015-08-23
status: publish
type: post
published: true
author: Hywel Carver
---
This is the eighth in a series. You might want to read the [previous post]({% post_url 2015-08-05-fun-with-assembly-7 %}) before reading this.

This post is based on the Novosibirsk level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

# First glance

As usual, let's start with a quick scan of the assembly, to see if there's anything that stands out or looks different to the code from previous levels.

Unusually, this level isn't using `puts` to write output, it's using `printf`. Interestingly, it also outputs the username that gets typed in. Which means that what we typed in will be passed directly to `printf` at some point. In that case, it's probably worth looking at the manual's description of `printf`.

> Prints formatted output to the console. The string str is printed as in
puts except for conversion specifiers. Conversion specifiers begin with the %
character.

> - `s` The argument is of type char* and points to a string.
> - `x` The argument is an unsigned int to be printed in base 16.
> - `c` The argument is of type char to be printed as a character.
> - `n` The argument is of type unsigned int*. Saves the number of characters printed thus far. No output is produced.

If you haven't used `printf` before, it's a function for writing output. E.g. `printf("My favourite number is %x", 100)` will print "My favourite number is 100 onto the screen".

The unusual one here is `%n`, which saves the number of characters so far - this is the only one that lets us change things in memory. The other options are for letting us output things from memory.

# Where does the input go?

Let's look at how the string we input ends up being `printf`ed.

    4454:  3e40 f401      mov   #0x1f4, r14
    4458:  3f40 0024      mov   #0x2400, r15
    445c:  b012 8a45      call  #0x458a <getsn>

This is where we're allowed to enter our string, with a max length of 500 characters (`0x1f4`), being written to the address `0x2400`, before it's then copied onto the stack.

This then gets used by calling with `printf`.

    4474:  0f12           push  r15
    4476:  b012 c645      call  #0x45c6 <printf>

`printf` takes its arguments from the stack. At this point, the stack looks like this:

    <4-byte address of our input string> <multiple bytes of the string itself>

So if we use the `%n` formatter in the string, it will be written to the second argument to `printf`, which will be taken as the first 4 bytes of our string. Handy.

# What shall we change?

There must be lots of ways we could change the program so that it opens the door. The one I'm going to use is this: when the program calls `conditional_unlock_door`, it does 

    44c6:  3012 7e00      push  #0x7e
    44ca:  b012 3645      call  #0x4536 <INT>

The `0x7e` argument tells the system to unlock the door if the password is right. However, there's another argument `0x7f` that tells it to simply unlock the door. That would we be great for us. So what we want to do is to change the code at `0x44c8` to have the value `0x7f`.

To do that, we need a string that begins `c844` (so that the address `0x44c8` gets written to). Then it needs to write `0x7f` (127) characters total before a `%n` to make sure that the right value gets written to the write place.

So in total, we need:

`c844<any old character repeated 125 times>256e` 

(because `%` and `n` have hex values `25` and `6e`).

# And the door opens...

You can't assume the input you get from your users is clean and safe to use. If you pass user input straight to your SQL database, a malicious user could access or delete your data. If you pass user input straight one to your webpage, a malicious user could write scripts that run when other users view the page. And if you pass user input straight into your `printf` function, they could open the lock to your front door.