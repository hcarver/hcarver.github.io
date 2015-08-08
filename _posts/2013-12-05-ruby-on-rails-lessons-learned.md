---
layout: post
title: 'Ruby on Rails: lessons learned'
date: 2013-12-05 18:41:43.000000000 +00:00
categories:
- software development
tags:
- gems
- ruby on rails
- web apps
status: publish
type: post
published: true
author: Hywel Carver
---
Ruby on Rails is the most mature web framework that most new companies use for making their web-apps. While Rails decides a lot for you, there's often a choice of which library or plugin to use to accomplish any given goal. After doing several varied Rails projects, these are the libraries I always use to make my life easier.


## Uploads

Use [Carrierwave](https://github.com/carrierwaveuploader/carrierwave). You'll see some people advocating for Paperclip or Attachment-Fu but I've tried all three and Carrierwave is easier to use. It works very well with storing on Amazon and makes it easy to do things like caching uploads on forms, generating expiring URLs and pre-processing uploaded images.

## User and Logins

Use [Devise](https://github.com/plataformatec/devise). It's very flexible, makes OpenID and automated emailing super-simple. It'll also auto-generate views for you, so you don't need to bother writing your own from scratch. I've never even thought to look for anything else, it's that good.

## Permissions

Use [CanCan](https://github.com/ryanb/cancan). Once you get your head around its syntax, it'll seem incredibly simple. One way in which it's great is that it lets you write all of your permissions code in one place. It also includes helper functions for your controllers that'll save you a lot of effort in getting / creating / updating objects.

If you're on Rails 4, use [Authority](https://github.com/nathanl/authority). It's extremely flexible, and makes it very simple to put permissions in one file or in multiple files. Its syntax is very easy to read and it feels very natural once you've used it a couple of times.

## Templating

Use [Haml](https://github.com/indirect/haml-rails) or [Slim](https://github.com/slim-template/slim-rails). The haml-rails gem will make any new views be generated in Haml rather than ERB. It makes most HTML much, much faster and more compact to write. Slim is nice too, so try that out, but don't stick with ERB just because it's the default. Trust me that you'll be glad you changed.

## Metrics

Use [Vanity](http://vanity.labnotes.org/rails.html). It has support for simple measurements, running experiments and split tests. Last time I checked, they didn't support Rails 4, but it's a very powerful little library, and will draw you pretty graphs too.

## Asynchronous Tasks

Use [DelayedJob](https://github.com/collectiveidea/delayed_job). Again, this library is very simple to use, and works well from pretty much any context in your code. The only hard thing about using it is working out how to make your users work with an asynchronous workflow.

I hope this list is useful to someone; these choices have become my default settings when approaching any new Rails project and knowing them would have saved me a lot of time when I was starting out.