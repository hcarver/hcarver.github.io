---
layout: post
title: The Internet of Things will Blow up your House and Trap You in Your Car
date: 2015-01-08 10:34:20.000000000 +00:00
categories:
- fun with assembly
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
Everyone I speak to at the moment seems excited about the Internet of Things. Everything in your home could be online, communicating and being controlled. Your fridge could Snapchat you when you run out of milk! Your shower could turn on the toaster when you stop running the water!

I'm not saying these aren't fun ideas that would make our lives better, but I think geeks have a responsibility to go beyond the hype and present a bigger picture.

<h2>A bigger picture</h2>
Before we had the internet this was pretty much the worst thing someone could do to your computer:

![Windows blue screen of death](/assets/Windows_9X_BSOD.png)

Crash it, wipe it, Blue-Screen-of-Death it.

With the Internet, this is pretty much the worst thing someone could get with your computer:

![iCloud photo leak](/assets/jennifer-lawrence-other-celebs-outraged-by-nude-photo-leak.jpeg)

Steal your private or secret data and sell it or <a href="http://en.wikipedia.org/wiki/2014_celebrity_photo_hack">give it away for free</a>. Having your dignity stolen and your privacy violated goes beyond just losing data.

With the Internet of Things, this is pretty much the worst thing someone could do to your computer:

![Exploding house](/assets/house_explosion.jpg)

Blow it up. Everything you own destroyed. Except your car, unless your car is on the Internet of Things too. Which it probably will be.

And even if your car doesn't get blown up, <a href="http://www.technologyreview.com/news/418969/is-your-car-safe-from-hackers/">an attacker could still disable the brakes, stop the engine and lock the doors</a>.

<h2>Think this is bullshit?</h2>
It isn't. Maybe your boiler has a firmware bug that can be exploited to cause a gas leak. <a href="http://en.wikipedia.org/wiki/Stuxnet">That's not a million miles away from a bug in a centrifuge that can make it rip itself apart.</a> If I can gain access to your IoT-enabled house, I can fill it with gas then turn on the toaster and blow it up.

That might seem crazy, but it only has to happen to one make of IoT-enabled boiler for it to be a real possibility. Once the exploit exists, running it against every Internet-connected boiler out there is not hard.

Less visually impressive exploits against your nest are possible too, that don't need me to find an exploit in the actual hardware, just a way into your home system. I can be anywhere in the world when I set the temperature of your freezer compartment to 10°C. Hope you like your meat slightly warmed for a few days!

Or maybe I get access to your thermostat and set a script to turn it off every 5 minutes. If you're young, fit and healthy in summer, that's not even an annoyance. If you're elderly and infirm, and it's a cold winter, that change to your thermostat is potentially deadly.

You might be thinking "why would a hacker want to freeze someone's home?" There are two wrong assumptions there. This kind of attack wouldn't be targeted. Once the attack exists, running it against millions of computers (and therefore millions of homes) is easy. You don't have to know whose home you're hitting, you just have to know there are homes out there to be hit. Your attacker might never even know she had been successful.

The second assumption is that this doesn't have to be intended to harm, for it to actually cause harm. <a href="http://www.telegraph.co.uk/technology/video-games/video-game-news/11314417/How-Lizard-Squad-ruined-Christmas.html">Sometimes hackers play around with the Internet for the sheer hell of it.</a> The person who hacks your house doesn't have to be malicious, just irresponsible. Or even bored.

<h2>Hacking hardware</h2>
I wrote this post because I recently started working my way through <a href="http://microcorruption.com">a hardware hacking "Capture the Flag" game called Microcorruption</a>. It's been fun, and I've learned a lot, but it's also made me realise that if someone who's crap with hardware (like me) can hack a craply made system, then someone who's good with hardware might be able to hack a well made system.

The Internet of Things is exciting, but it also massively increases the damage that a hacker can inflict on your life.

I'm going to be posting my progress with <a href="http://microcorruption.com">Microcorruption</a> here, talking about each level and how I beat it. It'll include explanations of assembly code, hacking the stack, shellcode and anything else that comes up, so you can learn how to hack hardware. <a href="http://www.twitter.com/h_carver">Follow me on Twitter</a> to get updates.

<small>Thanks to Tom Carver for reading a draft of this, and finding the article about car hacks.</small>

