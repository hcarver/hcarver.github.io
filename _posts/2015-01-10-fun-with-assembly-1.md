---
layout: post
title: 'Fun with Assembly 1: How your Computer Actually Works'
date: 2015-01-10 19:01:43.000000000 +00:00
categories:
- fun with assembly
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
This article is about what is actually going on inside your computer when you're running software. It's part of a series about how much fun assembly can be, designed to be readable by anyone trying to improve their coding and computer science skills. It covers the CPU, RAM, registers and assembly language.

The rest of the series is going to use a lot of assembly language (that's why it's called "Fun with Assembly"). Most software developers avoid assembly language whenever possible, but it's one of the most interesting programming languages around. Assembly is where software and hardware meet. It's as close to the metal of the CPU as you can get while programming so it feels very hardcore, but it also takes a LOT of code to actually achieve anything, so it feels very ineffective. Every task your computer does has to first be translated into assembly, so it must be very robust, but changing a single bit in the code can totally crash a program, so it seems very flimsy.


Here's an example of the assembly code for a simple programme that controls an electronic lock. This post will help you understand the code, and the rest of this series will help you find its flaws.

![Assembly code](/assets/assembly_code.png)

<small>I'm just a boy sitting in front of some assembly code, asking what any of it means.</small>

The other posts in this series are going to use examples from 
[microcorruption.com](http://microcorruption.com), a game about exploiting computer-controlled door locks (in your browser, not in real life). If you follow along, you'll understand how hackers can break into your computer from the other side of the world, and [why people should be very worried about the Internet of Things]({% post_url 15-01-08-hacking-the-internet-of-things %}).

<h2>Your CPU and its minions</h2>
You might already know roughly how a CPU works. Here's a quick intro anyway.

Your CPU is the brains inside your computer that does very simple calculations and operations really fast. It also tells the other components what to do. The CPU follows very precise and simple instructions given in assembly code. Actually, there isn't one single assembly language that runs on every single CPU in the world - different CPUs have different 'architectures' which each have an assembly language to go with them. We're only going to look at one here, but other assemblies are all pretty similar, so I'm just going to keep calling them "assembly language" as if there's only one language. I'm going to call code written in assembly language <em>assembly code</em>. I'll also sometimes refer to the CPU as the "processor".

The CPU only runs calculations on information that is stored in its registers. Registers are teeny tiny little bits of memory that are very close to the CPU. A very simple CPU might have 16 registers, each only 4 bytes of memory (which is very small indeed). Registers are vital, because they are where all calculations are done: when your CPU adds two numbers they have to be stored in registers, and the result will be stored in a register.

You can only store a small amount of data in registers - most of it is stored in RAM. A lot of the work your programs will be doing when they run is instructing the CPU to copy data from RAM into registers, or to copy the result of a calculation back into RAM from a register.

<h2>More about RAM</h2>
RAM (often just referred to as 'memory') is a big area of memory that can be used by your program, much bigger than registers. It works like those big racks where you get your shoes back after you've been bowling. You give them back their bowling shoes, they look at the number, go to that box and give you the right pair of shoes back. Same thing with memory, the CPU gives it a number and some data (or shoes) and it will store the shoes for you. The number is called an 'address' because it specifies the location for the data/shoes. Ask the RAM for the contents of that address, and it will give you back your data (or your shoes, if your computer uses Nike own-brand RAM).

Each program has 3 'segments' of RAM: the <strong>text</strong> segment is a copy of the program itself, loaded into memory so the CPU can run it. The others are the <strong>stack</strong> and <strong>heap</strong> segments, which contain the data produced by your program while it runs. The stack is used for data that can be predictably 'allocated' (stored) and 'deallocated', and the heap is used for everything else (it's for 'dynamic allocation')."

<h2>The stack</h2>
The stack contains a lot of your program's data, and it grows and shrinks as your program runs. Functions within your program will execute other functions which execute other functions, kind of like <a href="http://en.wikipedia.org/wiki/Matryoshka_doll">Matryoshka dolls</a>. Each of that chain of functions has some space on the stack: the first function in your program (called <code>main</code>, you can think of it like the biggest doll that contains all the others) is at the <em>highest</em> addresses in memory. The functions called by <code>main</code> have stack space at <em>lower</em> memory addresses, growing <em>downwards</em>. As a result, the lowest end of the stack always contains the RAM being used by the function that is currently executing on the CPU.

<h2>Registers</h2>
We're going to look at the Texas Instruments MSP430 processor here, because that's what's used by 
[Microcorruption](http://microcorruption.com), which the rest of the series will focus on. It’s a low-power processor, meaning it needs less electrical power and so less cooling, which makes it great for single-purpose hardware such as door-lock controllers.

I mentioned registers before - they are tiny pieces of memory directly attached right onto the CPU. The MSP 430 has 16 of them, and it's worth understanding what each is for.

<em>Register 0: pc</em>. The <em>Program Counter</em> register points to the address of the currently executing line of assembly (an <em>instruction</em>). After every operation, the CPU will move it to point to the next instruction.

<em>Register 1: sp</em>. The <em>Stack Pointer</em> register holds the address of the bottom edge of the stack. As your program calls more functions and needs more room for data, it will decrease this address to make the stack larger (remember: the stacks grows downwards).

<em>Register 2: sr</em>. The <em>Status Register</em> tells you whether the last calculation you did resulted in a negative number, overflowed, resulted in zero and some other things that we probably won't need to worry about.

<em>Register 3: cg</em>. The mysterious <em>Constant Generator</em> register seems to be only used internally by the CPU, so we don't need to know anything more about it.

<em>Registers 4-11</em> are general purpose, to be used within functions to do whatever is needed.

<em>Registers 12-15</em> are for passing up to 4 arguments to functions called by your code. If the function you're calling has one argument it goes in register 15. If it has two, you put the second argument in register 14 etc. When a function returns a value, that goes in register 15.

<small>Actually <a href="http://mspgcc.sourceforge.net/manual/x1248.html">the truth is a little more complicated than that</a>. As you'll see below, some functions that take a variable number of arguments (like <code>printf</code>), have all of their parameters on the stack.</small>

<h2>Assembly code</h2>
Let's get down to business with assembly code. Assembly code can be represented in two ways. The first is a series of 2-byte hex codes that are the binary, assembled version of the code. This seems like a good time to say that, if you've never used hex before, you're going to need to get familiar with them fast. Luckily <a href="https://www.google.co.uk/?q=0x13AF0431+%2B+0xcafebabe#q=0x13AF0431+%2B+0xcafebabe">Google is good at helping you with hex calculations</a>.

Hex code is a way of writing numbers in hexadecimal - a system which uses 16 different digits (unlike the 10 you're used to). The digits 0-9 work as normal, but we also add the digits a-f, which correspond to the numbers 10-15. Often, hex numbers are written with "0x" before them, so there's no confusion about whether it's in decimal or hex. E.g. <code>0x22</code> = 34 in decimal. Each byte in your program or in memory can be write as 2 digits of hex code between <code>0x00</code> and <code>0xff</code> (which is 0 to 255 in decimal).

Let's dig into some assembly code and see how it works. Here's some code written in hex, that we're going to analyse and understand:

<pre><code>3150 eaff 8143 0000 3012 e644 b012 c845 b140 1b45 0000 b012 c845
</code></pre>
Which might as well look like this:

![Matrix code](/assets/matrix-code.jpg)

However, <a href="https://microcorruption.com/assembler">that same code can be "disassembled" automatically</a> to get a more readable version.

<pre><code>
3150 eaff          add  #0xffea, sp
8143 0000          clr  0x0(sp)
3012 e644          push #0x44e6
b012 c845          call #0x45c8
</code></pre>
I said 'more readable', not 'readable'. Whoever originally developed assembly language had a strong hatred for words more than 4 letters long. On the left 2 columns, you can see the "assembled" hex codes for the instructions (exactly the same as in the assembled version above). The right two or three columns are the disassembled version which is assembly code.

Each instruction is a shortened version of a normal word, and you can normally guess what word is meant. The lines of assembly are always written like this: <code>&lt;instruction&gt; &lt;parameter&gt;</code> or <code>&lt;instruction&gt; &lt;parameter&gt; &lt;destination&gt;</code>.

The first line <em>adds</em> the value <code>0xffea</code> to the current value of the <em>stack pointer</em>. <code>0xffea</code> is almost the largest you can store with 4 digits of hex code so there's a pretty good chance that when the CPU adds this number to something, the result is bigger than you can contain in 4 digits of hex. If that happens, it will throw away the highest digit of the result. This is like if your car only had 3 digits on the mile-counter, then 999 would rollover to 000, instead of to 1000. The result is that adding a really big number to a register on the CPU is the same as subtracting a small number. In this case the CPU will actually decrease the stack pointer by <code>0×16</code> = 22. This phenomenon is called <a href="http://en.wikipedia.org/wiki/Integer_overflow">"integer overflow"</a>.

The second line sets an address in memory to 0 (it <em>clears</em> it). Here, it's whatever memory is being pointed to by the <em>stack pointer</em>. This command can also be used with offsets: code to clear memory 4 bytes away from the address in the stack pointer would be <code>clr 0x4(sp)</code>.

The third line pushes the number <code>0x44e6</code> onto the stack (so it makes the stack a little bit larger, and fills the new space with <code>0x44e6</code>). The fourth line <em>calls</em> a function at the address <code>0x45c8</code>. The original code I took this from has the function <code>printf</code> at that address, which writes words onto a terminal for a user to see. <code>printf</code> will be called, and will look on the stack for something to print. The last thing on the stack is <code>0x44e6</code>, the address of the thing we want to be <code>printf</code>-ed, so that's what <code>printf</code> will find.

<h2>Understanding assembly code</h2>
The more you see it, the more it looks familiar. You just have to wade through it for a while before things make sense. But when you start to get it, it feels like you've broken the Matrix.

![Matrix: broken.](/assets/breaking-the-matrix.jpg)

Here's a summary of key points about assembly language:

<ul>
<li><a href="https://microcorruption.com/assembler">Disassemble it first</a></li>
<li>Instructions are read as <code>&lt;operation&gt; &lt;source&gt; &lt;destination&gt;</code></li>
<li>Things beginning with <code>#</code> are constant numbers</li>
<li>Some instructions will refer to registers, sometimes with offsets</li>
<li>Questions about specific instructions can always be answered by Google.</li>
</ul>
<h2>What's next</h2>
This is a quick introduction to all the concepts you need to understand the rest of this series, which will look at increasingly complex and fun examples of assembly language from <a href="https://microcorruption.com">Microcorruption</a>.

You can read [the next article in this series here]({% post_url 2015-01-10-fun-with-assembly-2 %}).

<small>Thanks to Tom Carver for reviewing a draft of this.</small>

