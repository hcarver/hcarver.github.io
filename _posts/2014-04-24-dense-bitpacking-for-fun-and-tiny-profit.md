---
layout: post
title: Dense Bitpacking for Fun and (tiny) Profit
date: 2014-04-24 08:00:45.000000000 +01:00
categories: []
tags: []
status: publish
type: post
published: true
author: Hywel Carver
---
<strong>Premature optimisation</strong> (optimising things when you don't know that they need optimising) is the root of much terrible C++ code. This technique is an example of <strong>immature optimisation</strong>, which means optimising things when they definitely don't need optimising but doing it anyway because it's kind of fun.

I normally write about entrepreneurship and building the very earliest stages of a company, so technical posts are rare here. This post is about a technique I came up with for packing lots of bits of information into a small amount of memory, and accessing each individual bit in a single CPU op. The code for it is <a href="https://github.com/hcarver/Netflix/blob/master/NetflixDataNG.cpp">on GitHub</a>.

<h2>Background</h2>
During my undergraduate degree, I entered the <a href="http://www.netflixprize.com/">Netflix Competition</a>, a competition run by Netflix where entrants wrote algorithms to accurately predict what ratings different people would give movies they haven't seen. These algorithms produce the recommendations that e-commerce sites give you for other products you might want to buy.

The dataset Netflix provide has 100,000,000 datapoints, each linking one of 480,000 users with one of 17,770 movies and a rating the user gave that movie, from 1-5. There's also metadata including the date the user gave the rating.

I did alright at the competition but was nowhere near winning, so I spent time on making my code as fast and memory efficient as possible instead, which is a fun way for all C++ programmers to burn their time.

My approach took into account a load of different factors that could influence the rating given, including things like the day of the week, how long the user had been giving ratings for, how old the movie was and the last few ratings given by the user (an anchoring effect).

Of course these are all small compared to intrinsic properties of the user and movie themselves, which were included in the model. But they are big enough factors to make the difference between a 4 or a 5 (the jeez-I-hate-mondays effect).

<h2>RAM</h2>
The data I needed to store was 100,000,000 instances of:<br />
 - a rating value from 1-5<br />
 - a movie id from 0-17,769<br />
 - a "user era" of whether this rating is in the first 20%, 40%, 60%, 80% or 100% of the ratings a user made, stored as a number from 1-5<br />
 - a "movie era", like the user era, but to a finer scale of 2% (stored as a number in 1-50)<br />
 - the weekday the rating was made on, from 1-7<br />
 - the average of the user's previous ratings over 5 different timescales, scaled to integers from 0-99.

I didn't need to store the user number, because these ratings were stored in a vector for each different user. I couldn't also use the movie number as an index, because that's a super-inefficient way to store a sparse matrix like this dataset. Even with a single byte per rating, 480,000 * 17,770 * 1 is already 8GB of RAM.

<h2>Naive storage</h2>
Of the 10 types of metadata I wanted to store for each rating, 9 can be stored in 1 byte, and the movie id can be stored in 2 bytes.

At 11 bytes per datapoint, I could store it all in 11 * 100,000,000 = 1GB. Nice! But I think we can do better.

<h2>Bitpacking with C++</h2>
C and C++ include <a href="http://en.cppreference.com/w/cpp/language/bit_field">bitfields</a>, a syntax where you can create a struct and declare how many bits to allow for each of its member variables. The compiler then handles all of the bitfiddling so that you can transparently access them. It's a feature designed to solve problems exactly like this.

The rating could fit in 3 bits, the movie id in 15, the user era in 3, the movie era in 6, the weekday in 3, and the five items of rating history in 7 each. That's a total of 65 bits, which rounds up to 9 bytes per data point.

Not bad, but the accesses it's doing internally are two operations of a bit-mask + a bit-shift. What a waste!

<h2>Better bitpacking with modulo</h2>
Storing a 1-5 value in 3 bits is kind of wasteful. We're only using 5 of the 8 possible values for that space, effectively throwing away 38% of the memory. When we stored a 0-17769 value in 15 bits, we threw away nearly half of it.

If we only had to store two 1-5 values, we could store them as a single number from 1-25. To access the first, we take <code>x % 5</code>, to access the second we take <code>x / 5</code>.

When we have to store two 1-5 values and a 0-17769 value, we could store them as a single number from 0-444575 (19 bits instead of 21), and access the first as <code>x % 5</code>, the second as <code>(x / 5) % 5</code> and the third as <code>(x / 25)</code>.

This is better than C++ bitpacking because we can store everything as a number less than 5 * 17770 * 5 * 50 * 7 * (100^5), which fits into 61 bits (8 bytes).

But accessing the variables is starting to look ugly. The majority of our values will need to be accessed as <code>(x % something) / something_else</code>, taking an uneconomic 2 CPU ops.

<h2>Cleaner bitpacking with modulo</h2>
I didn't like the fact that it takes two operations to get to most of my metadata with the above system, so I instead came up with the following which requires only 1 op to access any part of the metadata and (sometimes) takes less memory.

For each rating, there is a single long integer, n, containing all the metadata. To access each type of information, there is a single constant <code>P_movie</code> or <code>P_rating</code> etc, and the metadata is found by <code>n % P_metadata_type</code>.

See, if you have a few prime numbers, let's say 5, 7 and 11, then every number below 5 * 7 * 11 = 385 has a unique set of values for <code>x % 5</code>, <code>x % 7</code> and <code>x % 11</code>. Those primes are our <code>P_movie</code> etc.

That means that we can encode all of our metadata using prime numbers. For each type of data, we find a different prime, greater than or equal to the top of its range. We can encode the metadata into a single number, and retrieve each part of it with a single modulo operation using the relevant prime.

For the metadata I have, the prime numbers used are 7 for the rating, 17783 for the movie id, 5 for the user era, 53 for the movie era, 11 for the weekday, and 101, 103, 107, 109, 113 for the rating histories.

That means all the metadata can be stored as an integer from 0 to 4,974,952,576,309,540,055, which still fits into 62 bits (8 bytes), but with only a single modulo operation to access each metadata item. <strong>Such efficient!</strong>

<h2>Making big numbers</h2>
The only step left is to encode our metadata into the single integer. The code for this is <a href="https://github.com/hcarver/Netflix/blob/2136aa5d28a209f902d4bbb2da8cfe731547bf5e/NetflixDataNG.cpp#L944">here</a>, but it is not beautiful.

The algorithm, in words, is this:

<pre><code>let n = 0, max_n = 1
foreach (metadata_value, prime_for_metadata) in metadata
    while(n % prime_for_metadata != metadata_value)
       n = n + max_n // This never changes the remainder with respect to all the primes we've used already
    max_n = max_n * prime_for_metadata // The max value of n is now bigger
</code></pre>
An example: we're going to store the value 2 for a variable that can go up to 3, a 4 for a variable that can go up to 5 and a 6 for a variable that can go up to 7. This can be stored in a number smaller than 105 (105 = 3 * 5 * 7).

<pre><code>First, n = 0, max_n = 1.
We're going to first add the value 2 for the prime 3.
0 % 3 != 2, so we're going to increase n by 1, and make n=1
1 % 3 != 2, so we're going to increase n by 1, and make n=2
2 % 3 == 2, so we're done with the first metadata value, and we make max_n = 1 * 3 = 3
Next, we're adding a value of 4 for a field that can go up to 5.
2 % 5 != 4, so we're going to increase n by 3, and make n = 5
5 % 5 != 4, so we're going to increase n by 3, and make n = 8
8 % 5 != 4, so we're going to increase n by 3, and make n = 11
11 % 5 != 4, so we're going to increase n by 3, and make n = 14
14 % 5 == 4, so we're done with the second metadata value, and we make max_n = 3 * 5 = 15
Lastly, we're going to add the value 3 for a field that can go up to 7.
14 % 7 != 3, so we're going to increase n by 15, and make n = 29
29 % 7 != 3, so we're going to increase n by 15, and make n = 44
44 % 7 != 3, so we're going to increase n by 15, and make n = 59
59 % 7 == 3, so we're done, and we make max_n = 15 * 7 = 105.
</code></pre>
Now we have our metadata value of 59. If we take <code>59 % 3</code> we get back 2, if we take <code>59 % 5</code> we get back 4, and if we take <code>59 % 7</code> we get back 3.

<h2>Summary</h2>
Prime numbers let you store several pieces of integer metadata information within a single integer, in a way that lets you access them with a single operation.

This probably isn't the fix to your slow code, but it is quite fun.

