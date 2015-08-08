---
layout: post
title: 'Fun with Assembly 5: Buffer Overflows to reach the parts other code can''t
  reach'
date: 2015-08-05 20:40:13.000000000 +01:00
categories: []
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
This is the fifth in a series. You might want to read the
[previous post]({% post_url 2015-07-12-fun-with-assembly-4 %})
before reading this.

This post is based on the Hanoi level on <a href="https://microcorruption.com">microcorruption.com</a>. Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

<h1>First glance</h1>
As usual, I'm going to start by looking through the <code>login</code> function.

<pre><code>4520:  c243 1024      mov.b #0x0, &amp;0x2410
4524:  3f40 7e44      mov   #0x447e "Enter the password to continue.", r15
4528:  b012 de45      call  #0x45de &lt;puts&gt;
452c:  3f40 9e44      mov   #0x449e "Remember: passwords are between 8 and 16 characters.", r15
4530:  b012 de45      call  #0x45de &lt;puts&gt;
</code></pre>
The first line here sets the value at <code>0x2410</code> to 0 - but it's hard to say why without reading more. The others write out some text. Reading on...

<pre><code>4534:  3e40 1c00      mov   #0x1c, r14
4538:  3f40 0024      mov   #0x2400, r15
453c:  b012 ce45      call  #0x45ce &lt;getsn&gt;
</code></pre>
This should look familiar too. The parameters for <code>getsn</code> are the max number of characters to get, and the place to store the result. Here, they are <code>0x1c</code> = 28 characters and the location <code>0x2400</code>.

This already looks like a good place to start. The message says that passwords can't be longer than 16 characters but the code actually accepts 28. Where do the extra characters go? They're going to start at <code>0x2410</code>, a number which sounds familiar. It was set to 0 at the very start of the function.

Looking further down, that number appears again:

<pre><code>455a:  f290 1a00 1024 cmp.b #0x1a, &amp;0x2410
4560:  0720           jne   #0x4570 &lt;login+0x50&gt;
4562:  3f40 f144      mov   #0x44f1 "Access granted.", r15
4566:  b012 de45      call  #0x45de &lt;puts&gt;
456a:  b012 4844      call  #0x4448 &lt;unlock_door&gt;
456e:  3041           ret
</code></pre>
First the code checks <code>0x2410</code> has the value <code>0x1a</code>. If it isn't equal, we jump to a later point in the function. If it is equal, we carry on: the text "Access granted" is printed, then the function to unlock the door is called.

We can use a buffer overflow here to put the value <code>0x1a</code> in <code>0x2410</code>. The code allows for 28 bytes to be input from <code>0x2400</code>, so 16 bytes of nothing followed by the <code>1a</code> byte should do the trick.

I used <code>0102030405060708090a0b0c0d0e0f101a</code> for my final input.

<h1>Endnotes</h1>
This was a straightforward example: the code very clearly expected 16 characters for input, but allowed room for many more. It's actually very easy to leave room for buffer overflows in real-world code, but they're normally harder to spot. The easiest way to find them is often to try 'fuzzing' with random long strings of input, to see if they make a program crash. If a program does crash, clearly something was unanticipated about the input you gave it, which could let you find a buffer overflow.

