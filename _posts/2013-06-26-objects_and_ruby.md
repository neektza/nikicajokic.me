---
layout: post
title: "Objects and Ruby"
tags: programming ruby oop
excerpt: Ruby's object orientated nature requires that an object must be instantiated if a function (ie. a method) is to be invoked. That is inconvenient.
comments: true
---

> It is better to have 100 functions operate on one data structure than to have 10 functions operate on 10 data structures.
>
> -- Alan J. Perlis

Not so recently I've become more and more frustrated with Ruby's object orientated nature. Specifically, the fact that one needs to instantiate something just to use a few functions.

I am not trying to say that it has to be done this way, because it mostly depends on style. However, it *is* the established norm in the Ruby/OOP world. *Service object* is the name of this idiom, I believe.

An example:

{% highlight ruby linenos %}

class SomeServiceObject
  def initialize(foo)
    @foo = foo
  end

  def a_method
    manipulate(foo)
  end
end

SomeServiceObject.new(foo).a_method

{% endhighlight %}

In the example, we keep state in the object, although we could have simply passed an argument to the function. It is a trivial example, for sure, but I'm just using it to illustrate a point. This kind of code pops up all over the place in Ruby libraries, albeit more complex. Nevertheless, keeping functions in an object just because they operate on a certain kind of data is a bad pattern, in my opinion. 

I much prefer passing data to a function than invoking a method. If data is complex and consists of multiple components, then we could represent it with an object and keep only state/data related methods in that object. Independent data processing functions should be kept out of the construct that represents data - i.e. object. 

## Object <> Function duality

On a related note, why should a function be an object (and in Ruby everything is) or be forced to reside in one (since even the global context is an object)? It should be a first-class citizen, like in Javascript, LISP or Haskell. Well, actually, a function in Javascript is also an object, but at least it is a first class citizen without special syntax for invocation.
