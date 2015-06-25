---
layout: post
title: "Well-defined HTTP APIs and Webmachine"
tags: ruby oop
class: talk
comments: false
---

Last week I held a talk[^1] about building web APIs and using HTTP properly while doing it.

Basically, it was a talk about [**Webmachine**](https://github.com/seancribbs/webmachine-ruby) and its Ruby implementation. It's a very interesting library that you should definitely check out if you want to explore how to build APIs differently. Much more differently than [Sinatra](http://www.sinatrarb.com/) or [Grape](http://intridea.github.io/grape/).

In short, it encodes the HTTP protocol in a state machine, and guides you in crafting well defined HTTP responses by exposing state machine transitions as method stubs.

<iframe src="/talks/well_defined_http.html" width="600" height="450"></iframe>

---
[^1]: The slides were build using [remark.js](https://github.com/gnab/remark) and are hosted [locally](/talks/well_defined_http.html).
