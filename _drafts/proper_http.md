---
layout: post
title: "Building proper RESTful APIs in Clojure"
tags: http protocol clojure
comments: false
---

A while ago I presented on [one of the CanJS community hangouts](http://youtu.be/AJUZOefv2NE?t=52m). Since I haven't had the time since then to write a follow-up post, why not do it now, at 5:30 am, not being able to sleep any more :)

Anyway, Clojure and liberator huh? I've been playing with Clojure (on and off) for a while now, so naturally, I wanted to produce something tangible in it. One of my playgrounds was a project in which I tried to create a RESTful API. Since that's what takes most of my regular working day, I wanted to see how it feels to build one in Clojure. And, as it turns out, it feels quite nice.

There are a couple of libraries in Clojure that make writing a properly built API really easy. I say libraries because in Clojure that's what most of things are, meaning that there are no real frameworks, mostly because there's no need for those. 

The usual combination of libraries is ring and Compojure (for middleware and routing) plus other domain specific libraries (database access, messaging, caching, etc.). To make things simple, for this particular project I wanted a minimal scope with only routing, database access and rendering JSON.

Half way through the process of building this thing, I started using a very good [reference book](https://pragprog.com/book/dswdcloj/web-development-with-clojure). In it there is a part about building services with [Liberator](https://github.com/clojure-liberator/liberator), so naturally I wanted to sew what was this thing about.
