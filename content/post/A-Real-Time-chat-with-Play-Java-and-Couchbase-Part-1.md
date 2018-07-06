+++
date = "2012-07-17"
draft = false
title = "A Real-Time chat with Play, Java and Couchbase - Part 1"
description = "This first post of a longer series introduces you to our project and helps you get everything up and running."
tags = ["java","couchbase","play"]
slug = "A-Real-Time-chat-with-Play-Java-and-Couchbase-Part-1"
+++

## Introduction
I've been mostly blogging about PHP and [Lithium](http://lithify.me), but recently I've also been looking into a very promising framework on the JVM - the [play! framework](http://www.playframework.org/). The current version (2.0) brings lots of enhancements and features and is (at least to me) the first framework that really boosts developer productivity on the JVM (and that I would work with in my free time).

In this project, we'll develop a chat application (called `couchplay`) that allows people to login with a username and then talk to others in real-time. We'll make use of the built-in [Comet](http://en.wikipedia.org/wiki/Comet_%28programming%29) functionality and also work with various other aspects of the whole stack. Our application will be powered by [Couchbase](http://www.couchbase.com/), a very fast and flexible NoSQL database that is exactly suited for an application like this.

Note that this tutorial requires a reasonable amount of Java knowledge. Play! itself is mostly written in [Scala](http://www.scala-lang.org/) (and uses it for its templates), but I think it should be easy to grasp - at least for the part we'll cover here.

I've also added the code to a [repository](https://github.com/daschl/couchplay/) on GitHub for your convenience. You can find a tag for every post I'll publish so you can just clone the repository and checkout the appropriate tag to follow along.

## About the stack
Our language of choice will be Java. I first thought about writing it in Scala, but then I realized that with Java I could attract a larger group of readers. Also, I did most of my past explorations in Scala, so this is a great opportunity for me to use the framework from the Java-side as well. The good thing is that the framework allows you to use both languages in parallel, so you can start out with one and then evolve if you need to.

As I said, the play! framework is mostly written in Scala. As a result, the default template engine requires a bit of Scala knowledge, but for our examples here the concepts should not be too hard to learn.  If you come from a mostly servlet-oriented background I predict you a very refreshing look on how web applications can be built also (it is built on top of [netty](http://netty.io/) and [akka](http://akka.io/)). Once you get the hang of it, you think twice about going back. If you don't believe me now, just follow along.

The framework provides support for relational databases out of the box, but to make things more interesting we're going to use a different database. Couchbase is a high-performant and flexible "NoSQL" database that is mostly used in real-time analytics, session stores and in general where you need sub-millisecond response times. At the end of the blog series you should have a good idea how you can use it for your own projects. Note that I also wrote a blog post on getting started with Couchbase on either [PHP](http://nitschinger.at/Getting-Started-with-Couchbase-and-PHP) and [Scala](http://nitschinger.at/Accessing-Couchbase-from-Scala), so you might want to check them out. For this tutorial, we're going to use the [Couchbase Java SDK](http://www.couchbase.com/develop/java/next), which has everything built-in that we need.

From here on I assume that you have Java 1.6 or later installed and included it in your `PATH`. If you haven't, there should be plenty of material available on the internet.

## Installing Couchbase
Before we start digging into the framework, we need to stop for a second and install our backend database.

Depending on your operating system, you have different choices available. Couchbase provides installers for Windows and Mac OS X, for Linux-based systems I'd recommend the package-manager installations. You can find all packages [here](http://www.couchbase.com/download).

We are going to use Couchbase 2.0, which is currently in developer preview (at the time of writing). Just download it, it is very stable and a beta release will be released soon. We are going to use this version because it includes a mechanism called "views" that allows us to query our data in a very flexible manner (if you're familiar with the [Map/Reduce-Pattern](http://en.wikipedia.org/wiki/MapReduce), this is the method we're going to use here). We'll cover all of those aspects in a later post.

After the installation has finished, open your browser, head over to `http://localhost:8091` and click through the setup wizard. You can go with the default settings, just make sure to give it a reasonable amount of RAM. If you run into trouble, the official [docs](http://www.couchbase.com/docs/couchbase-manual-2.0/couchbase-getting-started-install.html) are there to help you out.

## Setting up our play! project
Now that we have our database installed, it's time to set up the actual play! project. Go to [their official website](http://www.playframework.org/) and download the current stable release (2.0.2 at the time of writing).

After the download has finished, unpack the zipfile. Copy it over to a place where it doesn't get accidentally deleted and add it to your `PATH`. Adding it to the `PATH` here is pure convenience as I don't like to remember the full path everytime I need to run it.

Now we can use the `play new` command to setup a new project. I'm using Ubuntu Linux here, but the command also works the same on Windows (I haven't tried it on a Mac but it shouldn't be much different).

	michael@daschl:~$ play new couchplay
		_            _
	_ __ | | __ _ _  _| |
	| '_ \| |/ _' | || |_|
	|  __/|_|\____|\__ (_)
	|_|            |__/

	play! 2.0.2, http://www.playframework.org

	The new application will be created in /home/michael/couchplay

	What is the application name?
	> couchplay

	Which template do you want to use for this new application?

	1 - Create a simple Scala application
	2 - Create a simple Java application
	3 - Create an empty project

	> 2

	OK, application couchplay is created.

	Have fun!

The application generator asks you for an application name and if you want to either generate a Scala or Java application. You don't have to choose the language for your final application here, it only wants to know in what language it should generats the sample controller for example.

To make sure it's up and running, change into the directory and run the `play` command again. This should bring up the play console. For now, just type `run` and point your browser to `http://localhost:9000`. You should see the default welcome page of the framework.

	michael@daschl:~/couchplay$ play
	Getting org.scala-sbt sbt_2.9.1 0.11.3 ...
	:: retrieving :: org.scala-sbt#boot-app
		confs: [default]
		37 artifacts copied, 0 already retrieved (7245kB/44ms)
	[info] Loading project definition from /home/michael/couchplay/project
	[info] Set current project to couchplay (in build file:/home/michael/couchplay/)
		_            _
	_ __ | | __ _ _  _| |
	| '_ \| |/ _' | || |_|
	|  __/|_|\____|\__ (_)
	|_|            |__/

	play! 2.0.2, http://www.playframework.org

	> Type "help play" or "license" for more information.
	> Type "exit" or use Ctrl+D to leave this console.

	[couchplay] $ run

	[info] Updating {file:/home/michael/couchplay/}couchplay...
	[info] Resolving org.hibernate.javax.persistence#hibernate-jpa-2.0-api;1.0.1.Fin                                                                                [info] Done updating.
	--- (Running the application from SBT, auto-reloading is enabled) ---

	[info] play - Listening for HTTP on port 9000...

	(Server started, use Ctrl+D to stop and go back to the console...)

As you can see, the initial dependencies are resolved and loaded automatically. If you already have something running on port `9000` (like [php-fpm](http://php-fpm.org/)), then you can use the `run [PORTNUM]` command to specify a different port. There are lots of other possibilities here and we'll cover some of them as we progress further. If you're curious, you can type `help` to see the available commands.

You may experience a large response time (a few seconds) when you open up the page for the first time. This is because in development mode, play! compiles everything for you on the fly. If you don't change anything, the next reload should be pretty fast. Since we only modify short bits of code between every refresh, the compiling times should not be an issue. In production, everything is precompiled anyway. If you look on your console, you can see that the files have been compiled automatically for you:

	[info] Compiling 4 Scala sources and 2 Java sources to /home/michael/couchplay/target/scala-2.9.1/classes...
	[info] play - Application started (Dev)

One last tip here: if you use `~run` instead of `run`, the framework will detect code changes immediately (when you save your file) and compile it on the fly. If you use `run`, it will start compiling when you reload the page. As a result, when using `~run` the compile times should be even less noticable since you also need some time to switch back and forth between your IDE and the browser. Also keep in mind that all this is handled for you and you don't need to setup tomcat in your IDE and make sure everything is pushed correctly. If you are like me and have spent hours and hours of configuring eclipse and tomcat, this is pretty awesome news. You can of course still use Eclipse (or any other IDE), but you can concentrate on writing code, the rest will be done for you in the background.

## Connecting to Couchbase
Currently, our application doesn't know about our backend database. Since play! doesn't have built-in support for Couchbase, we're going to use the Couchbase Java SDK to fill the gap.

Couchbase thankfully provides a Maven repository, so we just need to define and load the dependency. Head over to the `project/Build.scala` file and add the following:

	val appDependencies = Seq(
		"couchbase" % "couchbase-client" % "1.1-dp"
	)

	val main = PlayProject(appName, appVersion, appDependencies, mainLang = SCALA).settings(
		resolvers += "Couchbase Maven Repository" at "http://files.couchbase.com/maven2"
	)

Back in the play console, hit `ctrl-d` to stop the server and type `reload` and `update` to fetch the new dependencies. If everything finished without errors, type "~run" again.

	[couchplay] $ reload
	[info] Loading project definition from /home/michael/couchplay/project
	[info] Set current project to couchplay (in build file:/home/michael/couchplay/)
	[couchplay] $ update
	[info] Updating {file:/home/michael/couchplay/}couchplay...
	[info] Resolving org.hibernate.javax.persistence#hibernate-jpa-2.0-api;1.0.1.Fin                                                                                [info] downloading http://files.couchbase.com/maven2/couchbase/couchbase-client/1.1-dp/couchbase-client-1.1-dp.jar ...
	[info] 	[SUCCESSFUL ] couchbase#couchbase-client;1.1-dp!couchbase-client.jar (1821ms)
	[info] downloading http://repo.typesafe.com/typesafe/releases/org/jboss/netty/netty/3.2.0.Final/netty-3.2.0.Final.jar ...
	[info] 	[SUCCESSFUL ] org.jboss.netty#netty;3.2.0.Final!netty.jar(bundle) (3661ms)
	[info] downloading http://repo.typesafe.com/typesafe/releases/org/codehaus/jettison/jettison/1.1/jettison-1.1.jar ...
	[info] 	[SUCCESSFUL ] org.codehaus.jettison#jettison;1.1!jettison.jar(bundle) (828ms)
	[info] downloading http://repo.typesafe.com/typesafe/releases/commons-codec/commons-codec/1.5/commons-codec-1.5.jar ...
	[info] 	[SUCCESSFUL ] commons-codec#commons-codec;1.5!commons-codec.jar (835ms)
	[info] downloading http://repo.typesafe.com/typesafe/releases/spy/spymemcached/2.8.1/spymemcached-2.8.1.jar ...
	[info] 	[SUCCESSFUL ] spy#spymemcached;2.8.1!spymemcached.jar (1446ms)
	[info] downloading http://repo.typesafe.com/typesafe/releases/org/apache/httpcomponents/httpcore/4.1.1/httpcore-4.1.1.jar ...
	[info] 	[SUCCESSFUL ] org.apache.httpcomponents#httpcore;4.1.1!httpcore.jar (1059ms)
	[info] downloading http://repo.typesafe.com/typesafe/releases/org/apache/httpcomponents/httpcore-nio/4.1.1/httpcore-nio-4.1.1.jar ...
	[info] 	[SUCCESSFUL ] org.apache.httpcomponents#httpcore-nio;4.1.1!httpcore-nio.jar (1062ms)
	[info] downloading http://repo.typesafe.com/typesafe/releases/stax/stax-api/1.0.1/stax-api-1.0.1.jar ...
	[info] 	[SUCCESSFUL ] stax#stax-api;1.0.1!stax-api.jar (1556ms)
	[info] Done updating.
	[success] Total time: 39 s, completed Jul 7, 2012 9:25:52 AM
	[couchplay] $ ~run

	--- (Running the application from SBT, auto-reloading is enabled) ---

	[info] play - Listening for HTTP on port 9000...

	(Server started, use Ctrl+D to stop and go back to the console...)

I'm quite sure you wonder about the strange syntax we've been using to define our dependency. This is actually Scala code, since play! uses the awesome [sbt](http://www.scala-sbt.org/) to manage its dependencies (the play console also builds upon it). There is a good introduction [video](http://www.youtube.com/watch?v=V2rl62CZPVc) available if you want to learn more.

The dependency is now in place, but we also need to connect to the database when the application starts and disconnect from it when it stops. If we'd use the databases supported by the framework, this would be handled for us (but that's half the fun, right?). Since this is not the case, we need to "hook" into some kind of `start` and `stop` events.

Play! provides us with something called the [application Global object](http://www.playframework.org/documentation/2.0.2/JavaGlobal) that has methods we can override. Create a new file called `Global.java` inside the `app` directory and insert the following.

	import play.*;

	import datasources.Couchbase;

	public class Global extends GlobalSettings {

		public void onStart(Application app) {
			Logger.info("Application started");
			Couchbase.connect();
		}

		public void  onStop(Application app) {
			Logger.info("Application stopped");
			Couchbase.disconnect();
		}

	}

We override the `onStart` and `onStop` methods to place our custom connect logic in there (and do some logging as well). We make use of a `Couchabase` connection class to abstract our connection logic, so we'd better write that one too. Create a new file called `Couchbase.java` inside the `app/datasources` directory (which you also need to create).

	package datasources;

	import java.net.URI;
	import java.util.LinkedList;
	import java.util.List;
	import java.io.IOException;
	import java.util.concurrent.TimeUnit;

	import play.*;
	import com.couchbase.client.CouchbaseClient;

	/**
	* The `Couchbase` class acts a simple connection manager for the `CouchbaseClient`
	* and makes sure that only one connection is alive throughout the application.
	*
	* You may want to extend and harden this implementation in a production environment.
	*/
	public final class Couchbase {

		/**
		* Holds the actual `CouchbaseClient`.
		*/
		private static CouchbaseClient client = null;

		/**
		* Connects to Couchbase based on the configuration settings.
		*
		* If the database is not reachable, an error message is written and the
		* application exits.
		*/
		public static boolean connect() {
			String hostname = Play.application().configuration().getString("couchbase.hostname");
			String port = Play.application().configuration().getString("couchbase.port");
			String bucket = Play.application().configuration().getString("couchbase.bucket");
			String password = Play.application().configuration().getString("couchbase.password");

			List<URI> uris = new LinkedList<URI>();
			uris.add(URI.create("http://"+hostname+":"+port+"/pools"));


			try {
				client = new CouchbaseClient(uris, bucket, password);
			} catch(IOException e) {
				Logger.error("Error connection to Couchbase: " + e.getMessage());
				System.exit(0);
			}

			return true;
		}

		/**
		* Disconnect from Couchbase.
		*/
		public static boolean disconnect() {
			if(client == null) {
				return false;
			}

			return client.shutdown(3, TimeUnit.SECONDS);
		}

		/**
		* Returns the actual `CouchbaseClient` connection object.
		*
		* If no connection is established yet, it tries to connect. Note that
		* this is just in place for pure convenience, make sure to connect explicitly.
		*/
		public static CouchbaseClient getConnection() {
			if(client == null) {
				connect();
			}

			return client;
		}

	}

We basically define a `connect` and a `disconnect` method, which get called from our `Gobal.java` class. Inside the `connect` method we read configuration settings (more on that shortly) and then try to open the Couchbase connection. The `disconnect` method waits 3 seconds to finish any queued operations and then disconnects. Also, we provide a `getConnection()` method that returns the actual instance of the client object (which we'll use later in our models).
	
It is always a good idea to just open one connection for the whole lifecycle instead of opening a new connection every time a request comes in. This removes unneeded handshake-latency and should therefore improve the overall performance of the application.

You may have noticed the various `Play.application().configuration().getString();` calls. With these method calls you can read the values from your `conf/application.conf` configuration file. By defining your connection settings there instead of the actual implementation, you can easily swap the settings if you deploy your application into production. More on the configuration file can be found [here](http://www.playframework.org/documentation/2.0.2/Configuration). To define the settings we need, open your `application.conf` file and add the following to the end (or somewhere in between):

	# Couchbase configuration
	# ~~~~~
	couchbase.hostname="localhost"
	couchbase.port=8091
	couchbase.bucket="default"
	couchbase.password=""


Now we have everything in place to configure the settings and connect to the database. If you start the server again and load the page for the first time, you can see the debug output of the Couchbase SDK:

	[info] application - Application started
	2012-07-17 19:12:43.788 INFO com.couchbase.client.CouchbaseConnection:  Added {QA sa=/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	2012-07-17 19:12:43.789 INFO com.couchbase.client.CouchbaseClient:  viewmode property isn't defined. Setting viewmode to production mode
	2012-07-17 19:12:43.790 INFO com.couchbase.client.ViewConnection:  Added localhost/127.0.0.1:8092 to connect queue
	2012-07-17 19:12:43.794 INFO com.couchbase.client.CouchbaseConnection:  Connection state changed for sun.nio.ch.SelectionKeyImpl@1d179fb
	[info] play - Application started (Dev)

Note that since we haven't defined a `viewmode` setting, Couchbase assumes you're running in production mode. You can change this setting, but I haven't done this here (we'll push our views into production before we use them inside the application).

If you press `ctrl+d` to stop the application, you can see that Couchbase correctly closes the connection:

	[info] application - Application stopped
	2012-07-17 19:24:52.216 INFO com.couchbase.client.CouchbaseConnection:  Shut down Couchbase client
	2012-07-17 19:24:52.226 INFO com.couchbase.client.ViewConnection:  Shut down Couchbase client
	2012-07-17 19:24:52.226 INFO com.couchbase.client.ViewNode:  Couchbase I/O reactor terminated

## Wrapping up
This post hopefully gave you an overview of what we're going to implement, namely a real-time web chat application. We're going to use Java, the play! framework and Couchbase. Also we've set up the database, the project itself and connected both. In the next post we'll build on this foundation and add session management to let users sign in and out.

As always, comment below if you have any thoughts on this!