+++
date = "2013-03-05"
draft = false
title = "Never awaitUninterruptibly() on Netty Channels"
description = "A quick tip about waiting on Channels in Netty."
tags = ["java","couchbase","netty"]
slug = "Never-await-Uninterruptibly-on-Netty-Channels"
+++

TL:DR; When acquiring [Channels](http://netty.io/3.6/api/org/jboss/netty/channel/Channel.html) in [Netty](http://netty.io), always use a [ChannelFutureListener](http://netty.io/3.6/api/org/jboss/netty/channel/ChannelFutureListener.html) and never [awaitUninterruptibly()](http://docs.jboss.org/netty/3.2/api/org/jboss/netty/channel/ChannelFuture.html#awaitUninterruptibly()). Curious why? Read on.

In the [Java SDK](http://www.couchbase.com/develop/java/current) for [Couchbase](http://www.couchbase.com/), we use Netty to establish and maintain a streaming connection to one of the cluster nodes in order to get notified when topology changes happen. This streaming connection needs to be established during the bootstrap process of the client and we need to block until the connection is established (actually we don't need to, but the current implementation works that way). The old implementation to acquire the Channel looked like this:

    ClientBootstrap bootstrap = new ClientBootstrap(factory);
    bootstrap.setPipelineFactory(new BucketMonitorPipelineFactory());
    ChannelFuture future = bootstrap.connect(new InetSocketAddress(host, port));

    channel = future.awaitUninterruptibly().getChannel();
    if (!future.isSuccess()) {
      bootstrap.releaseExternalResources();
      throw new ConnectionException("Something bad happened...");
    }

This works great, but there is a problem associated that is not obvious in the first place. As long as you use this code only in a client side context, Netty will not complain and happily work with your code. When people started to use our client library inside a Netty based server framework (for example [Play](http://www.playframework.com/)), Netty complained like this:

	Unexpected exception[IllegalStateException: await*() in I/O thread causes a dead lock or sudden performance drop. Use addListener() instead or call await*() from a different thread.]

The environment where this happens is clearly defined: we are bootstrapping a Netty client inside the I/O thread of a Netty server, so we basically have two Netty environments running and one is complaining about the other. Once you are aware of this situation, it is more or less easy to fix:

    bootstrap = new ClientBootstrap(factory);
    bootstrap.setPipelineFactory(new BucketMonitorPipelineFactory());
    ChannelFuture future =  bootstrap.connect(new InetSocketAddress(host, port));
    channelFuture.addListener(new ChannelFutureListener() {
      @Override
      public void operationComplete(ChannelFuture cf) throws Exception {
        if(cf.isSuccess()) {
          channel = cf.getChannel();
        } else {
          bootstrap.releaseExternalResources();
          throw new ConnectionException("Something bad happened...");
        }
      }
    });

Now, instead of waiting on the caller thread, we move the waiting part to a separate thread managed by the Netty execution context. There's only one problem left: we still need to block, because the code down the stack depends on a established Channel to work with. To solve this issue, we can use a [CountDownLatch](http://docs.oracle.com/javase/6/docs/api/java/util/concurrent/CountDownLatch.html) like this:

    final CountDownLatch channelLatch = new CountDownLatch(1);
    channelFuture.addListener(new ChannelFutureListener() {
      @Override
      public void operationComplete(ChannelFuture cf) throws Exception {
        if(cf.isSuccess()) {
          channel = cf.getChannel();
          channelLatch.countDown();
        } else {
          bootstrap.releaseExternalResources();
          throw new ConnectionException("Something bad happened...");
        }
      }
    });

    try {
      channelLatch.await();
    } catch(InterruptedException ex) {
      throw new ConnectionException("Interrupted while waiting for streaming "
        + "connection to arrive.");
    }

In the end we still block on the caller thread, but we are compliant with Netty. The main takeaway for me is that you should never block on acquiring Channels in Netty, just because of the fact that your client side code may be used in a server side context as well. This is especially true for library developers like me.