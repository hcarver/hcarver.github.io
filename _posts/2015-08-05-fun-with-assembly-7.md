---
layout: post
title: 'Fun with Assembly 7: Cracking attempts at security through obscurity'
date: 2015-08-05 22:23:43.000000000 +01:00
categories: []
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
This is the seventh in a series. You might want to read the [previous post]({% post_url 2015-08-05-fun-with-assembly-6 %}) before reading this.

This post is based on the Reykjavik level on [microcorruption.com](http://microcorruption.com). Like last time, we’re trying to find an input to open a lock without knowing the correct password, using knowledge of assembly language.

<h1>First glance</h1>
<pre><code>443e:  3e40 f800      mov   #0xf8, r14
4442:  3f40 0024      mov   #0x2400, r15
4446:  b012 8644      call  #0x4486 &lt;enc&gt;
444a:  b012 0024      call  #0x2400
</code></pre>
Hmm. We call a function called <code>enc</code>, passing the location <code>0x2400</code>, then we call <code>0x2400</code> itself. That's unusual because at this point, <code>0x2400</code> is full of junk, not usable assembly code. If we let the code run up until <code>0x444a</code>, we find that the content of <code>0x2400</code> has been updated. So it looks like <code>enc</code> was decrypting something that was there into runnable assembly.

<h1>Decompiling</h1>
By pausing the running code, we can copy the decrypted instructions into a decompiler, to convert it to readable assembly, which we can then understand. The decrypted code begins with one long function, so let's begin with that.

<pre><code>2400: 0b12           push   r11
2402: 0412           push   r4
2404: 0441           mov    sp, r4
2406: 2452           add    #0x4, r4
2408: 3150 e0ff      add    #0xffe0, sp
240c: 3b40 2045      mov    #0x4520, r11
2410: 073c           jmp    $+0x10
2412: 1b53           inc    r11
2414: 8f11           sxt    r15
2416: 0f12           push   r15
2418: 0312           push   #0x0
241a: b012 6424      call   #0x2464
242e: 2152           add    #0x4, sp
2420: 6f4b           mov.b  @r11, r15
2422: 4f93           tst.b  r15
2424: f623           jnz    $-0x12
2426: 3012 0a00      push   #0xa
242a: 0312           push   #0x0
242c: b012 6424      call   #0x2464
2430: 2152           add    #0x4, sp
2432: 3012 1f00      push   #0x1f
2436: 3f40 dcff      mov    #0xffdc, r15
243a: 0f54           add    r4, r15
243c: 0f12           push   r15
243e: 2312           push   #0x2
2440: b012 6424      call   #0x2464
2444: 3150 0600      add    #0x6, sp
2448: b490 8d8c dcff cmp    #0x8c8d, -0x24(r4)
244e: 0520           jnz    $+0xc
2450: 3012 7f00      push   #0x7f
2454: b012 6424      call   #0x2464
2458: 2153           incd   sp
245a: 3150 2000      add    #0x20, sp
245e: 3441           pop    r4
2460: 3b41           pop    r11
2462: 3041           ret
</code></pre>
There's a lot here, but two lines stand out

<pre><code>2450: 3012 7f00      push   #0x7f
2454: b012 6424      call   #0x2464
</code></pre>
<code>0x2464</code> looks like it's the interrupt function, because it's called so many times. And the manual tells us that <code>0x7f</code> is the argument passed to unlock the door. So how do we get this to be called? The two lines before have:

<pre><code>2448: b490 8d8c dcff cmp    #0x8c8d, -0x24(r4)
244e: 0520           jnz    $+0xc
</code></pre>
We can see that the lines that open the door are only called if <code>-0x24(r4)</code> is equal to the literal value <code>0x8c8d</code>. It's not much of a stretch to guess that <code>-0x24(r4)</code> is the start of the string. So let's try an input of <code>0x8d8c</code> (remember: we reverse the byte order). It works!

<h1>Endnotes</h1>
As a rule, security through obscurity doesn't work. Hiding <em>the way</em> you make something secure doesn't really secure it, because with a little hard work someone can come and uncover <em>the way</em> you make something secure. That's not the same as hiding <em>the password</em> to make something secure - it was easy for us to uncover the assembly code that was being run here. But that wouldn't have helped us at all if the password had been a better-kept secret.

