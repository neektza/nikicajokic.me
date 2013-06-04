---
layout: post
title: "Objects and Ruby"
date: 2013-06-03 18:41:50
tags: ruby oop
---

Recently I've become more and more frustrated with Ruby's object orientated nature. Specifically, that one needs to instatntiate something just to use one method/function. Not to say that it must be that way, because it mostly depends on the library author, but it *is* the norm in the Ruby world.

The biggest offender is an object created only to contain a few state variables and one method

A (contrived) example:

{% highlight ruby %}
	class Container
	  def initialize(foo, bar)
	  @foo = foo
	  end
	
	  def do_something
	    i_depend_on_foo @foo
	  end
	end

	# we need to do something...
	Container.new(foo,bar).do_something
{% endhighlight %}

The previous example is a standard one in the OOP world (especially in Java where everything is a class, and Ruby where everything is an object). I don't like it. Why should a function be an object or be forced to reside in one? It should be a first-class citizen (, LISP), but should it be forced to be a class-resident/object? I think not.

There are alternatives, of course, namely module class methods (does that sound wrong or what?!):

{% highlight ruby %}
module ContainerModule
  def self.why_do_i_exist?
    p "Nobody knows"
  end
end
{% endhighlight %}

In this example, 
