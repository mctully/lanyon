---
layout: default
comments: true
title: Using Java completable futures
---

> How much wood would a woodchuck chuck, if a woodchuck could chuck wood?

This post carries on from my previous [Intro to Java completable futures]({{ site.baseurl }}{% post_url 2018-07-08-java-completable-futures-intro %}) post.

The best way of learning how to use completable futures is to create a test project and just mess around with them. I find this a lot easier than trying to definitively understand the docs.

### Hello world

```java
CompletableFuture<String> lFuture = new CompletableFuture<>();

lFuture.complete("hello world");

try {
    System.out.println("GOT A VALUE : " + lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```
In this example, we create a `CompletableFuture`, and then immediately complete it with a value. This means the value is available as soon as we call `get()` on it. The result of this is that it prints `GOT A VALUE : hello world`.

This shows the basics of a `CompletableFuture`, it's something that can yield a value when asked using one of the resolving functions, such as `get()` as used in this example.

If you want to complete a future immediately like this, there is a convenience funtion `completedFuture` which can be used in the same way:

```java
CompletableFuture<String> lFuture = CompletableFuture.completedFuture("hello world");

try {
    System.out.println("GOT A VALUE : " lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```

This has the same output.

### What if things go wrong?

Now consider exceptions. If a future is completed with an exception, aka completed exceptionally, then when `get()` is called, it will throw an exception and so get caught by our exception handler. Here you see the use of `completeExceptionally`

```java
CompletableFuture<String> lFuture = new CompletableFuture<>();
lFuture.completeExceptionally(new Exception("it all went wrong!"));

try {
    System.out.println("GOT A VALUE : " + lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```
This prints `GOT AN EXCEPTION!!!!! ARGH java.lang.Exception: it all went wrong!`. So a `CompletableFuture` can carry either an exception or a value. If it's an exception, when you resolve it the exception is thrown, allowing the calling code to handle it as it wishes.

### Chaining completable futures

Now lets combine processing functions into the mix, using `thenApply()`.


```java
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lFuture = lBaseFuture.thenApply((s) -> s + " said the cow");

lBaseFuture.complete("moo");

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```

Here we create a completable future, and then chain a `thenApply()` to it, which receives the result of the future, transforms it using a lamba, and returns it. The result of `thenApply()` is a completed future with the result of that function.

If we run this example, we get `moo said the cow`.

So `thenApply()` is creating a completed future with the result of its function for us, can we create that future ourselves? Yes, if we use `thenCompose()`.

```java
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lFuture = lBaseFuture.thenCompose((s) -> CompletableFuture.completedFuture(s + " said the cow"));

lBaseFuture.complete("moo");

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```

This has the identical result of printing `moo said the cow`.

`thenCompose()` is more general purpose than `thenApply()`, as being able to explicitly control the future which is passed to the next step allows for the function to choose whether to yield a result immediately (ie a completed future) or to yield a completable future which is perhaps going to wait for some time before completing.

A good use of `thenCompose()` would be if you sometimes needed to do an additional DB read depending on the result of the first DB read. Your `thenCompose()` callback could inspect the result of the first, and either yield a completed future immediately, or kick off a second request and yield that. The original caller still gets to just wait on a single future and this "messiness" of the optional second request is hidden from it.

### More exceptions

What if a function in a completable future chain throws an exception? For example, if a `thenApply()` or `thenCompose()` throws?

Well, due to typing you can't throw *checked exceptions* from the functions specified in `thenApply()` or `thenCompose()`, but you could accidentally cause a *runtime exception*, so let's try that:

```java
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lFuture = lBaseFuture.thenCompose((s) -> {
    throw new RuntimeException("whoops said thenCompose()");
});

lBaseFuture.complete("moo");

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```

It prints `GOT AN EXCEPTION!!!!! ARGH java.lang.RuntimeException: whoops said thenCompose()`

So basically, the exception was caught and automatically converted into an exceptionally completed future, which was passed onto the next stage of the chain, and eventually popped out of our `get()`. It's exactly the same behaviour for `thenApply()`.

How do these exceptions affect subsequent stages of the completable future chain? Does `thenApply()` or `thenCompose()` still get called if the previous stage completes exceptionally?

```java
 CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lFuture = lBaseFuture.thenApply((s) ->
{
    System.out.println("thenApply() has executed");
    return s + " said the cow";
});

lBaseFuture.completeExceptionally(new Exception("failed to moo"));

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```

Prints `GOT AN EXCEPTION!!!!! ARGH java.lang.Exception: failed to moo`. The string `thenApply() has executed` is not printed, ie the `thenApply()` stage is completely bypassed if its previous stage completes exceptionally.

What if you wanted some processing to happen if the previous stage completed exceptionally? That's where `exceptionally()` can be useful:

```java
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lFuture = lBaseFuture.thenApply((s) ->
{
    System.out.println("thenApply() has executed");
    return s + " said the cow";
}).exceptionally((lEx) -> "couldn't moo, so here's a baa");

lBaseFuture.completeExceptionally(new Exception("failed to moo"));

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```

In this case, it prints `couldn't moo, so here's a baa`. So the `thenApply()` stage doesn't execute, but the `exceptionally` stage receives the exception, and can convert it back into a non exceptional result, ie provide a fall back value (which is converted into a completed future for us automatically).

So we can use `thenApply()` to receive a value from the previous stage, and `exceptionally()` to receive an exception from *any* of the previous stages and convert it into a value for the next stage.

What if we wanted to receive either? Perhaps we have some clean up code which should always execute regardless of whether the previous stage had an exception. That's where `handle()` can help:

```java
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lFuture = lBaseFuture.handle((s,ex) -> {
    if (ex != null) {
        return "it went wrong, but here's a moo anyway";
    } else {
        return s + "said the cow";
    }
});

lBaseFuture.completeExceptionally(new Exception("failed to moo"));

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```

This prints `it went wrong, but here's a moo anyway`. So that `handle()` stage executes in both the exceptional and normal cases, and can handle either, and returns a value which is converted into a completed future for the next stage. Note that you cannot pass the exception received by `handle()` to the next stage, as you have to return a value. However, if your function throws an unchecked exception, that will be forwarded onto the next stage as usual. **Be careful with handle() as it's an easy way to accidentally suppress exceptions which really should be propagated.**

Besides `handle()`, there is also `whenComplete()`. `whenComplete()` is similar to handle, in that it executes in both the exceptional and normal completion cases after its previous stage, but `whenComplete()` cannot alter the value to be passed into the next stage after it. It's more of a hook to execute code without affecting the outcome of the chain that follows. Although, if your function passed to `whenComplete()` throws an unchecked exception, that exception is propagated to the next stage.

```java
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lFuture = lBaseFuture.whenComplete((s,ex) ->
{
    if (ex != null) {
        System.out.println("whenComplete() noticed an exception");
    } else {
        System.out.println("whenComplete() noticed a result of " + s);
    }
});

lBaseFuture.completeExceptionally(new Exception("failed to moo"));

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```

Prints:

```
whenComplete() noticed an exception
GOT AN EXCEPTION!!!!! ARGH java.lang.Exception: failed to moo
```

And if the previous stage completed OK:

```java
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lFuture = lBaseFuture.whenComplete((s,ex) ->
{
    if (ex != null) {
        System.out.println("whenComplete() noticed an exception");
    } else {
        System.out.println("whenComplete() noticed a result of " + s);
    }
});

lBaseFuture.complete("moo");

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```
Prints:

```
whenComplete() noticed a result of moo
moo
```

## Quick summary so far:
These methods yield new values for the next stage.
### complete()
Lets us complete a completable future, causing it to yield a value to the next stage.
### thenApply()
Synchronous function that receives the value from the previous stage, and can yield a new value for the next stage. The type of the result of the function used in thenApply() doesn't have to be the same as the type passed into it, for example it could receive a simple string, and yield an object when it returns.
### thenCompose()
Function that receives the value from the previous stage, and can yield a new completable future to the next stage. It's basically the same as `thenApply()` except it returns a future instead of a value.
### completeExceptionally()
Lets us complete a completable future with an exception, which will be passed only onto subsequent stages which handle exceptions, eg `exceptionally()`, `handle()`, or `waitComplete()`.
### exceptionally()
Synchronous function which is only executed in the case of an previous stage completing exceptionally. Can return a value which is passed onto the next stage, effectively swallowing the exception and allowing the rest of the chain to run as if there had not been an exception.
### handle()
Synchronous function which executes in both normal and exceptional cases, and can yield a value for the next stage. `handle()` is like `exceptionally()` in that it also swallows the exception and allows the rest of the chain to run as if there had not been an exception.


## Some terminal functions
These values yield no value after they complete and so can only be used at the end of a completable future chain.
### thenAccept()
This is a terminal function, which receives a value from the previous stage and can execute a synchoronus function to do something with it, but it cannot supply anything to the next stage. It's a bit like `whenComplete()`, except it only executes when the previous stage completes normally, and not if it completed exceptionally.
### thenRun()
Another terminal function. It's a less useful version of `thenAccept()`, which only runs if the previous stage completed normally, but doesn't receive its value.

## Some functions that accept multiple CompletableFutures
### thenCombine()
A non terminal function that takes the input from the previous stage, and also takes a second completable future as a parameter. It execute when both input completable futures complete normally and can synchronously yield a value for the next stage of the chain.

```java
CompletableFuture<String> lAnotherFuture = new CompletableFuture<>();
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lFuture = lBaseFuture.thenCombine(lAnotherFuture,(res1,res2) ->
{
    return "I saw both a "+res1+" and a "+res2+" today";
});

lBaseFuture.complete("cow");
lAnotherFuture.complete("pig");

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```
Prints `I saw both a cow and a pig today`

### thenAcceptBoth()
Basically the same as `thenCombineBoth()`, except it doesn't yield a value for the next stage.

### runAfterBoth()
Like `thenAcceptBoth()`, except it doesn't take the previous stages' results as inputs.

### acceptEither()
Similar to `thenAcceptBoth()`, but it will execute when _either_ of its input futures resolve and will only receive the value of the first one that completes.

E.g.:

```java
CompletableFuture<String> lAnotherFuture = new CompletableFuture<>();
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<Void> lFuture = lBaseFuture.acceptEither(lAnotherFuture,(res1) ->
{
    System.out.println("I saw a "+res1);
});

lAnotherFuture.complete("pig");
lBaseFuture.complete("cow");

try {
    System.out.println(lFuture.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```
Prints:

```
I saw a pig
null
```

The `null` is coming from the printing of the `get()`, as the completable future is of type `Void` in this case (as `acceptEither()` is a terminal method which doesn't return a value).

If the code is edited to complete the futures the other way around, the output instead prints `I saw a cow` followed by `null`.

### runAfterEither()
Like `acceptEither()`, except doesn't take the previous stages' results as inputs.

## Stages that take an arbitrary number of input futures

### allOf()
Takes an array or list of input stages, and completes when they all complete. It doesn't yield any value, so you don't get the results of the input stages passed down. That seems like a bit of an odd restriction, but it can be worked around using some crazy structuring - you need to keep track of the input futures and then do a `get()` on them individually to retrieve each individual result. Each of the input futures can yield a different type.

#### Exceptions and allOf()
`allOf()` will always wait until all input futures are resolved with either a value or an exception. If any of the input futures complete exceptionally, when all of the rest complete, the `allOf()` future will yield the exception from the first inputted future which completed exceptionally. For example:

```java
CompletableFuture<String> lBaseFuture = new CompletableFuture<>();
CompletableFuture<String> lBaseFuture2 = new CompletableFuture<>();
CompletableFuture<Void> lAll = CompletableFuture.allOf(lBaseFuture,lBaseFuture2);

System.out.println("completing base future 2");
lBaseFuture2.completeExceptionally(new Exception("block2"));

System.out.println("completing base future 1");
lBaseFuture.completeExceptionally(new Exception("block1"));
lBaseFuture.complete("raw");


try {
    System.out.println(lAll.get());
} catch (Exception lExp) {
    System.out.println("GOT AN EXCEPTION!!!!! ARGH " + lExp.getMessage());
}
```

Prints:

```
completing base future 2
completing base future 1
GOT AN EXCEPTION!!!!! ARGH java.lang.Exception: block1
```

So even though `lBaseFuture2` exceptioned before `lBaseFuture1`, it was the exception from `lBaseFuture1` which was yielded.


### anyOf()
This is like `allOf()` except it completes when _any_ of its inputs complete. This one actually then completes with the value of the input that completed, so can be used to "race" inputs and operate on the result of the first that completed.

## Further reading
If you've looked at the completable futures API you may have noticed that pretty much all the methods mentioned so far also have "async" variants, for example `thenAcceptAsync()` or `whenCompleteAsync()` etc. The async suffix here refers to where the supplied callback will execute. The non-async versions execute the callback on the thread which is invoking the `complete()` or `completeExceptionally()` method.

Non-async is fine for short callbacks which do very little, but if your callback might end up doing some heavy lifting, or blocking, then it might be better to execute the callback on another thread - hence the provision of the async variants.

By default, the async variants will execute on a general purpose thread pool called the _fork join_ pool. This is essentially a thread pool that you can use without caring about too much. If you're going to do blocking or really heavy lifting in the callbacks, then the async variants also allow you to pass an `Executor` through which allows you to directly control which threads the callbacks happen on.

In all honesty I don't tend to use the async variants as I don't tend to use callbacks for anything heavy; I prefer to keep the logic in the task state machines instead. The main reason is that debugging completable futures is a bit of a pain. Breakpointing individual callbacks is ok, but stepping through them is difficult, so I find it's best to keep callbacks simple and concise.

## Fin
Hopefully this will help people on their completable future journey. If I had to make one suggestion to help with getting to grips with completable futures, it would be to make a simple test project and just write little test cases like I have used here, you won't regret it.
