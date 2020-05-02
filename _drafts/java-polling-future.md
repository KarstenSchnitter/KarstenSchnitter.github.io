---
layout: post
title: "A Polling Java Future"
date: 2020-04-25 13:48:00 +0200
categories: what-i-learned
tags: java concurrency future
---
There are times in Java, when we want to act on a state transition in some object.
Ideally, the object allows to register a listener, which is called when the state changes.
Unfortunately, this is not always the case, especially when the class stems from a third-party library and is outside our control.
In this case, we need to poll the state regularly, to detect the change we are interested in and then invoke the callback we want.
The [Java concurrency library](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/package-summary.html) offers a neat abstraction with the [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html).
In this post we try to wrap the polling into such a future, that can be used by a wide variety of frameworks, especially for reactive programming.

As a starting point we need to have an object, that we want to observe.
Let us assume, this object implements [org.springframework.context.Phased](https://github.com/spring-projects/spring-framework/blob/master/spring-context/src/main/java/org/springframework/context/Phased.java). Essentially, this interface looks like this:

```java
public interface Phased {
    int getPhase();
}
```

We want to create a [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html), that gets completed, when our observed phase has a certain value, say 1 for this post.
So we want to create some implementation for the following method:

```java
public <P extends Phased> CompletableFuture<P> awaitPhaseOne(P observable) {
    // poll observable until observable.getPhase() == 1
    return CompletableFuture.failedFuture(
        new IllegalStateException("not yet implemented")
    );
}
```

For now, we just return a failed future until we have the correct implementation.
But let us have a look at the message signature first: The function returns a future, that potentially wraps the observed object.
This allows consumers to execute further operations on this object:

```java
awaitPhaseOne(observable).thenApply(o -> doSomething(o));
```

This return type is not the only possibility, it just serves as a useful example.

Let us now have a look, how we can use [java.util.concurrent](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/concurrent/package-summary.html) to implement our polling future.
We will need three ingredients:

* the [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html), that we want to create
* a [Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html), that does the actual polling
* a [ScheduledExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html), that executes the Runnable

Putting it all together we arrive at the following implementation.

```java
public <P extends Phased> CompletableFuture<P> awaitPhaseOne(P observable) {
    ScheduledExecutorService executor =
        Executors.newSingleThreadScheduledExecutor();
    CompletableFuture<P> future = new CompletableFuture<>();
    ScheduledFuture<?> scheduled = executor.scheduleAtFixedRate(
        // the Runnable executed in each polling run
        () -> { 
            if (observable.getPhase() == 1) {
                future.complete(observable);
            }
        },
        100, // initial delay
        200, // period between executions
        TimeUnit.MILLISECONDS  // time unit
    );
    future.whenComplete((obs, thrown) -> scheduled.cancel(true));
    return future;
}
```

We start with creating a new `executor`.
This is a very simplistic approach, that we will revisit later.

Next we create the `future`, that is our desired [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html).
We have not checked our `observable` yet, so the future is not completed.

The polling is now started by scheduling a [Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html) with the `executor`.
We use an anonymous lambda function, that checks the phase of our `observable`.
If we find the desired phase 1, we complete the `future`.
The runnable is scheduled to start polling after 100 milliseconds with an interval of 200 milliseconds.
Of coures, these values will be configurable in a production setting.

Before we return the future we need to ensure, that the polling is stopped once the future is completed.
This is done with a small callback using `future.whenComplete`.

Our implementation separates nicely the code that is testing the `observable` from the polling infrastructure.
The underlying design is a closure consisting of the [Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html) accessing both the `observable` and the [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html).
A [ScheduledExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html) allows scheduling with a fixed rate, as we used in our implementation or scheduling with fixed delay in between executions.
More sophisticated schedules like for example exponential back-off require custom scheduling of the runs.

Let us come back to the creation of the [ScheduledExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html): With the code above, a new instance is created for every call to the function `awaitPhaseOne()`.
This implementation will spawn a new thread during each invocation.
If it is called at high frequency, this is a severe performance issue.
We can solve this issue by using a globally managed [ScheduledExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html) either in the surrounding class or from some other part of our application.

Another optimization is to check the polling condition before scheduling the [Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html) and return immediately.
Combining this pre-check with an extracted [ScheduledExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html), we get the following implementation:

```java
public class PollingFuture {

    private final ScheduledExecutorService executor;

    public PollingFuture(int corePoolSize) {
        this.executor = Executors.newScheduledThreadPool(corePoolSize);
    }

    public <P extends Phased> CompletableFuture<P> awaitPhaseOne(P observable) {
        if (isPhaseOne(observable)) {
            return CompletableFuture.completedFuture(observable);
        }
        CompletableFuture<P> future = new CompletableFuture<>();
        ScheduledFuture<?> scheduled = executor.scheduleAtFixedRate(
            // the Runnable executed in each polling run
            () -> {
                if (isPhaseOne(observable)) {
                    future.complete(observable);
                }
            }, 100, // initial delay
            200, // period between executions
            TimeUnit.MILLISECONDS // time unit
        );
        future.whenComplete((obs, thrown) -> scheduled.cancel(true));
        return future;
    }

    private <P extends Phased> boolean isPhaseOne(P observable) {
        return observable.getPhase() == 1;
    }
}
```

Note, that the first check of the polling condition is now executed during the function invocation in the main thread.
If this was a long-running execution, this early check may hurt performance more, than what can be gained by not scheduling the [Runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html).
The whole logic on when to complete the [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) is now extracted to the private function `isPhaseOne(...)`.
Everything else is the glue code to create the polling future.

In summary, we developed a small class, that allows to wrap polling for some condition into a [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html).
We have seen, that handling a [ScheduledExecutorService](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledExecutorService.html) is required for the implementation.
The resulting Future can be used in different asynchronous applications, e.g. with [Spring WebFlux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html) a reactive stack for building web applications.
