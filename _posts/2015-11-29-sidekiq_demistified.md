---
layout: post
title: "Sidekiq demistified"
tags: type_system meetup talk
excerpt: Follow up to a talk about Sidekiq's internals at a local Ruby meetup.
class: talk
comments: false
---

A couple of days ago I held a presentation about [Sidekiq's](http://sidekiq.org) internals at a local [Ruby meetup](http://www.meetup.com/rubyzg/events/223260725).

The fact that I didn't know how exactly Sidekiq worked always bugged me. I wanted to know more about how it queues and schedules jobs so I dug around its source code until I found out. It turns out it has a really elegant design and relies on Redis for queuing and scheduling as much as it can (which is a good thing - because Redis is awesome).

I'm gonna cover what I found out in detail in one of future blog posts and just post the slides here for now.

<iframe src="/talks/sidekiq_demistified.html" width="600" height="450"></iframe>

<br/>

The slides were build using [remark.js](https://github.com/gnab/remark) and are hosted [locally](/talks/type_systems.html).
