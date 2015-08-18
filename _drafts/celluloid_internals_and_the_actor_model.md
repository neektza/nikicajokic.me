---
layout: post
title: "Celluloid internals and the Actor model"
tags: concurrency ruby actors celluloid
class: article
comments: false
---

Continuing from previous two posts about [primitives and abstractions](link) in Ruby and the [reactor pattern](link), in the last part of the series we'll cover handpicked internals from the Celluloid library, which is a Ruby implementation of the Actor model, and explain the core ideas behind it.

## The Actor model

First, let's define what the *Actors* actually are.

> An actor is a computational entity that, in response to a message it receives, can concurrently:
> 
> * send a finite number of messages to other actors;
> * create a finite number of new actors;
> * designate the behavior to be used for the next message it receives.[^1]

This paragraph is saying is that an Actor model is a system inside of which computation is done only by sending messages. This is an ideal abstraction. Actual implementations of this model rarely limit themselves on doing computation only in this way. This is also the case with _Celluloid_.

The actor system is somewhat similar to the regular object system, the main difference being that in object based system everything done sequentially in contrast to actor based system where everything is done concurrently. In that regard, the actor model more accurately describes the real world because in the real world, everything happens concurrently.

In order to understand the Actor model implementation in Ruby we'll analyze several of the main classes where most of Celluloid's interesting ideas reside.

## Transforming objects into actors

When we want to turn an object into an actor, we include the Celluloid module into the target class. Once the `Celluloid` module is included, an `included` hook gets called and the transformation process begins [^2].

{% highlight ruby %}

 included(klass)

{% endhighlight %}

First, the target class gets extended with Celluloid's class and instance methods.

Next, the target class gets inheritable properties which are a cross between class-instance variables and class variables [^3]. Celluloid uses them to save configuration settings on the class level.

Finally, the **most significant steps** of the transformation process happen. The class gets three properties that turn it something that produces actors instead of regular objects:

1. `mailbox_class` property,

2. `proxy_class` property,

3. `task_class` property.

These three properties make up the foundational parts of the library and we'll have to cover each in detail if we hope to understand how Celluloid works. But first, we have to understand how actors are created.

Since Celluloid modifies the way objects are created it must do something with constructor method. Sho 'nuff, replaced with Celluloid's own. This constructor method, instead of producing regular objects, produces Cells.

## Cell

Cell is a Builder class that responsible for building _proxies_ which are objects that intercept regular method calls and convert them to something Celluloid calls **inter-actor message protocol**..

That protocol boils down to intercepting method calls in `method_missing`, wrapping them as `Call` objects and pushing them onto a `Mailbox` which we'll cover later.


{% highlight ruby %}
  class SyncProxy < AbstractProxy
    # ...
    def method_missing(meth, *args, &block)
    unless @mailbox.alive?
      raise DeadActorError, "attempted to call a dead actor"
    end
    if @mailbox == ::Thread.current[:celluloid_mailbox]
      args.unshift meth
      meth = :__send__
    end
    call = SyncCall.new(::Celluloid.mailbox, meth, args, block)
    @mailbox << call
    call.value
  end
end
{% endhighlight %}

### Call objects

`Call` objects represent method calls inside celluloid. `SyncCall` and `AsyncCall` objects represent synchronous and asynchronous method calls respectively.

## Actor

As expected, the Actor is a basic building block of Celluloid (quote from the docs):

> Actors are Celluloid's concurrency primitive. They're implemented as
normal Ruby objects wrapped in threads which communicate with asynchronous
messages.

So actor is just an object. But, what separates an actor from a regular object? Well, a few things, but most importantly, the actor has something called an "actor loop". The loop is responsible for listening to incoming messages and handling them. It's an infinite loop very similar to the event loop we covered in the last post about [EventMachine internals](/eventmachine_internals_and_the_reactor_pattern).

{% highlight ruby linenos %}

# Run the actor loop
def run
  while @running
    begin
      @timers.wait do |interval|
        interval = 0 if interval and interval < 0
        if message = @mailbox.check(interval)
          handle_message(message)
          break unless @running
        end
      end
    rescue MailboxShutdown
      @running = false
    end
  end
  shutdown
rescue Exception => ex
  handle_crash(ex)
  raise unless ex.is_a? StandardError
end

{% endhighlight %}

## Mailbox

An actor uses the mailbox to receive and store messages from other actors.

Implementation of the Mailbox is relatively straightforward. Events are added while a mutex is active to prevent race conditions and make sure the events  If the mailbox is dead or is full, the message is discarded. System events get added in front of the message queue, while regular events go at the back.

{% highlight ruby linenos %}
def <<(message)
  @mutex.lock
  begin
    if mailbox_full || @dead
      dead_letter(message)
      return
    end
    if message.is_a?(SystemEvent)
      @messages.unshift message
    else
      @messages << message
    end
	
    @condition.signal
    nil
  ensure
    @mutex.unlock rescue nil
  end
end
{% endhighlight %}

## Proxy

When we instantiate objects of the target class we don't actually get an instance of that class. Instead, we get a proxy object which act as an interface to the underlying object by intercepting all calls to it. This is where the bypassing of Ruby's messaging protocol actually happens, since the object never receives messages directly.

## Task

Tasks are used to resume and suspend method execution and by default Ruby's fibers are used to represent tasks. This is what enables code written using Celluloid to look like synchronous code, even though asynchronous operations are happening under the hood.

Tasks, as we said before are by default backed by Fibers. The implementation is relatively straightforward.

---

<div id="funnelFormContainer"></div>
<script type="text/javascript" src="//funnelnow.com/widget/361"></script>

---
[^1]: Wikipedia
[^2]: Using `included` and `extended` hooks to modify the target class is a well known pattern, an is used in many Ruby libs. There are many more [hooks](http://stackoverflow.com/a/5168554) that you can leverage in your library.
[^3]: Properties, unlike class-instance vars can be inherited and unlike class variables are invisible to instances.

