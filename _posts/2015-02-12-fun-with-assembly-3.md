---
layout: post
title: 'Fun with Assembly 3: Deeper into the Assembly Beast'
date: 2015-02-12 22:34:48.000000000 +00:00
categories:
- fun with assembly
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
This is the third in a series of articles about assembly language. You might want to read
[the previous post]({% post_url 2015-01-10-fun-with-assembly-2 %}), and the ones that came before it, before this one.

This post is based around the Sydney level on <a href="http://microcorruption.com">microcorruption.com</a>. Like last time, we're trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language. I'm going to explain more about assembly language than last time, and show more of the debugging process.

<h3> Diving into the <code>main</code> function</h3>
Let's get straight into the assembly. Here's the first half of the <code>main</code> function.

<pre><code>4438:  3150 9cff      add #0xff9c, sp
443c:  3f40 b444      mov #0x44b4 "Enter the password to continue.", r15
4440:  b012 6645      call  #0x4566 &lt;puts&gt;
4444:  0f41           mov sp, r15
4446:  b012 8044      call  #0x4480 &lt;get_password&gt;
444a:  0f41           mov sp, r15
444c:  b012 8a44      call  #0x448a &lt;check_password&gt;
4450:  0f93           tst r15
</code></pre>
Let's break that down further, and read it line by line.

<code>add    #0xff9c, sp</code> adds to the stack pointer. I've talked about <a href="http://en.wikipedia.org/wiki/Integer_overflow">integer overflow</a> before, so you should remember that this will actually decrease the value of the stack pointer by 100 bytes (0x100 - 0x9c). Stacks grow downwards, so this is making the stack larger by 100 bytes for the main function to use.

<pre><code>mov #0x44b4 "Enter the password to continue.", r15
call  #0x4566 &lt;puts&gt;
</code></pre>
will move the value <code>0x44b4</code> (which, the debugger helpfully tells us, points to a string saying "Enter the password to continue") into register 15, then calls the puts function. This is the same as calling puts with the string as an argument, so that it outputs to the console.

<pre><code>mov  sp, r15
call  #0x4480 &lt;get_password&gt;
</code></pre>
Move the stack pointer into register 15, then call the function at <code>0x4480</code> (again, we're helpfully told that it's called <code>get_password</code>). Remember, register 15 is the default place for the first argument for a function, so this is calling the function and passing the stack pointer as the argument.

<pre><code>mov sp, r15
call  #0x448a &lt;check_password&gt;
tst r15
</code></pre>
This looks similar to what came before. We call <code>check_password</code> with the stack pointer as an argument. The assembly has to copy it into register 15 again because the value in r15 might have been changed by <code>get_password</code> since we last moved the stack pointer. The last line <strong>tests</strong> r15. Which means, it will look at the value in r15 and populate the status register accordingly. If the value is negative, the status register's "negative" bit will be 1. If the value is zero, the status register's "zero" bit will be 1. Why does it do that? The answer is in the other half of the <code>main</code> function.

<pre><code>4452:  0520           jnz #0x445e &lt;main+0x26&gt;
4454:  3f40 d444      mov #0x44d4 "Invalid password; try again.", r15
4458:  b012 6645      call  #0x4566 &lt;puts&gt;
445c:  093c           jmp #0x4470 &lt;main+0x38&gt;
445e:  3f40 f144      mov #0x44f1 "Access Granted!", r15
4462:  b012 6645      call  #0x4566 &lt;puts&gt;
4466:  3012 7f00      push  #0x7f
446a:  b012 0245      call  #0x4502 &lt;INT&gt;
446e:  2153           incd  sp
4470:  0f43           clr r15
4472:  3150 6400      add #0x64, sp
</code></pre>
The first thing this does is <code>jnz #0x445e &lt;main+0x26&gt;</code>, which is to jump to <code>0x445e</code> if we have a non-zero number. Jumping to an address means that the current path of execution will stop, and the next instruction will be the one that's jumped to. If the jump doesn't happen (in this case, when the number is zero), it just carries on with the next line of assembly. The previous line (<code>tst r15</code>) tested register 15, which sets the bits in the status register depending on whether r15 is 0 or positive, etc. The line we're currently looking at uses <code>jnz</code> to check the value of the zero bit in the status register, so the overall effect is to test whether the contents of register 15 are 0. In this situation, we will jump to <code>#0x445e</code> if r15 was non-zero after <code>check_password</code> returned.

The next two lines output "Invalid password; try again.", then skip ahead to the end of the function.

If the jump does happen, the CPU will run the lines from <code>0x445e</code>. First of all, this outputs "Access Granted!". Then it does this:

<pre><code>push  #0x7f
call  #0x4502 &lt;INT&gt;
</code></pre>
This is a hardware <a href="http://en.wikipedia.org/wiki/Interrupt">interrupt</a>, which is necessary for all attempts to read input from another process, all attempts to send output to another process, or any other operation that interacts with the outside world. We generally call input or output 'IO', and operations that stop the program from continuing 'blocking' operations.

This interrupt has the argument <code>0x7f</code>, which is the magic code for "open the lock" (there's a manual which lists that kind of thing). This argument is passed on the stack, not in a register (for each CPU, rules called "calling convention" determine whether arguments are on the stack or in registers - it just so happens that this CPU's interrupt takes arguments on the stack). The next line, <code>incd sp</code> is to clear up the 2 extra bytes of stack space that were used by pushing the value <code>0x7f</code> onto the stack (2 bytes is the minimum variable size on a 16-bit system, so even though <code>0x7f</code> is 1 byte, it takes up 2 bytes on the stack). <code>incd</code> is short for "increment double", so it will add 2 bytes to the stack pointer (mirroring the 2 bytes used by the value we pushed).

After the <code>jnz</code> instruction branched code execution, the two branches join again at <code>0x4470</code>, which does <code>clr r15</code> and <code>add #0x64, sp</code>. This clears the value of r15, which means this function is returning 0, and then adds 100 on to the stack pointer, to clear up the extra stack space taken by the start of the function.

So everything here hinges on the return value of <code>check_password</code>. If it's non-zero the lock is opened. If it's zero, the lock stays shut.

<h3>Looking at <code>get_password</code></h3>
The <code>get_password</code> function is simple.

<pre><code>4480:  3e40 6400      mov #0x64, r14
4484:  b012 5645      call  #0x4556 &lt;getsn&gt;
4488:  3041           ret
</code></pre>
Put the value 100 in register 14, then call <code>getsn</code>. Remember that this function was called with the stack pointer in register 15? Well that will still be there. So <code>getsn</code> is called with the stack pointer as the first argument, and the value 100 as the second. This second argument instructs <code>getsn</code> to take at most 100 characters (including the null character) as input. The bad news (from our perspective) is that 100 is exactly how much stack space has been allocated for the input. I'm saying this is bad news because, if the stack had only allocated, say, 30 bytes, we'd be able to overwrite 70 bytes of memory which we weren't meant to get access to.

This function then returns.

<h3>Looking at <code>check_password</code></h3>
Here's the assembly.

<pre><code>448a:  bf90 342b 0000 cmp #0x2b34, 0x0(r15)
4490:  0d20           jnz $+0x1c
4492:  bf90 3529 0200 cmp #0x2935, 0x2(r15)
4498:  0920           jnz $+0x14
449a:  bf90 5a34 0400 cmp #0x345a, 0x4(r15)
44a0:  0520           jne #0x44ac &lt;check_password+0x22&gt;
44a2:  1e43           mov #0x1, r14
44a4:  bf90 3c4c 0600 cmp #0x4c3c, 0x6(r15)
44aa:  0124           jeq #0x44ae &lt;check_password+0x24&gt;
44ac:  0e43           clr r14
44ae:  0f4e           mov r14, r15
44b0:  3041           ret
</code></pre>
There's definitely a pattern here. Lots of lines of comparing a literal value, against something near where r15 points to. Remember that r15 is set to the stack pointer before this function is called. And the stack is where the inputted password is. By the way, <code>jnz</code> and <code>jne</code> can be used interchangeably (as can <code>jz</code> and <code>je</code>) - all 4 looking at the zero bit-flag on the status register.

So we compare the first pair of bytes of the input against <code>0x2b34</code>, the second pair with <code>0x2935</code> the third pair with <code>0x345a</code>. If ever the two things aren't equal, we jump to the same line (<code>0x44ac</code> - sometimes this is calculated as an offset. E.g. from <code>0x4490</code>, we jump <code>$+0x1c</code>, <a href="https://www.google.co.uk/search?q=0x4490+%2B+0x1c&amp;oq=0x4490+%2B+0x1c">which still takes the CPU to <code>0x44ac</code></a>).

The next lines are

<pre><code>mov #0x1, r14
cmp #0x4c3c, 0x6(r15)
jeq #0x44ae &lt;check_password+0x24&gt;
</code></pre>
So we put the value 1 into register 14, then do another comparison, of the 4th pair of bytes against <code>0x4c3c</code>. This time, we jump if they're equal - and we only jump over one instruction, skipping the line <code>0x44ac</code> (which the other jumps aim for).

That line <code>0x44ac</code> reset r14 to 0. So if any of the comparison are not equal, that line will be hit. Otherwise, it'll be set to 1. The next line, <code>mov r14, r15</code>, copies the value into register 15, which will be the return value. The last line performs the actual return.

<h3>The password</h3>
We've seen that pairs of bytes in the password need to equal <code>0x2b34</code>, <code>0x2935</code>, <code>0x345a</code> and <code>0x4c3c</code> in turn. So can we just input <code>0x2b342935345a4c3c</code> as the password, and open the lock?

No is the answer. The reason is to do with <a href="http://en.wikipedia.org/wiki/Endianness">endian-ness</a>. This is all to do with how bytes are stored in a memory not being the same as how they're written out. On the CPU we're using, <a href="http://en.wikipedia.org/wiki/TI_MSP430#MSP430_CPU">bytes are stored little-endian within each 16-bit word</a>. That means that a value in the real world of <code>0x0102030405060708</code> would be stored in memory as <code>0x0201040306050807</code>, with the bytes swapped in every pair.

There's no great trick to this, it's just something you have to be aware of whenever you're converting from real-world values to values in memory. Although there is a bit of a hint here from comparing the assembled version of the program with the disassembled version. E.g. <code>448a:  bf90 342b 0000 cmp #0x2b34, 0x0(r15)</code> The assembled version on the left shows <code>342b</code>, which is displayed as <code>0x2b34</code> on the right. That's as much a hint as you'll ever get.

To find the real password, we have to swap the bytes in every pair of the password we had above. <code>0x2b342935345a4c3c</code> becomes <code>0x342b35295a343c4c</code>.

And with that, the lock opens.

<h3>Endnotes</h3>
This hack was similar to the last one. The lock had a master password which was hard-coded in the program. The only extra challenge was that it wasn't hardcoded as a readable string, but hardcoded in the instructions.

Next time, we'll look at using a buffer overflow to open a lock by writing to memory we weren't meant to access.

<small>Thanks to Tom Carver for reading a draft of this.</small>

