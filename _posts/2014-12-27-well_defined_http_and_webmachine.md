---
layout: post
title: "Well-defined HTTP APIs and Webmachine"
tags: programming ruby oop
class: talk
comments: false
---

Last week I held a talk about building web APIs and using HTTP properly while doing it.

<script async class="speakerdeck-embed" data-id="1df787b06d1f0132d73b1a31a1db69c9" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

<br/>

Basically, it was a talk about [**Webmachine**](https://github.com/seancribbs/webmachine-ruby) and its Ruby implementation. It's a very interesting library that you should definitely check out if you want to explore how to build APIs differently. Much more differently than [Sinatra](http://www.sinatrarb.com/) or [Grape](http://intridea.github.io/grape/).

In short, it encodes the HTTP protocol in a state machine, and guides you in crafting well defined HTTP responses by exposing state machine transitions as method stubs. You'll be able to read more about this topic in an upcoming (full-blown) article.
