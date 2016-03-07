---
layout: post
title: "Celluloid internals: Proxy and Call objects"
tags: concurrency ruby actors celluloid
class: article
comments: false
---

<small> In line of previous two posts about [concurrency primitives and abstractions in Ruby](/concurrency_primitives_and_abstractions_in_ruby) and [EventMachine and the reactor pattern](/eventmachine_internals_and_the_reactor_pattern), in the last post of the series I intended to cover Celluloid's core ideas and internals. But, in the process of writing it I realized that Celluloid was much bigger than EventMachine. Consequently, this will instead be a series of posts, each covering an essential part of Celluloid.</small>

## The Actor model

First, let's define what the *Actors* actually are.

> An actor is a computational entity that, in response to a message it receives, can concurrently:
> 
> * send a finite number of messages to other actors;
> * create a finite number of new actors;
> * designate the behavior to be used for the next message it receives.[^1]

Translation: Actor model is a **system inside of which computation is done only by sending messages**. This is an ideal abstraction. Actual implementations of this model rarely limit themselves on doing computation only in this way. This is also the case with _Celluloid_.

An actor-based system is somewhat similar to an object-based system in that entities communicate with each other by sending messages. The difference between these two conceptual models is that in object based systems computation is sequential while in actor based systems computation is concurrent. In that regard, the actor model more accurately describes the real world because in the real world, everything happens concurrently.

Since Ruby has no built-in Actor facilities, Celluloid has to build them.

## Transforming objects into something more

When we want to turn our object into an actor, we include the `Celluloid` module into its class. Once the `Celluloid` module is included, an `included` hook gets called and the transformation process begins [^2].

{% highlight ruby %}

    # Actor creation
    module Celluloid
      def included(klass)
        klass.send :extend,  ClassMethods
        klass.send :include, InstanceMethods
      
        klass.send :extend, Internals::Properties
      
        klass.property :mailbox_class, default: Celluloid::Mailbox
        klass.property :proxy_class,   default: Celluloid::Proxy::Cell
        klass.property :task_class,    default: Celluloid.task_class
    
        # ... snip ...
      end
    end

{% endhighlight %}

First, the target class gets extended with Celluloid's class and instance methods. Next, the target class gets the ability to define **inheritable properties** which are a cross between class-instance variables and class variables.[^3] Celluloid uses them to save configuration settings in the class' scope. Finally, celluloid defines three properties that make its instances actors:

1. `mailbox_class` property,

2. `proxy_class` property,

3. `task_class` property.

These three properties make up the foundational parts of the library and we'll have to cover each in detail if we hope to understand how Celluloid works. But, first things first:

{% highlight ruby %}

    # Constructor overriding
    module Celluloid
      module ClassMethods
        def new(*args, &block)
          proxy = Cell.new(allocate, behavior_options, actor_options).proxy
          proxy._send_(:initialize, *args, &block)
          proxy
        end
        alias_method :spawn, :new
      end
    end

{% endhighlight %}

We can see in the snippet above that Celluloid hijacks the target's constructor method by overwriting the `new` method. Instead of returning instances of the target class, it starts returning _proxy objects_.

## Proxy objects

The only thing we need to know about the `Cell` class at this point is that it knows how and what kind of __proxies__. It does that in the [last line](https://github.com/celluloid/celluloid/blob/v0.17.3/lib/celluloid/cell.rb#L44) of its constructor method.

{% highlight ruby %}

    # Proxy creation
    class Cell
      def initialize(subject, options, actor_options)
        # ... bunch of other stuff ...
        (options[:proxy_class] || Proxy::Cell).new(
          @actor.mailbox, @actor.proxy, @subject.class.to_s)
      end
    end

{% endhighlight %}


_Proxies_ are objects that intercept regular method calls and convert them to something Celluloid calls **inter-actor message protocol**. The `Cell` factory constructs objects of class `Proxy::Cell` but the actual meat is in the `Proxy::Sync` and `Proxy::Async` classes which define what happens when sync and async methods are called.

The sync and async protocols boil down to intercepting method calls with Ruby's [`method_missing`](http://ruby-doc.org/core-2.1.0/BasicObject.html#method-i-method_missing) facility, wrapping them into `Call` objects and pushing them onto a `Mailbox` to be processed.

{% highlight ruby %}

    # Sync method interception
    class SyncProxy < AbstractProxy
      def method_missing(meth, *args, &block)
	    # ... snip
        call = ::Celluloid::Call::Sync.new(::Celluloid.mailbox, meth, args, block)
        @mailbox << call
        call.value
      end
    end

    # Async method interception
    class AsyncProxy < AbstractProxy
      def method_missing(meth, *args, &block)
        # ... snip
        @mailbox << ::Celluloid::Call::Async.new(meth, args, block)
        self
      end
    end

{% endhighlight %}

We can see that both proxy objects are intercepting all method calls and pushing them onto the mailbox. There are two important differences them though:

* `SyncProxy` objects invoke a `value` method that will ultimately respond with a result to the sender and `AsyncProxy` objects don't (they just return `self`),
* `SyncProxy` and `AsyncProxy` objects wrap the method call into a `SyncCall` and `AsyncCall` objects respectively.

## Call objects

`Call` objects represent requests towards an actor. They sit in the mailbox and wait patiently to be processed by the actor. `SyncCall` and `AsyncCall` objects represent synchronous and asynchronous requests (method calls) respectively.

When their time is up, they are processed by the actor (usually in the context of a `Task`) by invoking their `dispatch` method. The `dispatch` method basically just invokes the method that was intercepted in the `Proxy` object by using Ruby's `public_send` method:

{% highlight ruby %}

    # Call dispatch
    module Celluloid
      class Call
        def dispatch(obj)
          check(obj) # check if @method's arity matches
          _b = @block && @block.to_proc
          obj.public_send(@method, *@arguments, &_b)
        end
      end
    end

{% endhighlight %}

In addition to invoking the intercepted method, the `SyncCall` object additionally pushes the response onto the sender's mailbox.

{% highlight ruby %}

	# Result delivery
    result = obj.public_send(@method, *@arguments, &_b)
    @sender << result # @sender == sender's mailbox

{% endhighlight %}

## Performance considerations of Inter-actor message protocol

_Proxies_ and _calls_ are the foundation of what Celluloid calls inter-actor message protocol. Once (partially) broken down, we can see that the basis of that protocol is Ruby's `method_missing` extreme late-binding facility.

[Many](http://technology.customink.com/blog/2012/06/18/profiling-openstruct-eager-loading-method-missing-and-lazy-loading/) [words](http://franck.verrot.fr/blog/2015/07/12/benchmarking-ruby-method-missing-and-define-method/) have already been written about performance implications of Ruby's `method_missing`. Even without considering `method_missing` penalties, Celluloid wraps many things onto every method call increasing memory usage significantly.

Consequently, this means that there is a significant performance penalty to using Celluloid in your codebase. Every abstraction has a cost, but in my opinion, Celluloid is worth it in most cases. Even Mike Pernham says so, and he [removed it as a dependency from Sidekiq](http://www.mikeperham.com/2015/10/14/should-you-use-celluloid/).

---
[^1]: [Wikipedia](https://en.wikipedia.org/wiki/Actor_model)
[^2]: Using `included` and `extended` hooks to modify the target class is a well known pattern, an is used in many Ruby libs. There are many more [hooks](http://stackoverflow.com/a/5168554) that you can leverage in your library.
[^3]: [Properties](https://github.com/celluloid/celluloid-essentials/blob/master/lib/celluloid/internals/properties.rb), unlike class-instance vars can be inherited and unlike class variables are invisible to instances.

