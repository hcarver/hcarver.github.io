---
layout: post
title: 'Fun with Assembly 4: Breaking canaries, boomerangs and stacks'
date: 2015-07-12 18:30:43.000000000 +01:00
categories: []
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
This is the fourth in a series. You might want to read the
[previous post]({% post_url 2015-02-12-fun-with-assembly-3 %})
before reading this.

This post is based on the Johannesburg level on <a href="http://www.microcorruption.com">microcorruption.com</a>. Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

<h2>Let's look at the <code>login</code> function</h2>
<pre><code>452c:  3150 eeff      add   #0xffee, sp
4530:  f140 a200 1100 mov.b #0xa2, 0x11(sp)
4536:  3f40 7c44      mov   #0x447c "Enter the password to continue.", r15
453a:  b012 f845      call  #0x45f8 &lt;puts&gt;
453e:  3f40 9c44      mov   #0x449c "Remember: passwords are between 8 and 16 characters.", r15
4542:  b012 f845      call  #0x45f8 &lt;puts&gt;
4546:  3e40 3f00      mov   #0x3f, r14
454a:  3f40 0024      mov   #0x2400, r15
454e:  b012 e845      call  #0x45e8 &lt;getsn&gt;
</code></pre>
First up, we <code>add #0xffee, sp</code> which (as I've explained before) subtracts 17 from the stack pointer, increasing the size of the stack. Then we set the highest byte in this stack frame to <code>0xa2</code> (but it's not obvious why). And then the assembly outputs a  couple of messages, including one reminding the user that password are between 8 and 16 letters longing.

Next, we call <code>getsn</code> to get <code>#0x3f</code> (= 63 in decimal) characters of input, and store it at <code>#0x2400</code>. This looks like a great place to start actually because the program tells the user it'll accept 16 characters of password, but actually accepts 63.

Let's see what happens if we just chuck in a long input. I'm going to use <code>0x010203...63</code>, because it makes it easy to see where each character of your input ends up.

<h2>Canaries are idiots</h2>
Output on the console: <code>Invalid Password Length: password too long.</code>

That's weird - it must be doing some kind of checks on the length of the password after all. Let's look at where that error message is output:

<pre><code>4578:  f190 a200 1100 cmp.b #0xa2, 0x11(sp)
457e:  0624           jeq   #0x458c &lt;login+0x60&gt;
4580:  3f40 ff44      mov   #0x44ff "Invalid Password Length: password too long.", r15
</code></pre>
Ah - so that value of <code>0xa2</code> is being checked for later. After the password attempt is received, it gets copied into the stack - then the program checks the magic value of <code>0xa2</code> hasn't been changed. If it has been changed, the password was too long and shouldn't be allowed.

This is commonly called a canary - you set a value, then check it's still the same later to make sure everything's OK. <a href="https://en.wikipedia.org/wiki/Domestic_canary#Miner.27s_canary">It's just like a miner, setting hir pet canary to alive, then repeatedly checking to make sure it hasn't become dead</a>.

Canaries are idiots and easily fooled. If we had the right byte of our input set to <code>0xa2</code>, then we'd be overwriting the canary with the value that it already had, and we'd be able to overwrite other values on the stack too, without it being detected by the canary system. Those other values on the stack might be important.

This is how the program thinks the stack is at the moment.

<pre><code>&lt;- up to 16 bytes of input string, copied from 0x2400 -&gt;
&lt;- 1 canary byte of 0xa2 -&gt;
&lt;- 1 unused byte to round up to a multiple of 2 -&gt;
&lt;- first 2 bytes beyond the current stack frame -&gt;
</code></pre>
And those first 2 bytes beyond the current stack frame are where CPUs store the address to return to, which is hugely helpful.

<h2>How returns work in assembly</h2>
If you've worked in most other programming languages, you'll have heard people talk about the call stack - function A calls function B calls function C etc, creating  a stack of function calls. Function A doesn't continue until function B finishes, and function B won't continue until function C is finished.

That kind of nested structure isn't how memory works. Remember, memory is a long linear string of bytes. So in memory, you have to store the address that needs to be returned to. When function A calls function B, the current value of the program counter (which points to the next line of assembly to run), is pushed onto the stack, and the program counter changes to point to the first line of function B.

Function B might increase the size of the stack, do some calculations, then decrease the size of the stack again. When it returns, it pops the next values on the stack into the program counter, so that execution carries on from where it left of in function A.

So that important return value is always stored just outside the stack frame of the function that's currently executing.

<h2>Broken boomerang</h2>
Assembly returns are meant to return to exactly where they left off, like a boomerang. But if we can change the stored return address, we can change what the program will do after the function finishes, and break that boomerang.

Remember, this is how the stack looks to the program:

<pre><code>&lt;- up to 16 bytes of input string, copied from 0x2400 -&gt;
&lt;- 1 canary byte of 0xa2 -&gt;
&lt;- 1 unused byte to round up to a multiple of 2 -&gt;
&lt;- first 2 bytes beyond the current stack frame -&gt;
</code></pre>
So if we input: 16 bytes of anything, 2 bytes of <code>0xa2a2</code>, then 2 other bytes, when the function finishes it will return to the location of those 2 bytes at the end, which we can choose.

There's a function called <code>unlock_door</code> at <code>0x4446</code>, which sounds like a good bet. Remembering that byte order gets reversed within addresses, you end up with an input of <code>0x01020304050607080910111213141516a2a24644</code>. Entering that as the input will unlock the door.

<h2>Endnotes</h2>
This all started from noticing that the program allowed more bytes of input than it advertised - once we'd seen that, we could just try a really long input, and see what happened next. We found a canary system, but it didn't really get in the way once we understood it. It's much safer to prevent bad input sooner rather than later - this hack wouldn't have worked if the program had immediately rejected any character in the password after the 16th.

