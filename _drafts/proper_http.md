---
layout: post
title: "Building well defined APIs, part 1: HTTP"
tags: REST HTTP ruby clojure
comments: false
---

I've been playing with **Clojure** (on and off) for a while now, trying to switch over from **Ruby** so, naturally, I wanted to produce something tangible in it, so I set my mind to building a simple CMS API. Since I'm primarily a back-end developer, exposing REST APIs takes up a considerable chunk of my working day, I wanted to see how it feels to build one in Clojure. As it usually is in programming, there are many ways to do it, and although most of them are nice, one in particular stuck. But I'll get to that[^1].

## The conventional way

The usual set of libraries you would use when building HTTP APIs in **Clojure** is **Ring** for middleware management and **Compojure** for routing, along with other domain specific libraries for various other things (database access, messaging, caching, etc.).

Two analogous libraries in **Ruby** would be **Rack** for middleware and **Sinatra** for routing (Ring and Compojure were inspired by these two, coincidentally), or if you're don't mind relinquishing control you would use **Rails** which does everything (routing, database access, rendering etc.)

If you were to build an API in this way, you would have your routes mapped to various handler functions/methods. For example, the `GET /api/posts` endpoint would be mapped to a `fetch_and_render_posts` function in which you would then fetch data from a data store of some kind and render that data to JSON. In other words, nothing fancy.

Halfway through the process of building this pet project, I started using a very good reference book called [Web development with Clojure](https://pragprog.com/book/dswdcloj/web-development-with-clojure). In it there is a part about building services with a library called [Liberator](https://github.com/clojure-liberator/liberator), so naturally I wanted to see what was it about.

After a bit of research and trial-and error I came to a conclusion that the way I used to build HTTP APIs was not as good as I thought it to be, at least on the protocol level.

The main problem of this conventional approach is that it's up to you to know and decide how to map various resource states into proper HTTP status codes. For example, you have to waste brain cycles to decide which status code to respond with if the validation function of the resource decides that the resource is invalid.

## A way in which you use a state machine

Liberator does things a bit differently - it encodes all the possible states of the HTTP protocol into a state machine and provides you with hooks by which you can define the transitions, i.e. control the behavior of the state machine. It was inspired by the Erlang's `webmacihine` library (so much so that you could call it a port), and there is even a Ruby version called `webmachine-ruby`.

So, as I said, you control what happens inside the state machine by defining functions that decide on transitions. Let's see an example of that:

{% highlight ruby linenos %}
require 'resources/json_resource'

class StatusResource < JSONResource

  def allowed_methods
    %w(GET PUT DELETE)
  end

  def resource_exists?
    resource
  end

  def delete_resource
    db.delete(id)
  end

  def from_json
    response.body = db.save(id,
	  RestApiModel::Status.new(
	    params.merge({id: id})))
  end

  def to_json
    db.find(id).to_json
  end

  private
  
  def resource
    db.find(id)
  end

  def id
    request.path_info[:id].to_i
  end
  
  def params
    JSON.parse(request.body.to_s)
  end

  def db
    @db ||= RestApiModel.db(
	  RestApiModel::Status)
  end
end
{% endhighlight %}

The `method_allowed?` function checks if the request coming in is a `GET` and returns either a truthy or a falsey value. In the truthy case, the state machine is allowed further processing. Otherwise, the processing stops and Webmachine responds with an "405 Method Not Allowed" HTTP response code.

Have you ever come across a Rails codebase that uses that particular response code? I know I haven't. What you would usually get back from most APIs is (best case) a 404 response. This way, you know that you're hitting the right resource (and indeed that the resource exists), but you also know that that particular operation is not supported over the resource you're trying to interact with.

There are many such methods which you can override and define behavior. You can find the entire decision graph [here](http://for-get.github.io/http-decision-diagram/httpdd.fsm.html)

As you can see, this is a fundamentally different approach. Why? Well, there are two reasons and benefits I can think of:

1. You craft responses on a semantically higher level, because the language of the HTTP protocol is baked into the your code.

2. The framework guides you instead of providing defaults for the common cases and hiding the uncommon ones. The entire decision process is transparent and every state documented.

## You should use this when ...

Since webmachine is not as simple tool as Sinatra or Rails, you probably shouldn't be using webmachine always and for everything. When I say it's not simple I primarily mean that most of the things you get for free with Rails


---
[^1]: If you would prefer to watch a video, I presented about this topic on [one of the CanJS community hangouts](http://youtu.be/AJUZOefv2NE?t=52m). 
