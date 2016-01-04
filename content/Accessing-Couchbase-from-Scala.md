+++
date = "2012-04-26"
draft = false
title = "Accessing Couchbase from Scala"
description = "This post shows you how to access your Couchbase server from Scala."
tags = ["couchbase","scala"]
slug = "Accessing-Couchbase-from-Scala"
+++

In my recent adventure on exploring both [Couchbase](http://www.couchbase.com/) and [Scala](http://www.scala-lang.org/), I noticed that there are not 
many tutorials out there using both at the same time. We'll use the [couchbase-java driver](http://www.couchbase.com/develop/java/next) in its latest 
version (1.1), because it provides support for views and Couchbase 2.0. Note that I won't cover installing Couchbase here, since there is plenty of 
material [out there](http://www.couchbase.com/docs/couchbase-manual-2.0/couchbase-getting-started-install.html). Also, I assume you have scala and 
[sbt](https://github.com/harrah/xsbt) already installed. The [typesafe stack](http://typesafe.com/stack) provides a convenient shortcut to get you 
up and running in minutes, so check it out if you haven't already. For the purpos of simplicity, I assume that your Couchbase instance runs on 
`localhost`. If it runs elswhere, just modify the URI string later accordingly.

Let's create the standard directory structure for our project. We'll use sbt to manage our dependencies, so create a `build.sbt` file too.

	/couchbase-scala
		/src
			/main
				/scala
					Main.scala
		build.sbt

Let's write the `build.sbt` first:

	name := "couchbase-scala"
	
	version := "1.0"
	
	scalaVersion := "2.9.2"
	
	resolvers += "Couchbase Maven Repository" at "http://files.couchbase.com/maven2"
	
	libraryDependencies += "couchbase" % "couchbase-client" % "1.1-dp"

We need to add the [Couchbase Maven Repository](http://files.couchbase.com/maven2) to our resolvers list, since it contains the Couchbase package and all 
its dependencies. The last line adds the actual dependency. If you want the (older) stable release, use "1.0.2" instead of "1.1-dp". Tip: you can browse the 
repository directly in your browser and see what versions are available.

Let's add a simple `Main` object in `Main.scala`:

	object Main extends App {
		println("Hello World")
	}

Now, start sbt and run the application for the first time:

	couchbase-scala$ sbt
	> run
	** ((Omitting Fetch and Compile Output)) **
	[info] Running Main
	Hello World
	[success] Total time: 4 s, completed 24.04.2012 11:01:33

I've omited the part where sbt resolves our dependencies and compiles the libraries, but the output should look like the one above. Now that we have all our 
dependencies in place, we can start talking to the Couchbase server. Here's a short example with synchronous set and get operations (note that no exception 
handling takes place here):

	import collection.JavaConversions._ 
	import collection.mutable.ArrayBuffer

	import java.net.URI;
	import java.net.ConnectException;

	import com.couchbase.client.CouchbaseClient;

	object Main extends App {
		val uris = ArrayBuffer(URI.create("http://127.0.0.1:8091/pools"))
		val client = new CouchbaseClient(uris, "default", "")
		
		client.set("hello", 0, "world")
		println("Receiving -> " + client.get("hello"))
		
		client.shutdown()
		System.exit(0)
	}

Save the file, type `run` again in sbt and you should get the following output (you only need to leave the REPL if you've changed the `build.sbt` file):

	[info] Compiling 1 Scala source to <Path\to\Sourcecode>\couchbase-scala\target\scala-2.9.2\classes...
	[info] Running Main
	2012-04-24 11:05:11.850 INFO com.couchbase.client.CouchbaseConnection:  Added {QA sa=/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	2012-04-24 11:05:11.856 INFO com.couchbase.client.CouchbaseClient:  viewmode property isn't defined. Setting viewmode to production mode
	2012-04-24 11:05:11.856 INFO com.couchbase.client.CouchbaseConnection:  Connection state changed for sun.nio.ch.SelectionKeyImpl@10319d7
	2012-04-24 11:05:11.926 INFO com.couchbase.client.ViewConnection:  Added daschl-notebook/127.0.0.1:8092 to connect queue
	Receiving -> world
	2012-04-24 11:05:12.360 INFO com.couchbase.client.CouchbaseConnection:  Shut down Couchbase client
	2012-04-24 11:05:12.368 INFO com.couchbase.client.ViewConnection:  Shut down Couchbase client
	2012-04-24 11:05:12.374 INFO com.couchbase.client.ViewNode:  Couchbase I/O reactor terminated
	[success] Total time: 6 s, completed 24.04.2012 11:05:12

You see all the Couchbase log messages here, because the default logger logs directly to STDOUT. Right in the middle you can see the output of our synchronous [get](http://www.couchbase.com/docs/couchbase-sdk-java-1.1/couchbase-sdk-java-retrieve-get.html#table-couchbase-sdk_java_get) operation. You should also see the 
created document in your Couchbase [default bucket](http://localhost:8091/index.html#sec=documents&bucketName=default).

Let's go through the main code line by line and see what's going on.

	val uris = ArrayBuffer(URI.create("http://127.0.0.1:8091/pools"))	
	val client = new CouchbaseClient(uris, "default", "")

The `CouchbaseClient` accepts a list of URIs to communicate with as the first argument (you need to import `collection.JavaConversions._ `, so that `ÀrrayBuffer` gets 
converted to a `java.util.List`). The second argument is the bucket name, the third one is an optional bucket password (which is always empty for the `default` one).

	client.set("hello", 0, "world")
	println("Receiving -> " + client.get("hello"))

We can now throw all commands at the client, be it synchronous or asynchronous ones. You can find all supported methods [here](http://www.couchbase.com/docs/couchbase-sdk-java-1.1/api-reference-summary.html).

	client.shutdown()

Finally, we close the connection to the server. You can pass optional timeouts here, so that the client waits for the given amount of time before it finally closes the connection. 
This allows asynchronous commands to finish first.

This post was a very basic introduction on using Couchbase from Scala. The next post will translate the [tutorial](http://www.couchbase.com/docs/couchbase-sdk-java-1.1/tutorial.html) to Scala. As always, I'm open to suggestions and 
improvements!
