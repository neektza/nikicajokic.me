---
layout: post
title: "JSON, Aeson and Template Haskell for fun and profit"
tags: haskell json
comments: true
---

At times, handling JSON in Haskell might seem difficult, but you will definitely change your mind once you get the hang of it.

There are a few libraries that parse and encode JSON in Haskell, but one specifically gained a lot of popularity recently, and for good reason. The library in question is Bryan O'Sullivan's [^1] Aeson.

[Aeson](http://hackage.haskell.org/package/aeson) performs better than it's older relative [JSON](http://hackage.haskell.org/package/json), all while simplifying implementation of encoding/decoding functions by using Template Haskell.

Since all this imposing knowledge came out of an effort to build a *Meetup* API client library [^2] in Haskell, we'll be dealing with *Meetup* entitities in the following examples. Consequently, we have an ```Event``` type, that we get by querying the */2/events*  API endpoint (example [response](https://gist.github.com/neektza/d50ee5f749f985d65412#file-event-json)), and a ```RSVP``` type, which we get from the */2/rsvps* API endpoint (example [response](https://gist.github.com/neektza/d50ee5f749f985d65412#file-rsvp-json)).

# Hard labour

The regular way to parse JSON with Aeson is to implement a ```FromJSON``` instance for the data type we want to build from JSON. It tells Aeson what fields to pluck from JSON and how to translate those fields to a Haskell construct. Inversely, if we wanted to encode a type to JSON, we'd have to implement a ```ToJSON``` instance.

Here's an example of manually implementing a ```FromJSON``` instance for an ```Event``` type.

{% gist neektza/d50ee5f749f985d65412 Event.hs %}

You can load the file into the GHCi REPL with ```$ ghci src/Types/Event.hs``` and see what the ```sampleEvent``` spits out, if you're interested.

As you can see, following the ```Event``` type definition, there's a definition of the ```parseJSON``` function, and it looks like the ```parseJSON``` definition could be categorized as boilerplate code. Why? Because each time we add or remove a record field in the ```Event``` data constructor, we also need to change the ```parseJSON``` definition accordingly.

Not only that, but if there had been a ```ToJSON``` definition in the example above, we'd have to change it as well Now stretch your imagination for a second and imagine if we had more than one type. It seems that this could get out of hand very quickly (luckily, we have the type system to warn us about that, but it would still be annoying).

# Making the GHC work for you

Feeling tired after all the hard work we had to do previously, it kinda of made us wonder if there's an easier way to do this... Turns out there is, and it's called Template Haskell (called TH by friends).

I won't go into the *whats* and the *hows* of TH (mainly since I myself don't completely understand it yet), but you can think of it as Lisp's macro system but with types, because everything is better with some types.

Long story short, using the ```Data.Aeson.TH``` package we can make the GHC define the FromJSON and ToJSON instances at compile-time for us, and this is how:

{% gist neektza/d50ee5f749f985d65412 RSVP.hs %}

As before, you can load up the code into GHCi with ```$ ghci src/Types/Event.hs``` and check the result by calling ```sampleRSVP```.

We can see in the example above that, even though there is no ```fromJSON``` definition, we're still able to decode the ```ByteString``` to JSON. We still need to import ```Data.Aeson``` to have access to the ```decode``` function.

Not much convincing is needed to see that this approach is much better than the previous one. Once ```Event``` or ```RSVP``` types change, GHC changes the ```FromJSON``` and ```ToJSON``` definitions for us.

# Conclusion

Reduce bolierplate wherever possible, since the tools to do it are already there. You'll be happier and your general well-being will improve.

In some future post I'll try to figure out Template Haskell and explain it in an approachable way.

P.S.

If anyone can explain what does ["A continuation-based parser type."](https://hackage.haskell.org/package/aeson-0.7.0.3/docs/Data-Aeson-Types.html#t:Parser) mean, I'd be happy to hear it out.

---
[^1]: You should also check out the awesome [Wreq](http://hackage.haskell.org/package/wreq) HTTP library he made.
[^2]: You can find it over at [Github](https://github.com/neektza/hs_meetup)
