+++
date = "2013-10-11"
draft = false
title = "What's new in the Couchbase Java SDK 1.2"
description = "Read on to see whats new in the latest Couchbase Java SDK!"
tags = ["java","couchbase"]
slug = "What-s-new-in-the-Couchbase-Java-SDK-1-2"
+++

For all users of our Java SDK, we prepared some nice additions for you. This post covers them in detail and shows how you can get more productive.

Note that this blog post assumes you are running the 1.2.1 release, because there have been some slight changes between 1.2.0 and 1.2.1 that affect for example the listener support and metrics collection.

## Maven Central Distribution
From the 1.2.0 release forward, the Java SDK is distributed directly from Maven Central. This means that you don't need to include the Couchbase repository anymore. The following maven code is enough to get started (note that the groupId has changed):

    <dependencies>
        <dependency>
            <groupId>com.couchbase.client</groupId>
            <artifactId>couchbase-client</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>

This will automatically load the latest spymemcached dependency in as well (for 1.2.0 it's 2.10.0). Before we dig into what has changed, [here](http://docs.couchbase.com/couchbase-sdk-java-1.2/#release-notes-for-couchbase-client-library-java-120-ga-13-september-2013) are the release notes for a quick reference.

## Listener Support
Until now, there were two ways to get the result of an asynchronous request. Either by blocking the current thread like so:

    // do an async operation (returns immediately)
    OperationFuture<Boolean> setFuture = client.set("key", "value");
    
    // block the current thread
    Boolean result = setFuture.get();

Or to loop on the non-blocking future methods. This is especially helpful if you are dealing with a list of futures.

    List<OperationFuture<Boolean>> futures = new ArrayList<OperationFuture<Boolean>>();
    for (int i = 0; i < 100; i++) {
      futures.add(client.set("key-" + i, "value"));
    }

    while (!futures.isEmpty()) {
      Iterator<OperationFuture<Boolean>> iter = futures.iterator();
      while (iter.hasNext()) {
        OperationFuture<Boolean> future = iter.next();
        if (future.isDone()) {
          iter.remove();
        }
      }
    }

Now since 1.2.0, there is a new way to deal with responses - adding listeners. The idea is to supply a callback to the future which will be executed once the operation is done. A simple example is shown here:

    OperationFuture<Boolean> setFuture = client.set("key", "value");
    setFuture.addListener(new OperationCompletionListener() {
      @Override
      public void onComplete(OperationFuture<?> future) throws Exception {
        System.out.println(future.get());
      }
    });

Note that the `.get()` method on the future will not block anymore because the result is already computed. Whatever you put in the callback method will be executed asynchronously on the thread pool. To see how flexible that approach is, let's rewrite the example from above waiting until the 100 futures are done.


    final CountDownLatch latch = new CountDownLatch(100);
    for (int i = 0; i < 100; i++) {
      OperationFuture<Boolean> future = client.set("key-" + i, "value");
      future.addListener(new OperationCompletionListener() {
        @Override
        public void onComplete(OperationFuture<?> future) throws Exception {
          latch.countDown();
        }
      });
    }
    latch.await();

Here we are using a [CountDownLatch](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/CountDownLatch.html) which waits on the current thread as long as it has been counted down a hundred times. Exactly what we need in our situation, but the code is much easier to read. More importantly, its much more flexible because other things like firing off a new request, querying a web service or calculating a result can be done.

It is also possible to override the default `ExecutorService` implementation with a custom one. This may be needed if the default behavior (Basically a upper-bounded cachedThreadPool) does not suite your needs. Also, you should use this approach if you create a bunch of `CouchbaseClient` instances so you can share the same service across all of them.

    // Create the Builder
    CouchbaseConnectionFactoryBuilder builder = new CouchbaseConnectionFactoryBuilder();

    // Create a thread pool of 5 fixed threads
    ExecutorService service = Executors.newFixedThreadPool(5);
    // Set it in the builder
    builder.setListenerExecutorService(service);

    // Create the instance
    CouchbaseClient client = new CouchbaseClient(builder.buildCouchbaseConnection(...));

## Enhanced Profiling Capabilities
Getting insight into a running application is always difficult, so we set out to make it easier for you. We incorporated a library called [metrics](http://metrics.codahale.com/) that profiles, depending on the configuration level chosen.

Before you can use it, you need to add this optional dependency:

    <dependency>
        <groupId>com.codahale.metrics</groupId>
        <artifactId>metrics-core</artifactId>
        <version>3.0.1</version>
    </dependency>

On the builder, there is a method that allows you to activate the the profiler:

	CouchbaseConnectionFactoryBuilder builder = new CouchbaseConnectionFactoryBuilder();

	// enable metric collection
	builder.setEnableMetrics(MetricType.PERFORMANCE);

If you look at the `MetricType` enumeration you can see that there are three types of values you can choose from: OFF (which keeps metric collection off), PERFORMANCE (which only collects performance-relevant metrics) and DEBUG (which collects all kinds of metrics, including the performance ones). While the metrics library is quite efficient, keep in mind that metric collection takes some resources away from your application.

By default, the metric information will be printed out on the console every 30 seconds. You can run the following test code from your IDE and see how it looks:

    CouchbaseConnectionFactoryBuilder builder = new CouchbaseConnectionFactoryBuilder();
    builder.setEnableMetrics(MetricType.PERFORMANCE);

    CouchbaseConnectionFactory cf =
      builder.buildCouchbaseConnection(Arrays.asList(new URI("http://127.0.0.1:8091/pools")), "default", "");

    CouchbaseClient client = new CouchbaseClient(cf);

    while(true) {
      client.set("foo", "bar");
      Thread.sleep(100);
    }

Now wait 30 seconds and you'll see output like this in the console:

```
  10/8/13 12:04:14 PM ============================================================

  -- Histograms ------------------------------------------------------------------
  [MEM] Average Bytes read from OS per read
               count = 893
                 min = 24
                 max = 24
                mean = 24.00
              stddev = 0.00
              median = 24.00
                75% <= 24.00
                95% <= 24.00
                98% <= 24.00
                99% <= 24.00
              99.9% <= 24.00
  [MEM] Average Bytes written to OS per write
               count = 893
                 min = 38
                 max = 38
                mean = 38.00
              stddev = 0.00
              median = 38.00
                75% <= 38.00
                95% <= 38.00
                98% <= 38.00
                99% <= 38.00
              99.9% <= 38.00
  [MEM] Average Time on wire for operations (Âµs)
               count = 893
                 min = 179
                 max = 1730
                mean = 263.80
              stddev = 75.43
              median = 251.00
                75% <= 280.00
                95% <= 351.90
                98% <= 425.36
                99% <= 559.70
              99.9% <= 1730.00

  -- Meters ----------------------------------------------------------------------
  [MEM] Request Rate: All
               count = 893
           mean rate = 9.92 events/second
       1-minute rate = 9.85 events/second
       5-minute rate = 9.68 events/second
      15-minute rate = 9.63 events/second
  [MEM] Response Rate: All (Failure + Success + Retry)
               count = 893
           mean rate = 9.92 events/second
       1-minute rate = 9.85 events/second
       5-minute rate = 9.68 events/second
      15-minute rate = 9.63 events/second
```

I won't go into detail of all these metrics in this blog post, please refer to the documentation for a more complete picture. One more thing I want to show you is that the metrics library is also able to expose these metrics through JMX. All you need to do is set a system property that changes the output mode: `net.spy.metrics.reporter.type=jmx`. Other possible settings are `csv` and slf4j`. If you choose a logger that prints out information at a given interval you can change it by setting `net.spy.metrics.reporter.interval` to anything else than 30.

So if you put the line `System.setProperty("net.spy.metrics.reporter.type", "jmx");` before the code shown above, you can open (for example) jConsole and switch to the MBeans tab of the application. You'll see a `metrics` subsection exposed that contains the same metrics as they would show up in the logs.

## CAS with Timeout
Before 1.2.0, it was not possible in one command to do a `cas` update and set a new timeout at the same time. You had to do a second `touch` operation which was not efficient nor atomic. Now, the API exposes a new `cas()` method that allows you to pass in the timeout at the same time. It is easy to use:

	client.cas("key", cas, nexExpiration, value);

The asynchronous variations have been exposed since 1.2.1 as well.

## Initializing through Properties
One thing that comes in handy if your cluster ip addresses change often is that you can now initialize a `CouchbaseClient` object based on system properties. Here is an example:

    System.setProperty("cbclient.nodes", "http://127.0.0.1:8091/pools");
    System.setProperty("cbclient.bucket", "default");
    System.setProperty("cbclient.password", "");

    CouchbaseConnectionFactoryBuilder builder = new CouchbaseConnectionFactoryBuilder();
    CouchbaseConnectionFactory cf = builder.buildCouchbaseConnection();
    CouchbaseClient client = new CouchbaseClient(cf);

Of course you can set these properties in your application container or during startup, so it's very flexible and not tied into your code directly. Note that if you forget to set one of these properties, the code will warn you like this:

```
Exception in thread "main" java.lang.IllegalArgumentException: System property cbclient.nodes not set or empty
    at com.couchbase.client.CouchbaseConnectionFactory.<init>(CouchbaseConnectionFactory.java:160)
    at com.couchbase.client.CouchbaseConnectionFactoryBuilder$2.<init>(CouchbaseConnectionFactoryBuilder.java:318)
    at com.couchbase.client.CouchbaseConnectionFactoryBuilder.buildCouchbaseConnection(CouchbaseConnectionFactoryBuilder.java:318)
    at Main.main(Main.java:33)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:601)
    at com.intellij.rt.execution.application.AppMain.main(AppMain.java:120)
```

## Other Changes
In addition to the enhancements shown above, the release includes - as always - numerous smaller bugfixes. The default poll interval for `ReplicateTo` and `PersistTo` has been lowered to `10ms` to account for performance changes that went into the Couchbase Sever 2.2 release. Also, the client now uses the `CRAM-MD5` authentication mechanism automatically if the server supports it (since 2.2 as well).

These awesome new features should be enough reason to upgrade right now! If anything pops up that doesn't work as expected, please ask customer support or open a ticket [here](http://www.couchbase.com/issues/browse/JCBC).
