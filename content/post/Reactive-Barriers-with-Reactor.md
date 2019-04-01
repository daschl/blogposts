+++
date = "2019-04-01"
draft = false
title = "Reactive Barriers with Reactor"
description = "Non-blocking barriers with Reactor as an alternative to a CountDownLatch"
tags = ["java", "reactive", "reactor"]
slug = "Reactive-Barriers-with-Reactor"
+++

In the non-reactive java world, if you need a couple threads waiting on a [barrier](https://en.wikipedia.org/wiki/Barrier_(computer_science)) to move forward together, a common approach is to use a [CountDownLatch](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html).

Here is how the documentation describes it succinctly:

> A CountDownLatch is initialized with a given count. The await methods block until the current count reaches zero due to invocations of the countDown() method, after which all waiting threads are released and any subsequent invocations of await return immediately. 

To illustrate this concept, the following code shows two threads simulating work (by sleeping) and only proceeding once both have reached the barrier:

```java
final CountDownLatch barrier = new CountDownLatch(2);

final Supplier<String> now = () -> LocalDateTime.now().format(DateTimeFormatter.ISO_DATE_TIME);

new Thread(() -> {
    System.out.println(now.get() + ": Thread 1 starting work...");

    try {
        Thread.sleep(1000);

        barrier.countDown();
        barrier.await();
    } catch (Exception e) { }

    System.out.println(now.get() + ": All finished and moving on.");
}).start();

new Thread(() -> {
    System.out.println(now.get() + ": Thread 2 starting work...");

    try {
        Thread.sleep(5000);

        barrier.countDown();
        barrier.await();
    } catch (Exception e) { }

    System.out.println(now.get() + ": All finished and moving on.");
}).start();
```

The code spawns two threads and passes the `CountDownLatch` to each of them. Both threads start at (approximately) the same time, but one of them sleeps much longer than the other. Once the sleep is over they count down the latch, and once it reaches zero both threads can make progress again.

If you execute this snippet, you'll get an output like this (it shows the time as well to make clear that indeed 5 seconds have passed):

```
2019-03-28T15:19:02.080658: Thread 2 starting work...
2019-03-28T15:19:02.080665: Thread 1 starting work...
2019-03-28T15:19:07.092761: Both finished and moving on.
2019-03-28T15:19:07.092827: Both finished and moving on.
```

This is pretty neat, but it has one big problem when we look at it from a reactive angle: it blocks the threads until they can make progress - and blocking is the one thing you want to avoid at all cost when going reactive.

So, how can we coordinate two or more execution pipelines to be waiting on a barrier without blocking the underlying threading infrastructure?

## A simple `Barrier`

One possibility is to create our own barrier around the [MonoProcessor](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/MonoProcessor.html) that ships with [Project Reactor](https://projectreactor.io/). The `Processor` in reactive streams terminology acts both as a `Publisher` and a `Subscriber` - so we can store state in it while automatically following the reactive streams contract to notify our subscribers if needed.

So, here is the `Barrier`:

```java
class Barrier {

    private final MonoProcessor<Void> inner = MonoProcessor.create();
    private final AtomicInteger ready = new AtomicInteger();
    private final int needed;

    public Barrier(int needed) {
        this.needed = needed;
    }

    Mono<Void> await() {
        return Mono.defer(() -> {
            if (ready.incrementAndGet() >= needed)  {
                inner.onComplete();
            }
            return inner;
        });
    }
}
```

It works similar to the `CountDownLatch` in one aspect: it gets created with the amount of parties that need to be ready and waiting before all of them can proceed. Internally, we track all the ready parties with a thread-safe `AtomicInteger`. When `await()` is called, we increment our ready counter and if we are at our needed threshold we complete the `MonoProcessor`.

If you are confused about the nested `Mono.defer()`: in the next code snippet you'll see how the `await()` method is used. If the code is not turned into a "cold" mono by deferring it, the code will be executed at runtime time and basically immediately proceed without waiting for the actual delay event.

The following code snippet is modeled after our very first example, but using two reactive sequences instead of spawning two separate threads.

```java
final Barrier barrier = new Barrier(2);

final Supplier<String> now = () -> LocalDateTime.now().format(DateTimeFormatter.ISO_DATE_TIME);

System.out.println(now.get() + ": Starting both sequences...");

Mono
    .delay(Duration.ofSeconds(5))
    .then(barrier.await())
    .subscribe(v -> {}, e -> {}, () -> System.out.println(now.get() + ": All finished and moving on."));


Mono
    .delay(Duration.ofSeconds(1))
    .then(barrier.await())
    .subscribe(v -> {}, e -> {}, () -> System.out.println(now.get() + ": All finished and moving on."));
```

Instead of sleeping, our `Mono` instances use the `delay` operator to simulate work. Once the delay is over and the event is emitted, they will subscribe on the barrier and increment the internal counter. Once both reach this point the `Mono` will complete and they can both make progress.

In the console you'll again see that both are only proceeding once the barrier is lifted:

```
2019-03-28T15:29:02.64789: Starting both sequences...
2019-03-28T15:29:08.158664: All finished and moving on.
2019-03-28T15:29:08.15904: All finished and moving on.
```

## Carrying the `onNext` signals

One limitation of the approach outlined above is that we are effectively turning our chain into a `Mono<Void>`, loosing any value we might have carried around. If you only need a signal this might be okay, but in practice we often need to do something with those values after the barrier. To fix this, we can utilize the `delayUntil` operator in reactor and modify our barrier slightly. The `delayUntil` operator takes a function that waits for a `Processor` to complete as a signal.

We only need to make minor adjustments and implement the function interface the operator expects:

```java
class Barrier implements Function<Object, Publisher<Void>> {

    private final MonoProcessor<Void> inner = MonoProcessor.create();
    private final AtomicInteger ready = new AtomicInteger();
    private final int needed;

    public Barrier(int needed) {
        this.needed = needed;
    }

    @Override
    public Publisher<Void> apply(Object ignored) {
        if (ready.incrementAndGet() >= needed)  {
            inner.onComplete();
        }
        return inner;
    }
}
```

We only had to change the return type and the name of the method to implement the interface.

The following snippet shows the operator in action - note how similar it is to our previous approaches.

```java
Mono
    .delay(Duration.ofSeconds(5))
    .delayUntil(barrier)
    .subscribe(
        v -> System.out.println("My value is: " + v),
        e -> {},
        () -> System.out.println(now.get() + ": All finished and moving on.")
    );


Mono
    .delay(Duration.ofSeconds(1))
    .delayUntil(barrier)
    .subscribe(
        v -> System.out.println("My value is: " + v),
        e -> {},
        () -> System.out.println(now.get() + ": All finished and moving on.")
    );
```

## Summary
So there you have it! With a couple lines of code you can implement a reactive `Barrier` which works similar to its blocking `CountDownLatch` counterpart.

If you are using similar or other approaches let me know in the comments!