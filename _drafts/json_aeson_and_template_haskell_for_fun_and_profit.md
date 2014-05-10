---
layout: post
title: "JSON, Aeson and Template Haskell for fun and profit"
tags: haskell json
---

Handling JSON can look difficult in Haskell at times, but it's not so, once you get the hang of it (as with everything really).

There are a few libraries that parse and encode JSON in Haskell, but one specifically gained a lot of popularity recently, and for good reason. The library in question is Bryan O'Sullivan's [^1] Aeson.

[Aeson](http://hackage.haskell.org/package/aeson) performs better than it's older relative [JSON](http://hackage.haskell.org/package/json), all while simplifying creation of parsing/encoding functions by using Template Haskell.

Since all this awesome knowledge came out of an effort to build a *Meetup.com* client library [^2] in Haskell, we'll be dealing with Meetup.com entitities. Consequently, we have an ```Event``` type, that we get by querying *[/2/events](https://gist.github.com/neektza/d50ee5f749f985d65412#file-event-json)* API endpoint, and a ```RSVP``` type, which we get from the *[/2/rsvps](https://gist.github.com/neektza/d50ee5f749f985d65412#file-rsvp-json)* API endpoint.

# Hard labour

First, here's an example of manually implementing a FromJSON typeclass for an Event type.

{% gist neektza/d50ee5f749f985d65412 Event.hs %}

You can load the file into the GHCi REPL with ```$ ghci src/Types/Event.hs``` and see what the ```sampleEvent``` spits out, if you're interested.

As you can see, following the ```Event``` type definition, there's a definition of the ```parseJSON``` function, and it looks like the ```parseJSON``` definition could be categorized as boilerplate code. Why? Because each time we add or remove a record field in the ```Event``` data constructor, we also need to change the ```parseJSON``` definition accordingly.

Not only that, but if there had been a ```ToJSON``` definition in the example above, we'd have to change it also. Now stretch your imagination for second and imagine if we had more than one type. It seems this could get out of hand very quickly (luckily, we have the type system to warn us about that, but it would still be annoying).

# Making the GHC work for you

Feeling tired after all the hard work we had to do previously, it kinda' made us wonder if there's an easier way to do this... Turns out there is, and it's called Template Haskell (called TH by friends).

I won't go into the _whats_ and the _hows_ of TH (mainly since I myself don't completely understand it yet), but you can think of it as Lisp's macro system but with types, because everything is better with some types.

Long story short, using the ```Data.Aeson.TH``` package we can make the GHC define the FromJSON and ToJSON instances at compile-time for us, and this is how:

{% gist neektza/d50ee5f749f985d65412 RSVP.hs %}

As before, you can load up the code into GHCi with ```$ ghci src/Types/Event.hs``` and check the result by calling ```sampleRSVP```.

What we can see in the example above, is even though there is no ```fromJSON``` definition, we're still able to decode the ByteString to JSON. We still need to import Data.Aeson to have access to the ```decode``` function.

Not much convincing is needed to see that this approach is much better than the previous one. Once the Event or the RSVP types change, GHC changes the FromJSON and ToJSON definitions for us.

---
[^1]: He also made the awesome [Wreq](http://hackage.haskell.org/package/wreq) HTTP library.
[^2]: You can find it over at [Github](http://github.com/neetkza/meetup_hs)
