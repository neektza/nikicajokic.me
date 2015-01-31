---
layout: post
title: "Building proper RESTful APIi (Ruby || Clojure)"
tags: http clojure ruby
comments: false
---


A while ago I presented on [one of the CanJS community hangouts](http://youtu.be/AJUZOefv2NE?t=52m). 

I've been playing with Clojure (on and off) for a while now, so naturally, I wanted to produce something tangible in it. One of my pet projects was one in which I tried to create a RESTful API. Since that's what takes most of my regular working day, I wanted to see how it feels to build one in Clojure. And, as it usually is in programming, there are many ways to do it, and although most of them are nice, one in particular stuck. That way is using webmachine, or Liberator (name of the Clojure port).

IT

The usual combination of libraries when building HTTP APIs are ring and Compojure (for middleware and routing) with other domain specific libraries (database access, messaging, caching, etc.). To make things simple, for this particular project I wanted a minimal scope with only routing, database access and rendering JSON.

Half way through the process of building this pet project, I started using a very good [reference book](https://pragprog.com/book/dswdcloj/web-development-with-clojure). In it there is a part about building services with [Liberator](https://github.com/clojure-liberator/liberator), so naturally I wanted to see was it was about.






