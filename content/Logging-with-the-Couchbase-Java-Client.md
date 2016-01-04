+++
date = "2013-05-16"
draft = false
title = "Logging with the Couchbase Java Client"
description = "This blog post shows you everything that you need to know about logging with the Couchbase Java Client (and Spymemcached)."
tags = ["java","couchbase","logging"]
slug = "Logging-with-the-Couchbase-Java-Client"
+++

## Introduction
There is a huge variety in logging frameworks for Java, and its hard to please everyone. To understand how logging is currently handled in the SDK, we have to go back a few years. As you may know, the SDK depends on the [spymemcached](https://code.google.com/p/spymemcached/) library and therefore also inherits its logging mechanisms. Back in the days when [@dustin](https://twitter.com/dlsspy) wrote spy, there was no good abstraction for logging available (like SLF4J), so he wrote his own. Nowadays things have changed, but spy still inherits this legacy.

At the time of writing, the SDK supports logging to a simple default logger (logs to STDERR from INFO level up), [Log4J](http://logging.apache.org/log4j/1.2/) and the SunLogger (java.util.logging). In the upcoming 2.9.0 release of spymemcached, it will also support the SLF4J logging facade where you can plug in your own implementation. The next version of the SDK (most likely 1.1.7) will depend on spy 2.9, so you'll also get the benefits there.

Before we dig into the concepts, here are the supported Log Levels (defined by `net.spy.memcached.compat.log.Level`):

 - TRACE (with 2.9)
 - DEBUG
 - INFO
 - WARN
 - ERROR
 - FATAL

Keep in mind that different loggers implement different levels, so for some of them a mapping needs top happen. This will be noted during the descriptions of each implementation.

We'll now look at the different logging mechanisms available and how you can configure them. SLF4J will be covered towards the end.

## Switching Logging
If you don't change anything, the default logger will be used. This mechanism just prints log messages to STDERR (from INFO level upwards). Chances are that you want to integrate the SDK with the same logging library that you use as well. The LoggerFactory inside spy decides at construction which one to choose, based on a system property. So you can either change this programmatically or through a param to the `java` command.

If you want to use the Log4JLogger programmatically, do it this way (before initializing the `CouchbaseClient` object):

	Properties systemProperties = System.getProperties();
    systemProperties.put("net.spy.log.LoggerImpl", "net.spy.memcached.compat.log.Log4JLogger");
    System.setProperties(systemProperties);

Of course, you need to add the Log4J JAR to your CLASSPATH to make it work (as we'll see later). Alternatively, you can set it this way on the comman dline:

	java -Dnet.spy.log.LoggerImpl=net.spy.memcached.compat.log.Log4JLogger ...

Now that we know how to enable the different implementations, let's look at them in greater detail.

## The Simple Default Logger
If you don't change anything, the SDK will use the DefaultLogger (net.spy.memcached.compat.log.DefaultLogger). This logger has no dependencies and prints every log message that is INFO level or higher (INFO, WARN, ERROR and FATAL) to the systems STDERR. Since the STDERR is covered by most IDEs automatically, you'll also see them in the console output window.

Since its so simple, you can't customize this behavior. Every log message gets timestamped as well (the format is `yyyy-MM-dd HH:mm:ss.SSS`). Connecting to Couchbase commonly looks like this:

	2013-05-07 12:28:41.852 INFO com.couchbase.client.CouchbaseConnection:  Added {QA sa=/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	2013-05-07 12:28:41.862 INFO com.couchbase.client.CouchbaseConnection:  Connection state changed for sun.nio.ch.SelectionKeyImpl@3d9360e2
	2013-05-07 12:28:41.887 INFO com.couchbase.client.ViewConnection:  Added localhost to connect queue
	2013-05-07 12:28:41.888 INFO com.couchbase.client.CouchbaseClient:  viewmode property isn't defined. Setting viewmode to production mode
	2013-05-07 12:28:41.986 INFO com.couchbase.client.CouchbaseConnection:  Shut down Couchbase client
	2013-05-07 12:28:41.991 INFO com.couchbase.client.ViewConnection:  Node localhost has no ops in the queue
	2013-05-07 12:28:41.992 INFO com.couchbase.client.ViewNode:  I/O reactor terminated for localhost

So the format is always: `<timestamp> <level> <classname> <message>`. Remeber that DEBUG messages or so will not be logged, so you won't see them with the DefaultLogger.

## The SunLogger (java.util.logging)
The SunLogger also doesn't introduce additional dependencies, since it depends on the `java.util.logging` implementation. The `java.util.logging.Level` enum defines the following levels: ALL, CONFIG, FINEST, FINER, FINE, INFO, WARNING, SEVERE and OFF. Since this does not map well to our defined Levels, here is the mapping that happens:

 - TRACE to FINEST (with 2.9)
 - DEBUG to FINE
 - INFO to INFO
 - WARN to WARNING
 - ERROR to SEVERE
 - FATAL to SEVERE

Without any further changes, the SunLogger also prints from INFO level upwards like this:

	May 7, 2013 12:42:16 PM com.couchbase.client.CouchbaseProperties setPropertyFile
	INFO: Could not load properties file "cbclient.properties" because: File not found with system classloader.
	May 7, 2013 12:42:16 PM net.spy.memcached.MemcachedConnection createConnections
	INFO: Added {QA sa=/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	May 7, 2013 12:42:16 PM net.spy.memcached.MemcachedConnection handleIO
	INFO: Connection state changed for sun.nio.ch.SelectionKeyImpl@4ce2cb55
	May 7, 2013 12:42:16 PM com.couchbase.client.ViewConnection createConnections
	INFO: Added localhost to connect queue
	May 7, 2013 12:42:16 PM com.couchbase.client.CouchbaseClient <init>
	INFO: viewmode property isn't defined. Setting viewmode to production mode
	May 7, 2013 12:42:16 PM com.couchbase.client.CouchbaseConnection run
	INFO: Shut down Couchbase client
	May 7, 2013 12:42:16 PM com.couchbase.client.ViewConnection shutdown
	INFO: Node localhost has no ops in the queue
	May 7, 2013 12:42:16 PM com.couchbase.client.ViewNode$1 run
	INFO: I/O reactor terminated for localhost

If you want to change the log level to DEBUG and lower, you can do it like this:

	Logger.getLogger("com.couchbase.client").setLevel(Level.FINEST);

Now there is one more thing you need to do if you want to print all debug messages to the console. You set the logging level correctly, but the `ConsoleHandler` is not set to debug yet (so most likely you will pay the price for debug logging, but won't actually see anything in your IDE).

	for(Handler h : Logger.getLogger("com.couchbase.client").getParent().getHandlers()) {
        if(h instanceof ConsoleHandler) {
            h.setLevel(Level.FINEST);
        }
    }

So, here is a full example on how to use the `SunLogger` and get all Debug messages on the console.

    Properties systemProperties = System.getProperties();
    systemProperties.put("net.spy.log.LoggerImpl", "net.spy.memcached.compat.log.SunLogger");
    System.setProperties(systemProperties);

    Logger logger = Logger.getLogger("com.couchbase.client");
    logger.setLevel(Level.FINEST);
    for(Handler h : logger.getParent().getHandlers()) {
	    if(h instanceof ConsoleHandler){
	        h.setLevel(Level.FINEST);
	    }
    }

Then just go ahead and create your `CouchbaseClient` object, you will see detailed output like this (trimmed here):

	May 7, 2013 12:54:34 PM com.couchbase.client.vbucket.ReconfigurableObserver update
	FINEST: Received an update, notifying reconfigurables about a com.couchbase.client.vbucket.config.Bucketcom.couchbase.client.vbucket.config.Bucket@3d77949
	May 7, 2013 12:54:34 PM com.couchbase.client.vbucket.ReconfigurableObserver update
	FINEST: Received an update, notifying reconfigurables about a com.couchbase.client.vbucket.config.Bucketcom.couchbase.client.vbucket.config.Bucket@4e927aef
	May 7, 2013 12:54:34 PM com.couchbase.client.vbucket.ReconfigurableObserver update
	FINEST: It says it is default and it's talking to /pools/default/bucketsStreaming/default?bucket_uuid=adfff22b70e09fafaa26ca37b7e05e9d
	May 7, 2013 12:54:34 PM com.couchbase.client.vbucket.ReconfigurableObserver update
	FINEST: It says it is default and it's talking to /pools/default/bucketsStreaming/default?bucket_uuid=adfff22b70e09fafaa26ca37b7e05e9d

## Log4J
Most people will need more flexibility, and Log4J was (and still is) standard in lots of applications. The SDK provides support for Log4J as well. To make it work, you first need to set the instance correctly:

    Properties systemProperties = System.getProperties();
    systemProperties.put("net.spy.log.LoggerImpl", "net.spy.memcached.compat.log.Log4JLogger");
    System.setProperties(systemProperties);

Now if you run this, you'll get an error that some of the Log4J classes can not be found. This is not a surprise, because its not on the classpath. Let's fix this by adding it accordingly. If you use maven, add the `log4j.log4j` dependency (current version is 1.2.17). You can also just download the JAR and add it to the CLASSPATH as needed.

Now if we run it again, we get another error:

	log4j:WARN No appenders could be found for logger (com.couchbase.client.vbucket.ConfigurationProviderHTTP).
	log4j:WARN Please initialize the log4j system properly.
	log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.

One way to fix this is to get a correct `log4j.xml` configuration file into our CLASSPATH, but to make it work quickly Log4J provides a `BasicConfigurator`. Right after the system property configurations, add this:

	org.apache.log4j.BasicConfigurator.configure();

If you run it with the code change applied, you will see that we get nicely printed log messages. You can also see that they show up straight from the DEBUG level (and even contain information from which thread they got logged):

	69 [main] INFO com.couchbase.client.CouchbaseConnection  - Added {QA sa=/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	70 [main] DEBUG com.couchbase.client.vbucket.VBucketNodeLocator  - Updating nodesMap in VBucketNodeLocator.
	73 [main] DEBUG com.couchbase.client.vbucket.VBucketNodeLocator  - Adding node with hostname 127.0.0.1:11210.
	74 [main] DEBUG com.couchbase.client.vbucket.VBucketNodeLocator  - Node added is {QA sa=localhost/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=8}.
	74 [Memcached IO over {MemcachedConnection to localhost/127.0.0.1:11210}] DEBUG com.couchbase.client.CouchbaseConnection  - Done dealing with queue.
	74 [Memcached IO over {MemcachedConnection to localhost/127.0.0.1:11210}] DEBUG com.couchbase.client.CouchbaseConnection  - Selecting with delay of 0ms
	79 [Memcached IO over {MemcachedConnection to localhost/127.0.0.1:11210}] DEBUG com.couchbase.client.CouchbaseConnection  - Selected 1, selected 1 keys
	79 [Memcached IO over

You can control the logging levels through the usual Log4J mechanisms. I won't go into detail about them here, so please [check out](http://logging.apache.org/log4j/1.2/manual.html) their official documentation (for example on how to use the `PropertyConfigurator` instead).

Speaking of Log4J, [Steffen Larsen](https://twitter.com/zooldk) implemented a [Log4J appender](https://github.com/zooldk/log4j-couchbase) to store logs in Couchbase (instead of a file)!

## The new Facade: SLF4J
Not binding the application to a specific logging library is always a good idea. SLF4J is a facade for various pluggable logging frameworks behind it. So you can choose the logging implementation during runtime, be it [logback](http://logback.qos.ch/), Log4J or others. Since we already tried Log4J, let's make SLF4J work with Logback, one of the other very common log frameworks out there.

Note that SLF4J support will be available in the 1.9.0 release of spymemcached and therefore also in one of the next releases of the Couchbase Java SDK.

First, we need to configure it accordingly:

	Properties systemProperties = System.getProperties();
    systemProperties.put("net.spy.log.LoggerImpl", "net.spy.memcached.compat.log.SLF4JLogger");
    System.setProperties(systemProperties);

Now, we need to include two JARs into our classpath. The first one is the SLF4J facade API and the other one is our logging framework of choice. The facade API package is called `slf4j-api` (this package always needs to be in place) and since we want to use logback we need to include the `logback-classic` JAR. Note that this is not specific to the SDK, you can find this information [here](http://logback.qos.ch/manual/introduction.html). If you use maven, you can use this snippet:

    <dependency>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
      <version>1.7.5</version>
    </dependency>
    <dependency>
      <groupId>ch.qos.logback</groupId>
      <artifactId>logback-classic</artifactId>
      <version>1.0.12</version>
    </dependency>

SLF4J will automatically pick up our logback implementation, so the logs will look like this:

	13:25:43.692 [main] INFO  c.c.client.CouchbaseConnection - Added {QA sa=/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	13:25:43.694 [main] DEBUG c.c.c.vbucket.VBucketNodeLocator - Updating nodesMap in VBucketNodeLocator.
	13:25:43.697 [main] DEBUG c.c.c.vbucket.VBucketNodeLocator - Adding node with hostname 127.0.0.1:11210.
	13:25:43.697 [main] DEBUG c.c.c.vbucket.VBucketNodeLocator - Node added is {QA sa=localhost/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=8}.
	13:25:43.698 [Memcached IO over {MemcachedConnection to localhost/127.0.0.1:11210}] DEBUG c.c.client.CouchbaseConnection - Done dealing with queue.
	13:25:43.699 [Memcached IO over {MemcachedConnection to localhost/127.0.0.1:11210}] DEBUG c.c.client.CouchbaseConnection - Selecting with delay of 0ms
	13:25:43.702 [Memcached IO over {MemcachedConnection to localhost/127.0.0.1:11210}] DEBUG c.c.client.CouchbaseConnection - Selected 1, selected 1 keys
	13:25:43.703 [Memcached IO over {MemcachedConnection to localhost/127.0.0.1:11210}] DEBUG c.c.client.CouchbaseConnection - Handling IO for:  sun.nio.ch.SelectionKeyImpl@48ff2413 (r=false, w=false, c=true, op={QA sa=localhost/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=8})
	13:25:43.703 [Memcached IO over {MemcachedConnection to localhost/127.0.0.1:11210}] INFO  c.c.client.CouchbaseConnection - Connection state changed for sun.nio.ch.SelectionKeyImpl@48ff2413
	13:25:43.713

As you can see, they also include DEBUG level logging here. If you don't include the logging implementation during runtime, SLF4J will complain at startup:

	SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
	SLF4J: Defaulting to no-operation (NOP) logger implementation
	SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details

If you want to learn how to configure logback, [look here](http://logback.qos.ch/manual/configuration.html).

## Summary
Once you know the abstraction in spymemcached and how it works, switching logging implementations is easy and straightforward. If you work with one of the Couchbase people to report errors, please try to include output with DEBUG turned on, because this includes lots of useful information that can be used to determine the failure sources.

With the SLF4J facade added in the next spy release (2.9), you will be able to plug every large logging framework out there into the SDK. Let us know if you see a use case not covered with these mechanisms or if you have other comments on this.