+++
date = "2012-06-21"
draft = false
title = "Using Couchbase as a flexible session store"
description = "This post digs into the details of storing and querying session data with Couchbase."
tags = ["php","couchbase","session"]
slug = "Using-Couchbase-as-a-flexible-session-store"
+++

This post shows you how you can use the [Couchbase 2.0 server](http://www.couchbase.com) as a flexible session store. What do I mean by flexible? Well, the combination of a highly scalable key-value store and the possibility to query your data through views allows you to gain unique insight inside your data in near realtime.

Let's look at some obstacles that we as application developers face and then see how we may solve them through Couchbase and its functionality. Of course, there are lots of ways to solve the same problem domain and the solutions provided here may not fit your problem as good as another technology/stack/software. I'd love to hear your opinions on this if you have a more elegant or flexible approach with a similar technology.

## Introduction
The root of all session storage problems is that most web applications need to store some kind of state across pages and HTTP is by design [stateless](http://en.wikipedia.org/wiki/Stateless_protocol). Therefore, we need to use some of the solutions that were invented to fill exactly this gap.

As a developer, you can choose between storing session data either on the client-side or on the server-side. The client-side usually is a synonym for [cookies](http://en.wikipedia.org/wiki/HTTP_cookie), but new technologies like [HTML 5 client-side storage](http://www.html5rocks.com/en/tutorials/offline/storage/) slowly emerge. Since cookies are stored on the client-side, they are not under complete control of the application and are therefore a possible security issue (and have size limitations that may be too small for your requirements). Solutions like [encryption](http://nitschinger.at/Session-Encryption-with-Lithium) and [HMAC](http://en.wikipedia.org/wiki/HMAC) exist, but still they lack the dynamic part of backend databases.

On the server-side, you can usually choose between either a storage mechanism provided by your language/application server or a standalone software. For example, PHP provides a [$_SESSION](http://www.php.net/manual/en/reserved.variables.session.php) superglobal that stores data across requests.  In Java, you can work with [HttpSession](http://docs.oracle.com/cd/E17802_01/webservices/webservices/docs/1.6/api/javax/servlet/http/HttpSession.html) objects that allow you to do the same. All these mechanisms have one limitation: they are bound to the application container/webserver where the application is executed (therefore, they are called "sticky"). This means that when we need to use more than one application server to scale out at the same time and a [proxy](http://haproxy.1wt.eu/) (or load balancer) routes the requests randomly, it is possible that application server 1 has no idea what sessions are active on application server 2. As a result, a user may be logged out on the next request.

To overcome this issue, we need to refactor the session handling part out of the application server into a standalone software. This allows our application servers to read and write from a central session store and as a result all of them share the same session state.

One of the oldest and most used software solutions is [memcached](http://memcached.org/). Memcached was developed by [Danga Interactive](http://www.danga.com/) to remedy the scalability issues at [LiveJournal](http://www.livejournal.com/). It provides a distributed memory object caching system that has a key-based lookup mechanism and is fast because all data is stored in-memory. While memcached is amazing, it has one major limitation: it doesn't persist the data on disk, so if one server goes down it may take some time to prime the cache again (called warm-up time).

Some developers identified the additional need and started a full protocol-compatible project that was eventually called [Membase](http://en.wikipedia.org/wiki/Membase). Aside from disk persistence, Membase provided more functionality like replication and live cluster reconfiguration. In 2011, Membase merged with CouchOone (the main driver behind [Apache CouchDB](http://couchdb.apache.org/)) to form [Couchbase](http://www.couchbase.com/). Couchbase 1.8 is basically a renamed and slightly enhanced version of Membase, while Couchbase 2.0 (currently in developer preview release) adds incremental map/reduce, distributed indexing and cross-cluster replication.

That's the short story on the origins of Couchbase. Note that there are lots of other awesome projects out there that help you achieve similar results, with [Redis](http://redis.io/), [MongoDB](http://www.mongodb.org/) or [Riak](http://wiki.basho.com/) as the more prominent ones. Espcially Redis provides more powerful data structures on top of the "key/value" pattern and is definitely worth a look.

## Implementation
The most simple use case that all session stores need to fulfill is - of course - to store and read session data in a very performant way. Most of the time, a unique session identifier is used as the key and some arbitrary payload as the value (for example usernames, ids or other related data).

Note that this post aims to be a SDK-independent introduction to the topic, and the exact syntax may vary a bit from SDK to SDK (I'm using PHP here, but the syntax is nearly identical for the other SDKs). As the Couchbase API is similar to memcached, you should feel right at home if you've used it already (at least for the basic commands).

Couchbase provides you with `get` and `set` methods to read and store a value identified by a unique key. The key should be a string and the value preferably a JSON object. You can also store serialized or binary data, but you then loose the ability to query it through views. On the `set`-command, you can also pass in an optional expiration time, which comes in handy if you want to expire sessions automatically. Here's an example to store a session document:

	$user = array(
		'username' => 'daschl',
		'fullname' => 'Michael Nitschinger'
	);
	$document = json_encode($user);
	$couchbase->set('user:1234', $document, 60);

This stores the given JSON document with the key `user:1234` and a expiry time of 60 seconds in Couchbase. If you provide a expiry time that is larger than 30 days, Couchbase assumes that you pass in a unix timestamp (as a result, you can also pass absolute timestamps instead of relative ones).

To read it back afterwards, you can use the `get` method:

	$result = $couchbase->get('user:1234');

If you want to delete the key (for example to destroy the session), you can just remove the key:

	$result = $couchbase->delete('user:1234');

You can also just "touch" the document, so that the expiry key will be refreshed:

	$result = $couchbase->touch('user:1234', 60);

Of course that's just the simplest use case, but it is often enough to start with. If you're curious, you can also [look into](http://www.couchbase.com/docs/couchbase-sdk-php-1.1/api-reference-summary.html) `getMulti` and `setMulti` to fetch more keys at the same time. One thing to remember is that when you specify an expiration time the key will only be deleted on the next fetch and not automatically. If you have a large batch of keys that you want to remove you need some kind of background script that touches them and makes sure they are actually deleted. Note that you can also wait for the Couchbase background job to delete the expired keys for you which runs every hour by default.

Now, all of this can be done with nearly every key/value store out there (let's leave the other capabilities like replication and persistence out for now). Since CouchOne merged with Membase, Couchbase 2.0 includes a key feature of CouchDB: accessing your data through views.

Let's assume we want to write an admin dashboard that allows us to monitor some of our application KPIs to help us understand the current load it is facing. To achieve this, we want to graph the following information in realtime:

- How many users are currently active on the website.
- How many users are currently logged in.
- All users that come from Vienna/Austria (since our website sells local food here in Austria).

In order to achieve this, we need some way to differentiate if a user is just visiting the website or is logged in. Also, we need to store the location from where he is coming. Since this a more-or-less SDK-agnostic post, here are some hints how this could be done:

When an anonymous user requests our website, we initiate a session that identifies him by his ip-address and does a lookup of his location based on some external information (for example the [MaxMind GeoLite database](http://www.maxmind.com/app/geolite)). We store a session identifier in a cookie and the following document in Couchbase:

	// Key: user:<SESSION-IDENTIFIER>
	{
		"namespace": "sessions",
		"type": "anonymous",
		"lastSeen": "<timestamp>",
		"firstSeen": "<timestamp>",
		"remoteAddress": "<ipaddr>",
		"location": "Vienna/Austria"
	}

If the user is logged in, we can clear the old session and instantiate a new one with more details about our user:

	// Key: user:<SESSION-IDENTIFIER>
	{
		"namespace": "sessions",
		"type": "user",
		"userID": "<userID>",
		"lastSeen": "<timestamp>",
		"firstSeen": "<timestamp>",
		"remoteAddress": "<ipaddr>",
		"location": "Vienna/Austria",
		"name": "<full name>"
	}

The whole store, delete and update calls can be done with the basic methods described above, but what I want to concentrate on are the views. If you want to follow along, here is a short PHP script that populates the default bucket with some data to work with:

	<?php
		// Open the Connection
		$couchbase = new Couchbase("127.0.0.1:8091");

		// Create some fake data with random hashes to simulate session identifiers.
		$data = array(
			'68ea304124beaa7e648cc1327d793b0a' => array(
				'namespace' => 'sessions',
				'type' => 'anonymous',
				'lastSeen' => time(),
				'firstSeen' => time(),
				'remoteAddress' => '192.168.0.1',
				'location' => 'Vienna/Austria'
			),
			'6254b64d554a88b629bdbfc507d8c267' => array(
				'namespace' => 'sessions',
				'type' => 'anonymous',
				'lastSeen' => time(),
				'firstSeen' => time()-3600,
				'remoteAddress' => '10.10.10.10',
				'location' => 'Berlin/Germany'
			),
			'6148bae4fad02c0edfd7dbd8d54d79f1' => array(
				'namespace' => 'sessions',
				"type" => "user",
				"userID" => "1234",
				'lastSeen' => time()-7200,
				'firstSeen' => time()-7800,
				"remoteAddress" => "1.2.3.4",
				"location" => "Rome/Italy",
				"name" => "Luigi Manfrotto"
			),
			'4aeb265e299c41450dd6af813d0bef97' => array(
				'namespace' => 'sessions',
				"type" => "user",
				"userID" => "1104",
				'lastSeen' => time(),
				'firstSeen' => time(),
				"remoteAddress" => "2.3.4.5",
				"location" => "Vienna/Austria",
				"name" => "Hans Huber"
			)
		);

		// Store the documents in Couchbase
		foreach($data as $hash => $document) {
			$couchbase->set("user:$hash", json_encode($document));
		}
	?>

Now that we have four documents to work with, we can start creating views from the Couchbase GUI. Head over to the "Views" section and click on "Create Development View". The way Couchbase works is that you write your view queries in development mode on a subset of your documents and when you're finished you can publish them to production. Since we have only four documents in the bucket, it won't make any difference here.

The name of our design document is `_design/dev_sessions` and the first view is called `active`. This one will return all currently logged in users:

	function (doc) {
		if(doc.namespace == 'sessions') {
			emit(doc._id, doc);
		}
	}

If you save it and click `Show Results`, then you'll see all four documents below that got matched. You can also click on the URI string above `?connection_timeout=60000&limit=10&skip=0` to see the actual response in your browser. But what if we just want the total number of users logged in? We can use a built-in reduce function for that. In the "Reduce" textarea to the right just enter `_count`. If you click save and then show the results again, you can see that we just got one document back with no key and just `4` as the value.

The nice thing here is that we can pass `reduce=true/false` as a param and then get either the full response back or just the aggregated version. So this query gives us all active users on the website. Let's write another view that just returns the logged in users:

Modify the view like here and then click `Save As...` and name it `logged_in`:

	function (doc) {
		if(doc.namespace == 'sessions' && doc.type == 'user') {
			emit(doc._id, doc);
		}
	}

The reduce function can be reused in the same way. As you can see, with just a bit more code in JavaScript we have another view that just returns the logged in users. Now we want to display a third diagram that displays all users from `Vienna/Austria`. This time, we want to differentiate between anonymous and logged in users so that they can be displayed in two distinct graphs. Our map function may look like this:

	function (doc) {
		if(doc.namespace == 'sessions' && doc.location == 'Vienna/Austria') {
			emit(doc.type, 1);
		}
	}

Every time we find a document with the correct location, we emit a document with the key of either `anonymous` or `user`. As values we supply a value of `1` since we can reuse that in our reduce function. The reduce function is again the built in `_count` function, but this time we pass an additional param `group_level` with a value of `1` to our query. This way, Couchbase aggregates our data and returns a document like this:

	// Params: ?group_level=1&reduce=true&connection_timeout=60000&limit=10&skip=0
	{
		"rows": [
			{"key":"anonymous","value":1},
			{"key":"user","value":1}
		]
	}

If we insert another `user` document the counter would display `2`. If we don't use the grouping level setting, Couchbase would just return the total amount of documents emitted.

All of these views assumed that we have some kind of realtime polling to our backend, because the results didn't include grouping by a distinct timestamp. In order to load a graph with meaningful results for the last hour, we can return it aggregated by timestamp (we can reuse the same `_count` reduce function and the level 1 grouping):

	function (doc) {
		if(doc.namespace == 'sessions') {
			emit(doc.lastSeen, 1);
		}
	}

An example output would look like this:

	// Params: ?group_level=1&connection_timeout=60000&limit=10&skip=0
	{
		"rows":[
			{"key":1339997680,"value":1},
			{"key":1340004880,"value":3}
		]
	}

Now, if we have record for the last year in our database it won't make much sense to fetch them all on every request. If you look one entry below the `group_level` param, you'll find `startkey` and `endkey` options. With these two you can provide a timerange to fetch the results.

## Performance
I know that synthetic benchmarks never provide reproducible results in production environments, but they provide at least a rough estimation on how the application may be able to perform under certain conditions.

Thankfully, [Trond Norbye](http://trondn.blogspot.co.at/) added a nice little program to [libcouchbase](http://www.couchbase.com/develop/c/next) that allows you to generate load on your Couchbase server and then analyze the response times directly on the console (of course you can get real-time metrics from the Couchbase GUI as well). The program is called `pillowfight` and is distributed with the source of libcouchbase. It stores a bunch of keys (1000 by default) and then runs "mget" operations on them and measures the time they need to get returned. In order to run it, you need to compile it from source (which I'll show in a later post but it's not that hard if you are used to compiling on unix-based systems).

On my middle-class Dell notebook with an average processor (Core2 Duo P8700), 4 gigs of RAM and a SSD drive, 90% of the keys return in about 70 microseconds. If I look at the Couchbase GUI for my bucket, I can see that my instance runs somewhere between 10.000 and 12.000 operations per second. I recommend you to run this program on your dev machine just for fun and to get a feeling how it performs. It is also a handy way to generate a constant load on your Couchbase instances and add a large number of keys in an easy way (you can find more params with the -? switch).

If you ran the program, it would be cool to share your results in the comments below.

## Wrapping Up
From both its history and current feature set, chances are that Couchbase is a possible solution to your session store needs. It provides you with both fast and flexible querying mechanisms and is capable to scale up to more nodes with just a few clicks if you need it to. There are definitely other good storage systems out there (like redis), so make sure to dig deeper into the concepts to find out if Couchbase is right for your application.

As Couchbase 2.0 comes closer and closer to the final release, I think it's a good time to look into the system and see what it's capable of. Also, if you need any help I found that the guys behind Couchbase are very friendly and helpful. You can also ping me via twitter if you like!

If you're just getting your feet wet with Couchbase, I also have two introductory posts on [PHP](http://nitschinger.at/Getting-Started-with-Couchbase-and-PHP) and [Scala](http://nitschinger.at/Accessing-Couchbase-from-Scala) that you may find helpful!

Finally, kudos to [@ingenthr](http://twitter.com/ingenthr) for reviewing this post and providing valuable insight into Couchbase.