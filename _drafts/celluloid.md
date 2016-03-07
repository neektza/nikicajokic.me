
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

