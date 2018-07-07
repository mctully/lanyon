---
layout: default
comments: true
title: Intro to Java completable futures
---

> Well, what possible harm could one insane, mutant tentacle do?

I wouldn't consider myself a Java guy. I've used lots of languages over the years, and Java makes as much sense as any other - but there's plenty I don't know about it. One of the perks of having used lots of other languages is that you know when there's a gap in your knowledge - and that can lead to finding something new. That's pretty much how I came across _completable futures_.

I found the docs a little bit impenetrable, which is why I spent some time just experimenting. This is the write up of those experiments, which I hope might be useful to others.

## A little background

First let me explain the problem that completable futures can help with.

Let's say you have a thread in your Java server that is responding to a network request. It needs to make a DB read in order to complete the request and this might take "some time". Your thread can't complete its request and reply to the client until the DB read is complete, so now you have a choice, do you want to be _synchronous_ or _asynchronous_. Put another way, do you want to block your thread or not?

To make sure everyone is along for the ride, let's quickly explain the two options.

**I shall be synchronous!**

This means that your thread would just call the DB read method and wait for it to return. This is really simple, and keeps your code nice and easy to understand. Most typically the DB read will result in some sort of I/O operation, which will lead to your thread becoming blocked at the OS level, and so the OS will happily park it until the I/O is complete and use the freed up CPU core to do something else.

If you want that something else to be "processing more of your requests", then you will have to spawn some more threads, as the thread you were using is now occupied, waiting until that pesky DB I/O completes.

The number of concurrent requests your server can process is now limited by the number of threads you're willing / able to spawn.

This isn't an ideal scenario. Threads take memory for a start. Also, the scheduling of threads is the OS's job - your server is no longer fully in control of which responses should complete first when there's a lot concurrent requests in progress. If this becomes a problem, then you're stuck.

**Ok, I shall be asynchronous**

Ok, let's not do the blocking thing. We want our request thread to be able to go and do something more useful whilst the DB I/O is going on. How do we do this?

Some languages have solutions with names like _green threads_, _fibres_ or _co-routines_. They all pretty much do the same thing, which is to allow you to write synchronous looking code that secretly reuses the OS thread whenever it becomes blocked by I/O. Java doesn't offer this, it only offers you a full on OS thread; so if you want to reuse your thread you're going to do a bit of manual work and your code isn't going to look like synchronous code any more.

First, you can't call that blocking DB read anymore. Hopefully there's some sort of asynchronous version of the DB read that your thread can use instead; maybe one that will call a callback when the DB read has completed. If not, you can build one yourself by using a thread pool.

In either case, your thread now has to be re-written to use _state machines_. You re-write your request to be a state machine, and your thread runs it until it needs to make the DB read, which it then kicks off asynchronously. It then puts the request state machine into a "waiting on DB" state, pops it onto a list of state machines that are blocked and moves onto the next state machine. This way your thread keeps working instead of simply being parked by the OS for being blocked on I/O. When the DB read completes, the callback fires and flags the state machine as being able to continue and your thread can pick it up when it's ready to do so.

This tears up your nice synchronous code quite a bit, and makes it a harder to follow, but your request throughput will go up, which is usually the important thing.

### Completable futures help with async coding

To make things less painful, completable futures can help you out. They provide a standardised way to pass results around between bits of asynchronous code. They also provide useful concepts like "only call this callback when *all* of these asynchronous results arrive", and standard ways of passing exceptions back too.

Put very simply a completable future is just a place to store a result and have a callback called when the result is stored.

My [next post]({{ site.baseurl }}{% post_url 2018-07-09-using-java-completable-futures %}) goes into how to use completable futures.
