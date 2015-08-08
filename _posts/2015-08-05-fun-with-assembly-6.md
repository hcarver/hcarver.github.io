---
layout: post
title: 'Fun with Assembly 6: More buffer overflows to change return values'
date: 2015-08-05 21:09:21.000000000 +01:00
categories: []
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
This is the sixth in a series. You might want to read the
[previous post]({% post_url 2015-08-05-fun-with-assembly-5 %})
before reading this.

This post is based on the Cusco level on <a href="http://microcorruption.com">microcorruption.com</a>. Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

<h1>First glance</h1>
You should know the drill by now. Let's look at the <code>login</code> function.

<pre><code>4500:  3150 f0ff      add   #0xfff0, sp
4504:  3f40 7c44      mov   #0x447c "Enter the password to continue.", r15
4508:  b012 a645      call  #0x45a6 &lt;puts&gt;
450c:  3f40 9c44      mov   #0x449c "Remember: passwords are between 8 and 16 characters.", r15
4510:  b012 a645      call  #0x45a6 &lt;puts&gt;
</code></pre>
Adding <code>0xfff0</code> to the stack pointer decreases it by 16. That means adding 16 bytes to the stack for use by the current frame. Then the code writes out some text.

<pre><code>4514:  3e40 3000      mov   #0x30, r14
4518:  0f41           mov   sp, r15
451a:  b012 9645      call  #0x4596 &lt;getsn&gt;
</code></pre>
This should look familiar too - familiar code with a familiar bug. The code gets input by calling <code>getsn</code>, storing the result at the address of the stack pointer, and allowing up to 0x30 = 48 characters. Which is 32 more bytes than was just added to the stack frame to allow for it.

We've already seen in a previous episode that the first two bytes (on a 16-bit CPU) beyond the current stack frame are where the return address is stored. When <code>ret</code> is called from a stack frame, it will be those two bytes that the CPU returns to.

And it's those two bytes that will be the first we overwrite with a password beyond 16 characters. What address would we like the function to return to? Well, <code>unlock_door</code> at <code>0x 4446</code> sounds like a good candidate. We have to change the order of the bytes because addresses are stored in a <a href="https://en.wikipedia.org/wiki/Endianness#Little-endian">little-endian</a> way. So we just need an input with <code>0x4644</code> in the 17th and 18th bytes. I used <code>0x0102030405060708090a0b0c0d0e0f104644</code>.

<h1>Endnotes</h1>
Lots of this was similar to code we've seen before. Buffer overflows from an input that wasn't correctly sanitised, and overwriting return values. Next time: the myth of security through obscurity.

