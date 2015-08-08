---
layout: post
title: 'Fun with Assembly 2: Backdoors and Master Passwords'
date: 2015-01-10 20:01:15.000000000 +00:00
categories:
- fun with assembly
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
This is the second in a series of articles about assembly language. If you're reading this, you should [make sure you already understand how Assembly, CPUs, RAM and registers work.]({% post_url 2015-01-10-fun-with-assembly-1 %})

<h2>The setup</h2>
This post is going to be based around the New Orleans level of <a href="http://microcorruption.com">microcorruption.com</a>. You can login there and follow along if you like, but I'll be explaining everything in this post too.

The idea is that we have a test copy of a digital lock and we're trying to find some input to it that will make it open, even though we haven't been given the key (in this case, a password). We have the copy of the lock, access to disassembled code, the current state of memory, the CPU's registers, the output and a debugger.

Now with the debugger we could easily get it to run whatever code we want and open the lock. But that doesn't really help, because we also need to be able to use our exploit on a lock in the real world that we don't have privileged access to. We need some input that will open the lock, even though we haven't been told the password.

<h2>Diving in</h2>
<span style="line-height: 1.5em;">The first thing to look at is the main function, to see what the lock is going to do when it runs. It starts with this:</span>

<pre><code>4438 &lt;main&gt;
4438:  3150 9cff      add   #0xff9c, sp
443c:  b012 7e44      call  #0x447e &lt;create_password&gt;
4440:  3f40 e444      mov   #0x44e4 "Enter the password to continue", r15
4444:  b012 9445      call  #0x4594 &lt;puts&gt;
4448:  0f41           mov   sp, r15
444a:  b012 b244      call  #0x44b2 &lt;get_password&gt;
444e:  0f41           mov   sp, r15
4450:  b012 bc44      call  #0x44bc &lt;check_password&gt;
</code></pre>
So, <code>main</code> makes some room on the stack, calls a function called <code>create_password</code>, asks the user to put in their password, gets the password the user types, and then checks if it the password was right.

We're not going to look at the rest of <code>main</code> for now. Instead, let's look at what <code>create_password</code> is doing:

<pre><code>447e &lt;create_password&gt;
447e:  3f40 0024      mov   #0x2400, r15
4482:  ff40 6a00 0000 mov.b #0x6a, 0x0(r15)
4488:  ff40 2100 0100 mov.b #0x21, 0x1(r15)
448e:  ff40 4000 0200 mov.b #0x40, 0x2(r15)
4494:  ff40 5600 0300 mov.b #0x56, 0x3(r15)
449a:  ff40 4e00 0400 mov.b #0x4e, 0x4(r15)
44a0:  ff40 2f00 0500 mov.b #0x2f, 0x5(r15)
44a6:  ff40 2900 0600 mov.b #0x29, 0x6(r15)
44ac:  cf43 0700      mov.b #0x0, 0x7(r15)
44b0:  3041           ret
</code></pre>
Well there must be something special about the address <code>0x2400</code>. This function moves that address into register 15 then stores 7 specific bytes there, followed by a null <code>0x0</code> byte. That sounds a lot like a 7-character string: each character in a string is 1 byte, and they always end in a null byte. This function is called <code>create_password</code>, so those 7 characters are almost certainly a fixed, hard-coded password.

A quick look at <code>check_password</code> shows:

<pre><code>44bc &lt;check_password&gt;
44bc:  0e43           clr   r14
44be:  0d4f           mov   r15, r13
44c0:  0d5e           add   r14, r13
44c2:  ee9d 0024      cmp.b @r13, 0x2400(r14)
44c6:  0520           jne   #0x44d2 &lt;check_password+0x16&gt;
44c8:  1e53           inc   r14
44ca:  3e92           cmp   #0x8, r14
44cc:  f823           jne   #0x44be &lt;check_password+0x2&gt;
44ce:  1f43           mov   #0x1, r15
44d0:  3041           ret
44d2:  0f43           clr   r15
44d4:  3041           ret
</code></pre>
We could work through this, but line <code>0x44c2</code> stands out: that line is comparing two things, one of which is related to the address <code>0x2400</code>, which is where <code>create_password</code> was storing what it generated. So it's probably comparing the input we entered with the string from <code>create_password</code>. At this point, you should be pretty sure that the hunch was right, and that the string is the password.

If you put a breakpoint on the line after <code>create_password</code> is called from <code>main</code> (which is <code>0x4440</code>), and run the code to there, you can look at the contents of the magic location where our seven character password is expected to be, <code>0x2400</code>. I see <code>j!@VN/)</code> but it might be different for you. That's our password.

<h2>Is this realistic?</h2>
This was a quick and easy lock to hack into, because it had a saved, fixed password. Surely that would never happen in the real world?! Actually, it does happen often. Maybe not quite as simply as this, but a fixed password for gaining access to a system is normally called a <a href="http://en.wikipedia.org/wiki/Backdoor_%28computing%29">backdoor</a>.

Here's a list of <a href="https://www.gnu.org/philosophy/proprietary-back-doors.html">consumer devices with backdoors</a>. It includes all Android and iOS phones, Windows, Samsung phones, Amazon Kindles, routers, etc. It might even be possible to <a href="http://danluu.com/cpu-backdoors/">build backdoors right into CPUs</a>.

Most of the time a backdoor is written into software, it's to prevent illegal or unlicensed use. For example, <a href="http://www.nytimes.com/2009/07/18/technology/companies/18amazon.html?_r=0">the Orwellian time Amazon covertly removed 1984 from people's Kindles</a>. The problem with backdoors is that they are intended for one company's use but, once one exists, malicious hackers and other nefarious organisations will find out about the backdoor and start exploiting it for their own ends. Backdoors in the real world are harder to trigger than the one we broke into, so only the most dedicated hackers will be able to abuse them.

<h2>Congratulations!</h2>
You just cracked your first lock, and used knowledge of disassembly to find hidden capabilities in software! In the next part, we'll learn more assembly and solve a more complex lock.

<small>Thanks to Tom Carver for reading a draft of this post.</small>

