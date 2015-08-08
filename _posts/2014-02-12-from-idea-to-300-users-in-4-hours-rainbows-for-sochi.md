---
layout: post
title: 'From idea to 300 users in 4 hours: Rainbows for Sochi'
date: 2014-02-12 12:38:10.000000000 +00:00
categories: []
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
I made [a widget for adding a rainbow to your profile pictures](http://rainbows-for-sochi.herokuapp.com), so you can show your support of LBGTQ people during the Sochi Winter Olympics. This post is why I did it, how I did it and what I learned.

![My new Twitter profile picture](/assets/sochi_image.png)

# Background

[LGBTQ](http://en.wikipedia.org/wiki/LGBT) people have a [hard time of it in Russia](www.leftfootforward.org/2014/02/5-reasons-lgbt-activists-are-protesting-about-russia/). In the run-up to the Winter Olympics 2014 in Sochi (a city in West Russia), Russian president and Dobby the house-elf-lookalike Vladmir Putin said that gay people would be welcome but they [had to leave children in peace](http://www.theguardian.com/world/2014/jan/17/vladimir-putin-gay-winter-olympics-children). Anti-gay sentiments like this have led to [gang violence against gay people](http://www.theguardian.com/world/2014/feb/01/russia-anti-gay-gang-violence-homophobic-olympics).

During the Sochi opening ceremony, [Rebecca Front](http://en.wikipedia.org/wiki/Rebecca_Front) a British comic actress, popped up on my Twitter feed with this:

<blockquote class="twitter-tweet" lang="en">I've no idea by what magic he did it, but <a href="https://twitter.com/RupertMyers">@RupertMyers</a> is adding rainbowness to avatars for a bit of <a href="https://twitter.com/search?q=%23LGBT&src=hash">#LGBT</a> solidarity. Plus it looks nice

— Rebecca Front (@RebeccaFront) <a href="https://twitter.com/RebeccaFront/statuses/431858521966403584">February 7, 2014</a>
</blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Rupert Myers was adding Rainbow effects to people's Twitter profile pictures but was doing each one <strong>manually</strong> in PhotoShop. And he did <strong>a lot</strong>.

<blockquote class="twitter-tweet" lang="en">Ok, sorry. No more rainbowification for a while. This is really getting quite exhausting.

— Rupert (@RupertMyers) <a href="https://twitter.com/RupertMyers/statuses/431862388959105024">February 7, 2014</a>
</blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

I figured this was worth automating. I was hoping I could make everyone on my Twitter feed turn rainbow-coloured in response to homophobic and transphobic violence.

# V1

At the time, I was cooking dinner for the girlfriend. Luckily, she was late home, so I had a couple of hours to work.

I used Rails to put together an app with two actions: `GET` the index and `POST` a photo to be "rainbowed". These were stored on Amazon S3 via the carrierwave and fog gems. I used RMagick to blend the uploaded images with a square rainbow image. The image at the top of the post was the first one I made. I saw the tweets about at about 7pm, and had the first working version by 8:15. It took me until 9:50 to get a version that was working hosted on Heroku. This was only the standard gem and config problems when hosting.

That done, I started tweeting to the people who'd been turned down by Rupert, as well as to celebrities who were telling their followers where they'd got the rainbows on their profile pictures. One of these was <a href="http://en.wikipedia.org/wiki/Lauren_Laverne">Lauren Laverne</a>, a British presenter and DJ. She tweeted a link to her 294,000 followers at about 11:30pm. Rebecca Front also retweeted me to her 81,000 followers.

<blockquote class="twitter-tweet" lang="en">For everyone asking about my rainbow <a href="https://twitter.com/search?q=%23rainbowsforSochi&src=hash">#rainbowsforSochi</a> <a href="https://twitter.com/h_carver">@h_carver</a> <a href="https://twitter.com/branchenergy">@branchenergy</a> <a href="https://twitter.com/EmmaGx">@EmmaGx</a> Automate your rainbows at <a href="https://t.co/fZoB5sdolm">https://t.co/fZoB5sdolm</a> !”

— Lauren Laverne (@laurenlaverne) <a href="https://twitter.com/laurenlaverne/statuses/431932950922682370">February 7, 2014</a>
</blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

# V2

Now the focus was to try to help spread the existence of the app. I was posting to friends on the Facebook, and tweeting to strangers but I wanted to encourage users to share the site.

I watched some TV and chatted with the girlfriend for a while. I'm pretty lucky that she understands if I have an urge to work at weird times. At midnight, I added the latest version which used the 'social-share-button' gem to add a load of iconographic share links to the site's home page.

# V3

It's really exciting seeing people using something you've made. It's also really exciting seeing the number of users ticking up on the admin interface. I wanted to make it even easier to use the app to update your Twitter profile.

The next version added a second option to put the rainbow on your Twitter profile in 2 clicks. One to start the OAuth process, one to accept the permissions requested. Twitter has a coarse-grained permissions model so the permissions requested are pretty big. I left up the manual method too for everyone who balked at the list of permissions. I got this live at 2:15am on Saturday morning.

The big delay from the previous version was because it took me ages to work out whether the Twitter API gem included the function to push new profile pictures. The gem is comprehensive but its documentation isn't; I ended up digging through the source on GitHub.

# V4

Saturday passed. On Sunday morning, I realised I should be monitoring how many people were using the automatic vs the manual path.

Up until now, the manual path ended with your `POST` request redirecting you to download the rainbowed image. The automatic path (via the Twitter API) ended by redirecting back to your Twitter profile. I realised I was missing a trick: when users were most enthused about the rainbowing process (right after their profile had been rainbowed), I was taking them away from my site.

I added the vanity gem to start monitoring how many users used each path, expecting the majority to be using the Twitter API. I then added a page to the end of the Twitter API workflow, which showed the user the resulting rainbowed photo, told them it was already on their profile and suggested they share to Facebook / Twitter. I also changed to using the social-like JQuery plugin. It gives much larger share buttons, which I hoped would encourage sharing.

# V5

A left the site up for a couple of hours before looking at the stats. These went pretty strongly against my expectations. I thought the average user would take convenience over security, i.e. they'd use the two-click Twitter API method and trust in my disclaimer that I wouldn't abuse all the permissions they had to grant me.

The opposite turns out to be true. 2 out of every 3 users were using the <strong>manual</strong> method of uploading a picture then downloading the result. It might be that this was spreading on the Facebook in a way I couldn't see, and users there weren't interested in the Twitter method. Either way, at about midday, I put up the latest version which redirect users from both workflows to the same finishing page with their photo, a download button and some big share links.

# Tailing off

At this point, usage seemed to be tailing off. So I started contacting people on Twitter who'd written vaguely similar tweets, pointing them to the site. Not much happened.

# Graph

Everyone likes a graph.

<iframe width="100%" height="500" src="http://jsfiddle.net/tGzZM/embedded/result" allowfullscreen="allowfullscreen" frameborder="0"></iframe>

There's a really obvious step up when Lauren Laverne tweeted a link to the site. This is much larger than all other effects except for the change at 9am every day when the British public wake up (it seems that the rainbows did not make it across the Atlantic). The changes I made to make both ranibowing and sharing easier are tiny compared to those.

# Lessons

<ul>
<li><strong>Iterative development is amazing.</strong> You can strike while the iron's hot, you get users even though the app isn't perfect, and you respond to the use cases you see. I've never seen a clearer example of that.</li>
<li><strong>Tweeting to random famous people doesn't always work.</strong> They're probably getting thousands of tweets like mine every day. If they're not already invested in what you're doing, they probably won't respond. </li>
<li><strong>Feedback can be totally unexpected.</strong> A friend told me that she has synaesthesia and that all of the rainbowed profiles looked like a second Wednesday afternoon in G Major. I've no idea if this is a common problem, or even if it's a problem at all.</li>
<li><strong>Enthusiastic users will spread things even when you make it hard for them to.</strong> It can be hard to get your head around this but sometimes a user will just really like what you've made.</li>
<li><strong>Get sharing right early.</strong> If you're aiming to create a viral phenomenon, your most important features are those that promote sharing.</li>
<li><strong>Get measurements in early.</strong> Measurements tell you what to do.</li>
<li><strong>This is an amazing time to make things.</strong> In 1.5 hours, I had a webapp that took photos and did a complex operation with them and sent the result back to the user. In another 2 hours, it was available to anyone in the world. In a few more hours, hundreds of people around the world were using it and sharing it. None of this has cost me anything*.</li>
</ul>
<small>* I'll probably be charged like $3 for S3 storage this month.</small>

