---
layout: post
title: "JSON, Aeson and Template Haskell for fun and profit"
tags: haskell json
comments: true
published: true
---

At times, handling JSON in Haskell might seem difficult, but once you get the hang of it, you will in fact see it is not.

There are a few libraries that parse and encode JSON in Haskell, but recently one particular has gained a lot of popularity, and for a good reason. The library in question is Bryan O'Sullivan's [^1] Aeson.

[Aeson](http://hackage.haskell.org/package/aeson) performs better than its older relative [JSON](http://hackage.haskell.org/package/json), all while simplifying the implementation of encoding/decoding functions by using Template Haskell.

Since all this imposing knowledge came out of an effort to build a *Meetup* API client library [^2] in Haskell, we'll be dealing with *Meetup* entities in the following examples. Consequently, we have an ```Event``` type that was obtained by querying the */2/events*  API endpoint (example [response](https://gist.github.com/neektza/f76e8a44267a669f564a#file-event-json)), and an ```RSVP``` type, which we get from the */2/rsvps* API endpoint (example [response](https://gist.github.com/neektza/f76e8a44267a669f564a#file-rsvp-json)).

# Hard labour

The regular way to parse JSON with Aeson is to implement a ```FromJSON``` instance for the data type we want to build from JSON. It tells Aeson what JSON fields to pluck from and how to translate those fields to a Haskell construct. Inversely, if we wanted to encode a type to JSON, we'd have to implement a ```ToJSON``` instance.

Here's an example of how to manually implement a ```FromJSON``` instance for an ```Event``` type.

{% gist neektza/f76e8a44267a669f564a Event.hs %}

If you are interested, you can load the file into the GHCi REPL with ```$ ghci src/Types/Event.hs``` and see what the ```sampleEvent``` spits out.

As you can see, following the ```Event``` type definition, there's a definition of the ```parseJSON``` function, and it looks like the ```parseJSON``` definition could be categorized as a boilerplate code. Why? Because each time we add or remove a record field in the ```Event``` data constructor, we also need to change the ```parseJSON``` definition accordingly.

There is more. If the example above contained a ```ToJSON``` definition, we'd have to change it as well. Now, stretch your imagination for a second and imagine we had more than one type. This situation could get out of hand very quickly and even though the type system can inform us of it, it would still be annoying.

# Making the GHC work for you

Feeling tired after all the hard work we had to do previously, it kind of makes us wonder if there's an easier way to do this... Turns out there is, and it's called Template Haskell (called TH by friends).

I won't go into the *whats* and the *hows* of TH (mainly since I myself don't completely understand it yet), but you can think of it as Lisp's macro system but with types. Because everything is better with some types.

Long story short, using the ```Data.Aeson.TH``` package we can make the GHC define the FromJSON and ToJSON instances at compile-time for us, and this is how:

{% gist neektza/f76e8a44267a669f564a RSVP.hs %}

As before, you can load up the code into GHCi with ```$ ghci src/Types/Event.hs``` and check the result by calling ```sampleRSVP```.

We can see in the example above that, even though there is no ```fromJSON``` definition, we're still able to decode the ```ByteString``` to JSON. We still need to import ```Data.Aeson``` to have access to the ```decode``` function.

Not much convincing is needed to see that this approach is much better than the previous one. Once ```Event``` or ```RSVP``` types change, GHC changes the ```FromJSON``` and ```ToJSON``` definitions at our demand.

# Conclusion

Reduce boilerplate wherever possible, since the tools to do it are already there. You'll be happier and your general well-being will improve.

In some future post I'll try to figure out Template Haskell and explain it in an approachable way.

P.S.

If anyone can explain what ["A continuation-based parser type."](https://hackage.haskell.org/package/aeson-0.7.0.3/docs/Data-Aeson-Types.html#t:Parser) means, I'd be happy to hear it out.

---
[^1]: You should also check out the awesome [Wreq](http://hackage.haskell.org/package/wreq) HTTP library he made.
[^2]: You can find it over at [Github](https://github.com/neektza/hs_meetup)
