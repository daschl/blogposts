+++
date = "2013-04-17"
draft = false
title = "Couchbase Java SDK Internals"
description = "Learn about how the Java SDK works internally!"
tags = ["java","couchbase"]
slug = "Couchbase-Java-SDK-Internals"
+++

## Motivation
This blog post is intended to be a very detailed and informative article for those who already have used the Couchbase Java SDK and want to know how the internals work. This is not a introduction on how to use the Java SDK and we'll cover some fairly advanced topics on the way.

Normally, when talking about the SDK we mean everything that is needed to get you going (Client library, documentation, release notes,...). In this article though, the SDK refers to the Client library (code) unless stated otherwise.

As always, if you have feedback please let me/us know!

## Introduction
First and foremost, it is important to understand that the SDK wraps and extends the functionality of the [spymemcached](https://github.com/couchbase/spymemcached) (called "spy") memcached library. One of the protocols used internally is the memcached protocol, and a lot of functionality can be reused. On the other hand, once you start to peel off the first layers of the SDK you will notice that some components are somewhat more complex because of the fact that spy provides more features than the SDK needs in the first place. The other part is to remember that a lot of the components are interwoven, so you always need to get the dependency right. Most of the time, we release a new spy version at the same date with a new SDK, because new stuff has been added or fixed.

So, aside from reusing the functionality provided by spy, the SDK mainly adds two blocks of functionality: automatic cluster topology management and since 1.1 (and 2.0 server) support for Views. Aside from that it also provides administrative facilities like bucket and design document management.

To understand how the client operates, we'll dissect the whole process in different life cycle phases of the client. After we go through all three phases (bootstrap, operation and shutdown) you should have a clear picture of whats going on under the hood. Note that there is a separate blog post in the making about error handling, so we won't cover that here in greater detail (which will be published a few weeks later on the same blog here).

## Phase 1: Bootstrap
Before we can actually start serving operations like `get()` and `set()`, we need to bootstrap the `CouchbaseClient` object. The important part that we need to accomplish here is to initially get a cluster configuration (which contains the nodes and vBucket map), but also to establish a streaming connection to receive cluster updates in (near) real-time.

We take the list of nodes passing during bootstrap and iterate over it. The first node in the list that can be contacted on port 8091 is used to walk the RESTful interface on the server. If it is not available, the next one will be tried. This means that going from the provided `http://host:port/pools` URI we eventually follow the links to the bucket entity. All this happens inside a `ConfigurationProvider`, which is in this case the `com.couchbase.client.vbucket.ConfigurationProviderHTTP`. If you want to poke around on the internals, look for `getBucketConfiguration` and `readPools` methods.

A (successful) walk can be illustrated like this:

 - GET /pools
 - look for the "default" pools
 - GET /pools/default
 - look for the "buckets" hash which contains the bucket list
 - GET /pools/default/buckets
 - parse the list of buckets and extract the one provided by the application
 - GET /pools/default/buckets/<bucketname>

Now we are at the REST endpoint we need. Inside this JSON response, you'll find all useful details that gets also be used by SDK internally (for example `stre	amingUri`, `nodes` and `vBucketServerMap`). The config gets parsed and stored. Before we move on, let's quickly discuss the strange `pools` part inside our REST walk:

The concept of a resource pool to group buckets was designed for Couchbase Server, but is currently not implemented. Still, the REST API is implemented that way and therefore all SDKs support it. That said, while we could theoretically just go directly to `/pools/default/buckets` and skip the first few queries, the current behaviour is future proof so you won't have to change the bootstrap code once the server implements it.

Back to our bootstrap phase. Now that we have a valid cluster config which contains all the nodes (and their hostnames or ip addresses), we can establish connections to them. Aside from establishing the data connections, we also need to instantiate a streaming connection to one of them. For simplicity reasons, we just establish the streaming connection to the node from the list where we got our initial configuration.

This gets us to an important point to keep in mind: if you have lots of CouchbaseClient objects running on many nodes and they all get bootstrapped with the same list, they may end up connecting to the same node for the streaming connection and create a possible bottleneck. Therefore, to distribute the load a little better I recommend shuffling the array before it gets passed in to the CouchbaseClient object. When you only have a few CouchbaseClient objects connected to your cluster, that won't be a problem at all.

The streaming connection URI is taken from the config we got previously, and normally looks like this:

	streamingUri: "/pools/default/bucketsStreaming/default?bucket_uuid=88cae4a609eea500d8ad072fe71a7290"

If you point your browser to this address, you will also get the cluster topology updates streamed in real-time. Since the streaming connection needs to be established all the time and potentially blocks a thread, this is done in the background handled by different threads. We are using the NIO framework [Netty](http://netty.io) for this task, which provides a very handy way of dealing with asynchronous operations. If you want to start digging into this part, keep in mind that all read operations are completely separate from write operations, so you need to deal with handlers that take care of what comes back from the server. Aside from some wiring needed for Netty, the business logic can be found in `com.couchbase.client.vbucket.BucketMonitor` and `com.couchbase.client.vbucket.BucketUpdateResponseHandler`. We also try to reestablish this streaming connection if the socket gets closed (for example if this node gets rebalanced out of the cluster). 

To actually shuffle data to the cluster nodes, we need to open various sockets to them. Note that there is absolutely no connection pooling needed inside the client, because we manage all sockets proactively. Aside from the special streaming connection to one of the severs (which is opened against port 8091), we need to open the following connections:

 - Memcached Socket: Port 11210
 - View Socket: Port 8092

Note that port 11211 is not used inside the client SDKs, but used to connect generic memcached clients that are not cluster aware. This means that these generic clients do not get updated cluster topologies.

So as a rule of thumb, if you have a 10 node cluster running, one CouchbaseClient object open about 21 (2*10 + 1) client sockets. These are directly managed, so if a node gets removed or added the numbers will change accordingly.

Now that all sockets have been opened, we are ready to perform regular cluster operations. As you can see, there is a lot of overhead involved when the CouchbaseClient object gets bootstrapped. Because of this fact, we strongly discourage you from either creating a new object on every request or running a lot of CouchbaseClient objects in one application server. This only adds unnecessary overhead and load on the application server and adds on the total sockets opened against the cluster (resulting in a possible performance problem).

As a point of reference, with regular INFO level logging enabled this is how connecting and disconnecting to a 1-node cluster (Couchbase bucket) should look like:

	Apr 17, 2013 3:14:49 PM com.couchbase.client.CouchbaseProperties setPropertyFile
	INFO: Could not load properties file "cbclient.properties" because: File not found with system classloader.
	2013-04-17 15:14:49.656 INFO com.couchbase.client.CouchbaseConnection:  Added {QA sa=/127.0.0.1:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	2013-04-17 15:14:49.673 INFO com.couchbase.client.CouchbaseConnection:  Connection state changed for sun.nio.ch.SelectionKeyImpl@2adb1d4
	2013-04-17 15:14:49.718 INFO com.couchbase.client.ViewConnection:  Added localhost to connect queue
	2013-04-17 15:14:49.720 INFO com.couchbase.client.CouchbaseClient:  viewmode property isn't defined. Setting viewmode to production mode
	2013-04-17 15:14:49.856 INFO com.couchbase.client.CouchbaseConnection:  Shut down Couchbase client
	2013-04-17 15:14:49.861 INFO com.couchbase.client.ViewConnection:  Node localhost has no ops in the queue
	2013-04-17 15:14:49.861 INFO com.couchbase.client.ViewNode:  I/O reactor terminated for localhost

If you are connecting to a Couchbase Server 1.8 or against a Memcache-Bucket you won't see View connections getting established:


	INFO: Could not load properties file "cbclient.properties" because: File not found with system classloader.
	2013-04-17 15:16:44.295 INFO com.couchbase.client.CouchbaseConnection:  Added {QA sa=/192.168.56.101:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	2013-04-17 15:16:44.297 INFO com.couchbase.client.CouchbaseConnection:  Added {QA sa=/192.168.56.102:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	2013-04-17 15:16:44.298 INFO com.couchbase.client.CouchbaseConnection:  Added {QA sa=/192.168.56.103:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	2013-04-17 15:16:44.298 INFO com.couchbase.client.CouchbaseConnection:  Added {QA sa=/192.168.56.104:11210, #Rops=0, #Wops=0, #iq=0, topRop=null, topWop=null, toWrite=0, interested=0} to connect queue
	2013-04-17 15:16:44.306 INFO com.couchbase.client.CouchbaseConnection:  Connection state changed for sun.nio.ch.SelectionKeyImpl@38b5dac4
	2013-04-17 15:16:44.313 INFO com.couchbase.client.CouchbaseClient:  viewmode property isn't defined. Setting viewmode to production mode
	2013-04-17 15:16:44.332 INFO com.couchbase.client.CouchbaseConnection:  Connection state changed for sun.nio.ch.SelectionKeyImpl@69945ce
	2013-04-17 15:16:44.333 INFO com.couchbase.client.CouchbaseConnection:  Connection state changed for sun.nio.ch.SelectionKeyImpl@6766afb3
	2013-04-17 15:16:44.334 INFO com.couchbase.client.CouchbaseConnection:  Connection state changed for sun.nio.ch.SelectionKeyImpl@2b2d96f2
	2013-04-17 15:16:44.368 INFO net.spy.memcached.auth.AuthThread:  Authenticated to 192.168.56.103/192.168.56.103:11210
	2013-04-17 15:16:44.368 INFO net.spy.memcached.auth.AuthThread:  Authenticated to 192.168.56.102/192.168.56.102:11210
	2013-04-17 15:16:44.369 INFO net.spy.memcached.auth.AuthThread:  Authenticated to 192.168.56.101/192.168.56.101:11210
	2013-04-17 15:16:44.369 INFO net.spy.memcached.auth.AuthThread:  Authenticated to 192.168.56.104/192.168.56.104:11210
	2013-04-17 15:16:44.490 INFO com.couchbase.client.CouchbaseConnection:  Shut down Couchbase client


## Phase 2: Operations
When the SDK is bootstrapped, it enables your application to run operations against the attached cluster. For the purpose of this blog post, we need to distinguish between operations that get executed against a stable cluster and operations on a cluster that is currently experiencing some form of topology change (be it planned because of adding nodes or unplanned because of a node failure). Let's tackle the regular operations first.

### Operations against a stable cluster
While not directly visible in the first place, inside the SDK we need to distinguish between memcached operations and View operations. All operations that have a unique key in their method signature can be treaded as memcached operations. All of them eventually end up getting funneled through spy. View operations on the other hand are implemented completely inside the SDK itself.

Both View and memcached operations are asynchronous. Inside spy, there is one thread (call the I/O thread) dedicated to deal with IO operations. Note that in high-traffic environments, its not unusual that this thread is always active. It uses the non-blocking Java NIO mechanisms to deal with traffic, and loops around "selectors" that get notified when data can either be written or read. If you profile your application you'll see that this thread spends most of its time waiting on a `select` method, it means that it is idling there waiting to be notified for new traffic. The concepts used inside spy to deal with this are common Java NIO knowledge, so you may want to look into the [NIO internals](https://www.ibm.com/developerworks/java/tutorials/j-nio/) first before digging into that code path. Good starting points are the `net.spy.memcached.MemcachedConnection` and `net.spy.memcached.protocol.TCPMemcachedNodeImpl` classes. Note that inside the SDK, we override the `MemcachedConnection` to hook in our own reconfiguration logic. This class can be found inside the SDK at `com.couchbase.client.CouchbaseConnection` and for memcached-type buckets in `com.couchbase.client.CouchbaseMemcachedConnection`.

So if a memcached operations (like `get()`) gets issued, it gets passed down until it reaches the IO thread. The IO thread will then put it on a write queue towards its target node. It gets written eventually and then the IO thread adds information to a read queue so the responses can be mapped accordingly. This approach is based on futures, so when the result actually arrives, the Future is marked as completed, the result gets parsed and attached as Object.

The SDK only uses the memcached binary protocol, although spy would also support ASCII. The binary format is much more efficient and some of the advanced operations are only implemented there.

You may wonder how the SDK knows where to send the operation? Since we already have the up-to-date cluster map, we can hash the key and then based on the node list and vBucketMap determine which node to access. The vBucketMap not only contains the information for the master node of the array, but also the information for zero to three replica nodes. Look at this (shortened) example:

	vBucketServerMap: {
	hashAlgorithm: "CRC",
	numReplicas: 1,
	serverList: [
		"192.168.56.101:11210",
		"192.168.56.102:11210"
	],
	vBucketMap: [
	[0,1],
	[0,1],
	[0,1],
	[1,0],
	[1,0],
	[1,0]
	//.....
	},

The `serverList` contains our nodes, and the `vBucketMap` has pointers to the `serverList` array. We have 1024 vBuckets, so only some of them are shown here. You can see from looking at it that all keys that has into the first vBucket have its master node at index 0 (so the `.101` node) and its replica at index 1 (so the `.102` node). Once the cluster map changes and the vBuckets move around, we just need to update our config and know all the time where to point our operations towards.

View operations are handled differently. Since views can't be sent to a specific node (because we don't have a way to hash a key or something), we round-robin between the connected nodes. The operation gets assigned to a [com.couchbase.client.ViewNode](http://www.couchbase.com/autodocs/couchbase-java-client-1.1.5/com/couchbase/client/ViewNode.html) once it has free connections and then executed. The result is also handled through futures. To implement this functionality, the SDK uses the third party Apache HTTP Commons (NIO) library.

The whole View API hides behind port 8092 on every node and is very similar to [CouchDB](http://couchdb.apache.org/). It also contains a RESTful API, but the structure is a little bit different. For example, you can reach a design document at `/<bucketname>/_design/<designname>`. It contains the View definitions in JSON:

	{
		language: "javascript",
		views: {
			all: {
				map: "function (doc) { if(doc.type == "city") {emit([doc.continent, doc.country, doc.name], 1)}}",
				reduce: "_sum"
			}
		}
	}

You can then reach down one level further like `/<bucketname>/_design/<designname>/_view/<viewname>` to actually query it:

	{"total_rows":9,"rows":[
	{"id":"city:shanghai","key":["asia","china","shanghai"],"value":1},
	{"id":"city:tokyo","key":["asia","japan","tokyo"],"value":1},
	{"id":"city:moscow","key":["asia","russia","moscow"],"value":1},
	{"id":"city:vienna","key":["europe","austria","vienna"],"value":1},
	{"id":"city:paris","key":["europe","france","paris"],"value":1},
	{"id":"city:rome","key":["europe","italy","rome"],"value":1},
	{"id":"city:amsterdam","key":["europe","netherlands","amsterdam"],"value":1},
	{"id":"city:new_york","key":["north_america","usa","new_york"],"value":1},
	{"id":"city:san_francisco","key":["north_america","usa","san_francisco"],"value":1}
	]
	}

Once the request is sent and a response gets back, it depends on the type of View request to determine on how the response gets parsed. It makes a difference, because reduced View queries look different than non-reduced. The SDK also includes support for spatial Views and they need to be handled differently as well.

The whole View response parsing implementation can be found inside the [com.couchbase.client.protocol.views](http://www.couchbase.com/autodocs/couchbase-java-client-1.1.5/com/couchbase/client/protocol/views/package-frame.html) namespace. You'll find abstract classes and interfaces like `ViewResponse` in there, and then their special implementations like `ViewResponseNoDocs`, `ViewResponseWithDocs` or `ViewResponseReduced`. It also makes a different if `setIncludeDocs()` is used on the Query object, because the SDK also needs to load the full documents using the memcached protocol behind the scenes. This is also done while parsing the Views.

Now that you have a basic understanding on how the SDK distributes its operations under stable conditions, we need to cover an important topic: how the SDK deals with cluster topology changes.

### Operations against a rebalancing cluster
Note that there is a separate blog post upcoming dealing with all the scenarios that may come up when something goes wrong on the SDK. Since rebalancing and failover are crucial parts of the SDK, this post deals more with the general process on how this is handled.

As mentioned earlier, the SDK receives topology updates through the streaming connection. Leaving the special case aside where this node actually gets removed or fails, all updates will stream in near real-time (in a eventually consistent architecture, it may take some time until the cluster updates get populated to that node). The chunks that come in over the stream look exactly like the ones we've seen when reading the initial configuration. After those chunks have been parsed, we need to check if the changes really affect the SDK (since there are many more parameters than the SDK needs, it won't make sense to listen to all of them). All changes that affect the topology and/or vBucket map are considered as important. If nodes get added or removed (be it either through failure or planned), we need to open or close the sockets. This process is called "reconfiguration".

Once such a reconfiguration is triggered, lots of actions need to happen in various places. Spymemcached needs to handle its sockets, View nodes need to be managed and new configuration needs to be updated. The SDK makes sure that only one reconfiguration can happen at the same time through locks so we don't have any race conditions going on.

The Netty-based `BucketUpdateResponseHandler` triggers the `CouchbaseClient#reconfigure` method, which then starts to dispatch everything. Depending on the bucket type used (i.e. memcached type buckets don't have Views and therefore no ViewNodes), configs are updated and sockets closed. Once the reconfiguration is done, it can receive new ones. During planned changes, everything should be pretty much controlled and no operations should fail. If a node is actually down and cannot be reached, those operations will be cancelled. Reconfiguration is tricky because the topology changes while operations are flowing through the system. 

Finally, let's cover some differences between Couchbase and Memcache type buckets. All the information hat you've been reading previously only applies to Couchbase buckets. Memcache buckets are pretty basic and do not have the concept of vBuckets. Since you don't have vBuckets, all that the Client has to do is to manage the nodes and their corresponding sockets. Also, a different hashing algorithm is used (mostly Ketama) to determine the target node for each key. Also, memcache buckets don't have views, so you can't use the View API and it doesn't make much sense to keep View sockets around. So to clarify the previous statement, if you are running against a memcache bucket, for a 10 node cluster you'll only have 11 open connections.

Phase 3: Shutdown
-----------------
Once the `CouchbaseClient#shutdown()` method is called, no more operations are allowed to be added onto the `CouchbaseConnection`. Until the timeout is reached, the client wants to make sure that all operations went through accordingly. All sockets for both memcached and View connections are shut down once there are no more operations in the queue (or they get dropped). Note that that the `shutdown` methods on those sockets are also used when a node gets removed from the cluster during normal operations, so it's basically the same, but just for all attached nodes at the same time.

Summary
-------
After reading this blog post, you should have a much more clear picture on how the client SDK works and why it is designed the way it is. We have lots of enhancements planned for future releases, mostly enhancing the direct API experience. Note that this blog post didn't cover how errors are handled inside the SDK; this will be published in a separate blog post because there is also lots of information to cover.
