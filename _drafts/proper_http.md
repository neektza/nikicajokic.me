# Building well defined APIs, part 1: HTTP

I've been playing with **Clojure** (on and off) for a while now, trying to switch over from **Ruby** so, naturally, I wanted to produce something tangible in it. Since writing REST APIs takes most of my regular working day, I wanted to see how it feels to build one in Clojure. As it usually is in programming, there are many ways to do it, and although most of them are nice, one in particular stuck. But I'll get to that.

## The conventional way

The usual set of libraries when building HTTP APIs in **Clojure** is **Ring** for middleware management and **Compojure** for routing, along with other domain specific libraries for various other things (database access, messaging, caching, etc.).

Similarly, in **Ruby**, you have **Rack** for middleware and **Sinatra** for routing (Ring and Compojure were inspired by these two, coincidentally), or if you're don't mind relinquishing control you would use **Rails** which does everything (routing, caching, database access etc.)



If you were to build an API in this way, you would have your routes mapped to various handler functions/methods. For example, the `GET /api/posts` endpoint would be mapped to a `fetch_and_render_posts` function in which you would then fetch data from the database and render that data to JSON. In other words, nothing fancy.

Halfway through the process of building this pet project, I started using a very good reference book called [Web development with Clojure](https://pragprog.com/book/dswdcloj/web-development-with-clojure). In it there is a part about building services with a library called [Liberator](https://github.com/clojure-liberator/liberator), so naturally I wanted to see what was it about.

After a bit of research and trial-and error I came to a conclusion that the way I used to build APIs was completely broken, at least on the protocol (HTTP) level.

## A way in which you use a state machine

Liberator does things a bit differently - it encodes all the possible states of the HTTP protocol into a state machine and provides you with hooks by which you can define the transitions, i.e. control the behavior of the state machine. It was inspired by the Erlang's `webmacihine` library (so much so that you could call it a port), and there is even a Ruby version called `webmachine-ruby`.

So, as I said, you control what happens inside the state machine by defining functions that decide on transitions. Let's see an example of that:

```
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
    response.body = db.save(id, RestApiModel::Status.new(params.merge({id: id})))
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
    @db ||= RestApiModel.db(RestApiModel::Status)
  end
end

```

The `method_allowed?` function checks if the request coming in is a `GET` and returns either a truthy or a falsey value. In the truthy case, the state machine is allowed further processing. Otherwise, the processing stops and Webmachine reponds with an "405 Method Not Allowed" HTTP reponse code.

There are many such methods which you can override and define behavior. You can find the entire decision graph [here](http://for-get.github.io/http-decision-diagram/httpdd.fsm.html)

As you can see, this is a fundamentally different approach. Why? Well, there are number of reasons:

1. You craft responses on a semantically higher level, because the language of the HTTP protocol is baked into the your code.

2. The framework guides you instead of providing defaults for the common cases and hiding the uncommon ones. The entire decision process is transparent and every state documented.

3. Bla bla.

## You should use this when ...

Since webmachine is not as simple tool as Sinatra or Rails, you probably shouldn't be using webmachine always and for everything. When I say it's not simple I primarily mean that most of the things you get for free with Rails


---
[^1] A while ago I presented about this topic on [one of the CanJS community hangouts](http://youtu.be/AJUZOefv2NE?t=52m). 
