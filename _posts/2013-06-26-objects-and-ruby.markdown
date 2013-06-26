---
layout: post
title: "Objects and Ruby"
date: 2013-06-26
tags: programming ruby oop
---

Not so recently I've become more and more frustrated with Ruby's object orientated nature. Specifically, that one needs to instatntiate something just to use a few functions.

Not to say that it must be that way, because it mostly depends on style, but it *is* the norm in the Ruby/OOP world. *Service object* is the name of this idiom, I belive.

An example:

{% gist 5872010 %}

<br/>

In the example, we keep state in the object, when we could have simply passed an argument to the function. It is a trivial example sure, but I'm just using it to illustrate a point. This kind of code pops up all over the place in Ruby libraries, albeit more complex. Nevertheless, keeping functions in an object just because they operate on a certain kind of data is a bad pattern, in my opinion. 

I much prefer passing data to a function than invoking a method. If data is complex, and it consists of multiple compononets, we could then represent it with an object and keep only state/data related methods in that object. Independent, data processing functions should be kept out of the of the construct that represents data - ie. object.

On a related note, why should a function be an object (and in Ruby everything is), or be forced to reside in one (since event the global context is an object)? It should be a first-class citizen, like in Javascript, LISP or Haskell. Well, actually, in Javascript, a function is also an object, but at least it is a first class citizen without any special syntax for invocation.
