---
layout: post
title: 'Building an 8-bit Computer: ALU'
date: 2019-01-15
status: published
type: post
published: true
author: Hywel Carver
---

It's been a few months since my last update, but the registers are now all fully working. I've been slowly building the ALU, and tested it today. It worked first time!

[This post is part of a longer series about the build process for an 8-bit computer.]

Here is the new ALU showing that 255 + 128 = 127 (mod 256), with the carry bit set.

![Register layout](/assets/alu_working.jpg)

One of the things that's really slowed me down is that the programme I'm using to design my stripboard layouts has a nasty habit of crashing. And when it crashes,
it deletes the file I'm working on so I have to start again. It's called Fritzing - and it's brilliant except for this one (quite major) problem.

Next up, I'm going to build a status register. I've been thinking hard about the carry bit - whether to make it
a true carry bit, an overflow bit, or simply use the carry out of the sum / subtraction.

I've decided to do it the simple way for now. That will mean that it's inaccurate in some instances due to implementation, and there won't be an overflow flag.
For now, I'm going to compensate for that in the code I write. I'll know the ins-and-outs of the processor, and can work
around its imperfections.

