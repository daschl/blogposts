+++
date = "2013-08-13"
draft = false
title = "Using the Reactor Processor for High-Performance TCP"
description = "This blog post shows how you can use the new Reactor library to create high performant TCP clients."
tags = ["java","couchbase","reactor","tcp"]
slug = "Using-the-Reactor-Processor-for-High-Performance-TCP"
+++

First, a disclaimer: the all-new [Reactor](https://github.com/reactor/reactor) framework is still under heavy development, but it already provides a very promising basement for applications and libraries that need high throughput and low latency. We at [Couchbase](http://www.couchbase.com/) aim to provide the highest throughput at the lowest latency, so it is very critical to build upon an infrastructure that can provide it. Current, we are performing early investigations for a possible "next generation Java SDK" and Reactor seems very promising so far.

This blog post shows you how to quickly set up a Processor (we'll see in a minute what that is) that dispatches requests to Consumers (also a very common term in Reactor). In our case, the Consumer is actually a TCP socket. Please note that the actual numbers, while impressive, can't be used for real world measurements. What you'll see here is a raw throughput test to define a baseline what can be expected under ideal conditions. We are also using localhost here to avoid network latency (which is the bottleneck for most network applications).

I'm going to use a Couchbase server as an endpoint, but feel free to use whatever you want instead. The whole API is very generic and the consumers can be exchanged easily.

# Setup
Before we can get started, all we need to do is include the `reactor-tcp` artifact from maven. Now you can do this through gradle, maven, ivy or what you want, but at this point, I would recommend you to check out the [reactor](https://github.com/reactor/reactor) project directly from github and build it on your own, so you'll have the latest and greatest code in your local repository:

	$ git clone https://github.com/reactor/reactor.git
	$ cd reactor
	$ ./gradlew install

This will download and install everything you need. The next step is to create a maven or gradle project in your IDE, but I'll leave that part up to the reader. The maven dependency you need to include is:

	<dependency>
		<groupId>org.projectreactor</groupId>
		<artifactId>reactor-tcp</artifactId>
		<version>1.0.0.BUILD-SNAPSHOT</version>
	</dependency>

Or, for all gradle folks:

	dependencies {
		compile 'org.projectreactor:reactor-tcp:1.0.0.BUILD-SNAPSHOT'
	}

# The Processor
Now, let's get to some actual code. The Processor is a very lightweight abstraction over the [LMAX Disruptor](https://github.com/LMAX-Exchange/disruptor) a high performant RingBuffer. A RingBuffer provides much better characteristics than a normal Queue, and the Disruptor is heavily optimized for dispatching tasks in the nanosecond area (which means millions of operations per second). I recommend you to read [this paper](https://github.com/LMAX-Exchange/disruptor/blob/master/docs/Disruptor.docx) and also check out talks by [Martin Thompson](http://mechanical-sympathy.blogspot.co.at/) if you are interested.

The basic idea is that we decouple consumers from producers, so that each of them can work on their own pace (not blocking each other) and also benefiting from batching if the producers are faster than the consumers. We'll see why this is particularly important with TCP in a second.

A `Processor` can be created by instantiating a `ProcessorSpec` and defining some mandatory options. Then, the `Processor` is built for us.

```java
ProcessorSpec<Event<Buffer>> writeProcessor = new ProcessorSpec<Event<Buffer>>()
	.dataSupplier(new Supplier<Event<Buffer>>() {
		@Override
		public Event<Buffer> get() {
		  return new Event<Buffer>(new Buffer());
		}
	})
	.consume(new Node("127.0.0.1", environment))
	.get();
```

Note that this `Spec` pattern is very common in reactor, as it allows for very easy and yet flexible object creation. There are a few things that we need to cover.

First, the generic type here is `Event<Buffer>`. The producers will wrap the payload (here we use raw `Buffers`) in an `Event` and the consumers will unwrap and use it properly. The `Event` type also allows for headers that can be used for custom routing, but we won't cover that here.

Second, the `dataSupplier` is a speciality of the Disruptor RingBuffer. In order to minimize garbage collection and make effective use of memory layout, we need to pre-allocate our container objects. They will be reused throughout the application and will never be garbage collected.

Third, through the `consume` method we can tell the `Processor` who will be notified when new data is added to the RingBuffer. In our case, the `Node` represents the TCP client which we'll build in a second.

Now, how do we write to the Processor? Let's add a simple method that does it for us:

```java
public void write(final Buffer data) {
	final Operation<Event<Buffer>> op = writeProcessor.get();
	op.get().setData(data);
	op.commit();
}
```

Here, we get a `Operation` out of the processor (that wraps our data) and override it with the data that we actually want to store this time. You can see that we are not allocating new objects in the RingBuffer, we just use the old one that has been provided for us. With the `commit` method we put it back into the RingBuffer. Actually, behind the scenes, it makes use of Sequences and Barriers inside the RingBuffer, but this is completely hidden from us.

Here is the full code for the lazy reader:

```java
import reactor.core.Environment;
import reactor.core.processor.Operation;
import reactor.core.processor.Processor;
import reactor.core.processor.spec.ProcessorSpec;
import reactor.event.Event;
import reactor.function.Supplier;
import reactor.io.Buffer;

public class MessageDispatcher {

  private final Environment environment;

  private final Processor<Event<Buffer>> writeProcessor;

  public MessageDispatcher() {
    this(new Environment());
  }

  MessageDispatcher(final Environment env) {
    environment = env;

    writeProcessor = new ProcessorSpec<Event<Buffer>>()
      .dataSupplier(new Supplier<Event<Buffer>>() {
        @Override
        public Event<Buffer> get() {
          return new Event<Buffer>(new Buffer());
        }
      })
      .consume(new Node("127.0.0.1", environment))
      .get();

  }

  public void write(final Buffer data) {
    final Operation<Event<Buffer>> op = writeProcessor.get();
    op.get().setData(data);
    op.commit();
  }

}
```

One final note before we move on: did you spot the `Environment` here? This is also a common theme in the Reactor framework. The `Environment` is used in many places to signal information about - who would have thought that - the JVM environment. The general recommendation is to create only one `Environment` instance per JVM, so we happily pass it around in our small application.

# The Consumer
Before we get into the nitty-gritty network details, let's add a consumer that just prints out the data that he "sees". If you want to try this sample, make sure to change the previous code temporarily from `.consume(new Node(...))` to `.consume(new EchoConsumer())`.

```java
public class EchoConsumer implements Consumer<Event<Buffer>> {

  @Override
  public void accept(final Event<Buffer> bufferEvent) {
    System.out.println(Arrays.toString(bufferEvent.getData().asBytes()));
  }

}
```

The `accept` method is always called once there is information available on the Processor. Let's add a simple test case to verify that this works:

```java
import org.junit.Test;
import reactor.io.Buffer;

public class MessageDispatcherTest {

  @Test
  public void echoSomeGarbage() {
    MessageDispatcher dispatcher = new MessageDispatcher();
    dispatcher.write(Buffer.wrap("Hello World!"));
  }

}
```

Now if we run this test, we should see `[72, 101, 108, 108, 111, 32, 87, 111, 114, 108, 100, 33]` printed on the console - great! This is the byte array representation of our wrapped buffer. Of course we could make our lives easier and use `Strings` instead of `Buffer` in our implementation, but the `Buffer` works much better with network communication.

The next step would be to send the data over the network. Let's replace our Consumer with a more intelligent one. Be aware that the `TcpClient` that you'll see doesn't communicate with Java NIO directly - it makes use of the excellent [Netty](http://netty.io/) project which provides a convenient and performant wrapper around NIO and OIO (we use NIO here).

```java
public class Node implements Consumer<Event<Buffer>> {

  private final TcpClient<Buffer, Buffer> client;
  private TcpConnection<Buffer, Buffer> conn = null;

  public Node(String hostname, Environment env) {
    client = new TcpClientSpec<Buffer, Buffer>(NettyTcpClient.class)
      .env(env)
      .connect(hostname, 11210)
      .get();

    try {
      conn = client.open().await();
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

  @Override
  public void accept(final Event<Buffer> bufferEvent) {
    Buffer buf = bufferEvent.getData();

    final CountDownLatch latch = new CountDownLatch(1);
    conn.send(buf, new Consumer<Boolean>() {
      @Override
      public void accept(final Boolean success) {
        latch.countDown();
      }
    });

    try {
      latch.await(1, TimeUnit.SECONDS);
    } catch (final Exception ex) {
      throw new RuntimeException("Something went wrong while waiting :(", ex);
    }

  }

}
```

While this might seem like a lot of code, it's not so bad. We do two things here. During object construction, we create a new `TcpClient` through the `TcpClientSpec` (see the Spec again?) and pass it the environment and the socket to connect to. The next thing we need to do is actually open the connection and wait until finished.

Now that we have an open connection, we can write to it. Since everything is non-blocking in Reactor, so is socket writing. In order to not overwhelm the underlying infrastructure, we have to wait until it is actually finished before moving on to handle the next `Event` in the Processor. We do this by using a `CountDownLatch`, which will be counted down once the writing has finished. In our simple example we just fail if it took longer than one second. In a real application, one could report errors or retry with Backoff.

Before we can run that code, we need to add a test case to make it all work. 

```java
Buffer[] buffers = new Buffer[10];

@Before
public void initBuffers() {
	for (int i = 0; i < 10; i++)  {
		Buffer buf = new Buffer(30, false);
		buf.append((byte)0x80); // Magic Byte
		buf.append((byte)0x09); // GETQ Opcode
		buf.append(new byte[] {0x00, 0x03}); // 3 byte keylength (KEY)
		buf.append((byte)0x00); // Extra Length
		buf.append((byte)0x00); // data type
		buf.append(new byte[] {0x00, 0x00}); // reserved
		buf.append(new byte[] {0x00, 0x00, 0x00, 0x03}); // total body size
		buf.append(new byte[] {0x00, 0x00, 0x00, 0x00}); // Opaque
		buf.append(new byte[] {0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00}); // CAS
		buf.append(new byte[] {0x65, 0x6F, (byte)(i % 9)});
		buffers[i] = buf;
	}
}

@Test
public void giveItSomeLoad() {
	MessageDispatcher dispatcher = new MessageDispatcher();

	long amountOfOps = 100000000;
	for (long i = 0; i < amountOfOps; i++) {
		dispatcher.write(new Buffer(buffers[(int)(i % 9)]).flip());
	}
}
```

To actually simulate something real, this test creates ten buffer instances with Couchbase messages. Those familiar with the memcached binary protocol can identify it as a GETQ request. This means it does not return anything when the key is not found (which is what we want, because in this case we want to benchmark the upper limit for write throughput and not concern us with parsing of error responses). Once the data is created, we run a given amount of operations and call the `write` method on the `MessageDispatcher`.

# Batch all the things!
If we run this, we get a - very disappointing - number of only 50K ops/s. In addition, we get lots of CPU usage on the Java process (100% and more on a quad core processor here). Why is it so slow? The answer is: network overhead. In our case, we send out 27 byte chunks over the network. With all the TCP, IP and Ethernet headers, there is lots of unnecessary overhead involved that puts down our performance. The answer to that is batching! If our producers are faster than the consumers, we can batch the intermediate data up into a buffer and send it over the wire in one chunk. This will give us a much better goodput ratio. 

To help us with batching the Disruptor and the Processor expose the start end end of a batching phase to our consumer. To get this information, we need to extend from the `BatchConsumer` instead of the regular `Consumer`. Let's refactor our node and add some batching characteristics:

```java
public class BatchingNode implements BatchConsumer<Event<Buffer>> {

  private final TcpClient<Buffer, Buffer> client;
  private TcpConnection<Buffer, Buffer> conn = null;
  private Buffer writeBuffer;

  public BatchingNode(String hostname, Environment env) {
    client = new TcpClientSpec<Buffer, Buffer>(NettyTcpClient.class)
      .env(env)
      .connect(hostname, 11210)
      .get();

    try {
      conn = client.open().await();
    } catch (InterruptedException e) {
      e.printStackTrace();
    }

    writeBuffer = new Buffer(1500, true);

  }

  @Override
  public void start() {
  }

  @Override
  public void end() {
    flush();
  }

  @Override
  public void accept(final Event<Buffer> bufferEvent) {
    Buffer buf = bufferEvent.getData();

    if (writeBuffer.remaining() <= buf.remaining()) {
      flush();
    }

    writeBuffer.append(buf);
  }

  private void flush() {
    final CountDownLatch latch = new CountDownLatch(1);
    writeBuffer.flip();
    conn.send(writeBuffer, new Consumer<Boolean>() {
      @Override
      public void accept(final Boolean success) {
        latch.countDown();
      }
    });
    try {
      latch.await(1, TimeUnit.SECONDS);
    } catch (final Exception ex) {
      throw new RuntimeException("got ex", ex);
    }
    writeBuffer.clear();
  }

}
```

The code here is not much different. Note that we refactored the writing part out into the `flush` method. In the `accept` method, we write to the network if the buffer is full, otherwise we just add it to our write buffer. Note that we also need to `flush` if the batching is over (notified through the `end` method), otherwise we would potentially keep data around for a longer time than needed (and latency is still important to us).

Let's run the test case again... now we get 500k ops/s with only 40% CPU load on the java process! Now that's what I call an improvement!

# Summary
This was a very quick introduction into the Processor, just one piece in the very promising Reactor framework. There is so much more to blog about in the future, so stay tuned!