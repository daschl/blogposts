+++
date = "2013-06-18"
draft = false
title = "Fun with Couchbase Views and MessagePack"
description = "This post shows you how you can store and query documents encoded with MessagePack!"
tags = ["couchbase","messagepack","encoding"]
slug = "Fun-with-Couchbase-Views-and-Message-Pack"
+++

Alright, before we start I have to admit that this is a little bit of a hack. Not that it doesn't work, but of course Couchbase Server 2.0 officially only supports JSON documents to be queried through Views. In addition to that, it is not so much known that you have access to all the other documents through Base64 encoding. Recently, [Sergey](http://avsej.net/2013/analyzing-binary-data-in-couchbase/) showed us how to very easily analyze binary data in Couchbase Views and this brought me on the idea to take it one step further.

In this blog post we're going to store [MessagePack](http://msgpack.org/) formatted data in Couchbase and then make its content available to Views (and allow us to query it). Note that you don't need to patch Couchbase for this, its just a snippet of JavaScript code that you need to include in your map function. We'll be using Java on the client side here, but nearly every combination of our official SDKs and MessagePack modules works for this.

## Storing the Data
In order to store something meaningful we can query later, let's create a Maven project with all needed dependencies. Note that I'm assuming you have a Couchbase Server installation running and are familiar with the Couchbase Java SDK (at least a little bit). Also I won't cover the View fundamentals since this is a little advanced topic.

Here are the required dependencies to start with the fun:

    <dependencies>
        <dependency>
            <groupId>couchbase</groupId>
            <artifactId>couchbase-client</artifactId>
            <version>1.1.7</version>
        </dependency>
        <dependency>
            <groupId>org.msgpack</groupId>
            <artifactId>msgpack</artifactId>
            <version>0.6.7</version>
        </dependency>
    </dependencies>

Also, don't forget that the Couchbase SDK lives in its own repository:

	<repository>
		<id>couchbase</id>
		<name>Couchbase Maven Repository</name>
		<url>http://files.couchbase.com/maven2/</url>
		<snapshots>
			<enabled>false</enabled>
		</snapshots>
	</repository>

Now, we can create a simple script that connects to Couchbase, creates some HashMaps with meaningful content, encodes them with the MessagePack encoder and stores them in the database:

	import com.couchbase.client.CouchbaseClient;
	import org.msgpack.MessagePack;

	import java.net.URI;
	import java.util.Arrays;
	import java.util.HashMap;

	public class Main {

	  public static void main(String[] args) throws Exception {

	    // Connect to Couchbase
	    CouchbaseClient cb = new CouchbaseClient(Arrays.asList(new URI("http://127.0.0.1:8091/pools")), "default", "");

	    // Init MessagePack
	    MessagePack msgpack = new MessagePack();

	    // Create a few Documents with some content
	    HashMap<String, String> user1 = new HashMap<String, String>();
	    user1.put("firstname", "Michael");
	    user1.put("lastname", "Nitschinger");

	    HashMap<String, String> user2 = new HashMap<String, String>();
	    user2.put("firstname", "Matt");
	    user2.put("lastname", "Ingenthron");

	    HashMap<String, String> user3 = new HashMap<String, String>();
	    user3.put("firstname", "Sergey");
	    user3.put("lastname", "Avseyev");

	    // Encode and store them
	    cb.set("user:michael", msgpack.write(user1)).get();
	    cb.set("user:matt", msgpack.write(user2)).get();
	    cb.set("user:sergey", msgpack.write(user3)).get();

	    // Close the Connection
	    cb.shutdown();
	  }

	}

The only new part to Couchbase folks is the "MessagePack" code. You create a new instance of `MessagePack` and then pass the data to the `write` method. I'm not an expert on this library, but it looks like you can also make it work with generic POJOs, which would be something to look into if you model an actual application with it.

## Querying the Data
Now if we run the code, we'll see that three documents have been persisted, but they show up as binary. That's because its not JSON and kind of expected. Now, go create a View on your bucket and start out with an empty map function like this:

	function(doc, meta) {
	  emit(doc, null);
	}

If you query it, you'll see the document keys emitted like `gqhsYXN0bmFtZadBdnNleWV2qWZpcnN0bmFtZaZTZXJnZXk`, which is [Base64](http://en.wikipedia.org/wiki/Base64) encoding. Now, let's change the view function a bit and use the built-in `decodeBase64` method:

	function(doc, meta) {
	  emit(decodeBase64(doc), null);
	}

Now your output looks more like `[130,168,108,97,115,116,110,97,109,...121]`, which is the native format of MessagePack! You would also see the same output if you take the result of `msgpack.write()` and print out the byte array properly.

The first step is done, we have direct access to the stored data. Now we need to decode it. The good news is that there is a JavaScript library for MessagePack, but it doesn't work out of the box with our view code. The library does lots of browser stuff which we don't have in the environment. [here](https://raw.github.com/msgpack/msgpack-javascript/master/msgpack.js) is the link to the original library.

Now, after some fiddling with the code and removing unneeded parts (but actually not changing how it works), I ended up with something that works. You can go ahead and grab the full snippet [here](https://gist.github.com/daschl/5796263). Personally, I prefer to have my map functions short and sweet, so here is the same code, but [minified](https://gist.github.com/daschl/5796268).

Now, we can plug this code into our map function and use the method to decode our data on the fly:

	function(doc, meta) {
	    /*!{id:msgpack.js,ver:1.05,license:"MIT",author:"uupaa.js@gmail.com"}*/
	    /* Modified by @daschl and @avsej to strip out whats not needed */
	    var msgunpack=function(){function k(){var g,d,e,b=0,f,h,c=m;d=c[++a];if(224<=d)return d-256;if(192>d){if(128>d)return d;144>d?(b=d-128,d=128):160>d?(b=d-144,d=144):(b=d-160,d=160)}switch(d){case 192:return null;case 194:return!1;case 195:return!0;case 202:return b=16777216*c[++a]+(c[++a]<<16)+(c[++a]<<8)+c[++a],f=b>>23&255,h=b&8388607,!b||2147483648===b?0:255===f?h?NaN:Infinity:(b&2147483648?-1:1)*(h|8388608)*Math.pow(2,f-127-23);case 203:b=16777216*c[++a]+(c[++a]<<16)+(c[++a]<<8)+c[++a];d=b&2147483648;
	    f=b>>20&2047;h=b&1048575;if(!b||2147483648===b)return a+=4,0;if(2047===f)return a+=4,h?NaN:Infinity;b=16777216*c[++a]+(c[++a]<<16)+(c[++a]<<8)+c[++a];return(d?-1:1)*((h|1048576)*Math.pow(2,f-1023-20)+b*Math.pow(2,f-1023-52));case 207:return b=16777216*c[++a]+(c[++a]<<16)+(c[++a]<<8)+c[++a],4294967296*b+16777216*c[++a]+(c[++a]<<16)+(c[++a]<<8)+c[++a];case 206:b+=16777216*c[++a]+(c[++a]<<16);case 205:b+=c[++a]<<8;case 204:return b+c[++a];case 211:return b=c[++a],b&128?-1*(72057594037927936*(b^255)+
	    281474976710656*(c[++a]^255)+1099511627776*(c[++a]^255)+4294967296*(c[++a]^255)+16777216*(c[++a]^255)+65536*(c[++a]^255)+256*(c[++a]^255)+(c[++a]^255)+1):72057594037927936*b+281474976710656*c[++a]+1099511627776*c[++a]+4294967296*c[++a]+16777216*c[++a]+65536*c[++a]+256*c[++a]+c[++a];case 210:return b=16777216*c[++a]+(c[++a]<<16)+(c[++a]<<8)+c[++a],2147483648>b?b:b-4294967296;case 209:return b=(c[++a]<<8)+c[++a],32768>b?b:b-65536;case 208:return b=c[++a],128>b?b:b-256;case 219:b+=16777216*c[++a]+(c[++a]<<
	    16);case 218:b+=(c[++a]<<8)+c[++a];case 160:f=[];d=a;for(g=d+b;d<g;)e=c[++d],f.push(128>e?e:224>e?(e&31)<<6|c[++d]&63:(e&15)<<12|(c[++d]&63)<<6|c[++d]&63);a=d;return 10240>f.length?l.apply(null,f):n(f);case 223:b+=16777216*c[++a]+(c[++a]<<16);case 222:b+=(c[++a]<<8)+c[++a];case 128:for(h={};b--;){g=c[++a]-160;f=[];d=a;for(g=d+g;d<g;)e=c[++d],f.push(128>e?e:224>e?(e&31)<<6|c[++d]&63:(e&15)<<12|(c[++d]&63)<<6|c[++d]&63);a=d;h[l.apply(null,f)]=k()}return h;case 221:b+=16777216*c[++a]+(c[++a]<<16);case 220:b+=
	    (c[++a]<<8)+c[++a];case 144:for(f=[];b--;)f.push(k());return f}}function n(a){try{return l.apply(this,a)}catch(d){}for(var e=[],b=0,f=a.length,h=p;b<f;++b)e[b]=h[a[b]];return e.join("")}var j={},p={},m=[],a=0,l=String.fromCharCode;return function(g){var d;if("string"===typeof g){d=[];var e=g.split(""),b=-1,f;f=e.length;for(g=f%8;g--;)++b,d[b]=j[e[b]];for(g=f>>3;g--;)d.push(j[e[++b]],j[e[++b]],j[e[++b]],j[e[++b]],j[e[++b]],j[e[++b]],j[e[++b]],j[e[++b]])}else d=g;m=d;a=-1;return k()}}();
	  
	    var obj = msgunpack(decodeBase64(doc));
	    if(obj.firstname) {
	        emit(obj.firstname, null);
	    }
	}

This map function decodes our MessagePack data and just emits the value with key "firstname". Once you query this view, you'll see stuff like this emitted:

	{
		total_rows: 3,
		rows: [{
			id: "user:matt",
			key: "Matt",
			value: null
		},{
			id: "user:michael",
			key: "Michael",
			value: null
		},{
			id: "user:sergey",
			key: "Sergey",
			value: null
		}]
	}

Isn't that lovely? We can now query this view from our Java SDK and use the MessagePack decoding facilities to retrieve the full documents as needed:

    // Query View
    View view = cb.getView("designname", "viewname");
    ViewResponse response = cb.query(view, new Query().setIncludeDocs(true));

    // Iterate and load full documents
    for (ViewRow row : response) {
      System.out.println(msgpack.read((byte[]) row.getDocument()));
    }

On the console, you'll see the full document content:

	{"lastname":"Ingenthron","firstname":"Matt"}
	{"lastname":"Nitschinger","firstname":"Michael"}
	{"lastname":"Avseyev","firstname":"Sergey"}

Of course, feel free to use all the well-known query mechanisms for Views.

## Benefits, anyone?
Now one could argue that this is nice to play around with, but what do I actually gain from it? Arguably, there is more work included for the developer, and maybe for the Server side as well.

MessagePack makes bold claims that its faster and smaller than JSON, so lets try to verify this in our environment. 

To verify the size argument, let's go ahead and create 1M docs with a "reasonable" size and store them both through JSON and MessagePack. Since the overhead for each document is static inside Couchbase Server, we can very easily see how much RAM is used to hold the documents and conclude on the size difference (which is what matters at the end).

    int numdocs = 1000000;
    HashMap<String, Object> user = new HashMap<String, Object>();
    user.put("firstname", "Michael");
    user.put("lastname", "Nitschinger");
    user.put("age", 25);
    user.put("loggedIn", false);
    user.put("active", true);

    for (int i = 0; i < numdocs; i++) {
      cb.set("user:" + i, msgpack.write(user)).get();
    }

For 1M documents on my machine with Couchbase Server 2.0, the amount of RAM used is 166MB. Now, let's do the same with JSON. I'm going to use [Google GSON](https://code.google.com/p/google-gson/) to convert the HashMap:

    int numdocs = 1000000;
    HashMap<String, Object> user = new HashMap<String, Object>();
    user.put("firstname", "Michael");
    user.put("lastname", "Nitschinger");
    user.put("age", 25);
    user.put("loggedIn", false);
    user.put("active", true);

    for (int i = 0; i < numdocs; i++) {
      cb.set("user:" + i, gson.toJson(user)).get();
    }

After inserting those 1M JSON records, Couchbase Server reported exactly 198MB of RAM used! So the difference is 32MB (1/5th of the total data size), or 32 byte per document. That pays off if you have lots of records in your cluster! After doubling the content of the HashMap and storing twice as many documents (2M) MessagePack is ahead of raw JSON by around 96MB. That's another 400K documents you could store (one document is around 230 bytes compared to 285 with raw JSON).

Of course, YMMV depending on the number and size of the documents. I'm curious if you could do the same runs on your real workload and see by how much you could improve.

Now, before we talk about RAW encoding/decoding performance, we also need to look into the overhead involved on the server side. Storing and retreiving documents is not slower than any other document, because the server doesn't need to do anything special. The only overhead I can think of is that during index creation, more CPU will be consumed because you execute more JavaScript logic. I guess this CPU overhead increases with the document sizes, but I don't have any actual numbers to share here. Personally, I'd use the [sizing]() guidelines for Views and maybe add some more CPU just to be safe. It's important to note that at query time, you won't see a difference between JSON and MessagePack, because the index is already created.

Congratulations if you followed to this point! As a bonus, we'll look into the raw encoding/decoding performance of JSON and MessagePack on the JVM. Since there are lots of JSON libraries out there, we're going to use [Jackson](http://jackson.codehaus.org/) and Google GSON. Benchmarking on the JVM is not a trivial task, so I'll do my best to keep it objective and we're going to use [Google Caliper](https://code.google.com/p/caliper/) to handle things like JVM warmup. Make sure to add everything to your `pom.xml` like this:

    <dependency>
        <groupId>org.msgpack</groupId>
        <artifactId>msgpack</artifactId>
        <version>0.6.7</version>
    </dependency>
    <dependency>
        <groupId>com.google.code.gson</groupId>
        <artifactId>gson</artifactId>
        <version>2.2.4</version>
    </dependency>
    <dependency>
        <groupId>com.google.caliper</groupId>
        <artifactId>caliper</artifactId>
        <version>1.0-beta-1</version>
    </dependency>
    <dependency>
        <groupId>com.google.code.java-allocation-instrumenter</groupId>
        <artifactId>java-allocation-instrumenter</artifactId>
        <version>2.1</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.2.2</version>
    </dependency>

In this simple benchmark, we are only testing smaller and slightly larger `HashMap`s. I know that this test is not conclusive, but it should give you a starting point from where you can do your own research.

To write a Caliper benchmark, you need to extend the `Benchmark` class from the caliper package. its as simple as this:

	public class EncodingBenchmark extends Benchmark {

	  private final Gson gson = new Gson();
	  private final MessagePack msgpack = new MessagePack();
	  private final ObjectMapper mapper = new ObjectMapper();

	  private HashMap<String, Object> smallMap;
	  private HashMap<String, Object> largeMap;

	  @Override
	  protected void setUp() {
	    smallMap = new HashMap<String, Object>();
	    smallMap.put("firstname", "Foo");
	    smallMap.put("lastname", "bar");
	    smallMap.put("active", false);

	    largeMap = new HashMap<String, Object>();
	    largeMap.put("firstname", "Foo");
	    largeMap.put("lastname", "bar");
	    largeMap.put("active", false);
	    largeMap.put("firstname1", "Foo");
	    largeMap.put("lastname1", "bar");
	    largeMap.put("active1", false);
	    largeMap.put("firstname2", "Foo");
	    largeMap.put("lastname2", "bar");
	    largeMap.put("active2", false);
	    largeMap.put("firstname3", "Foo");
	    largeMap.put("lastname3", "bar");
	    largeMap.put("active3", false);
	  }

	  public void timeMessagePackSmall(final int reps) throws Exception {
	    for (int i = 0; i < reps; i++) {
	      msgpack.write(smallMap);
	    }
	  }

	  public void timeGoogleGsonSmall(int reps) throws Exception {
	    for (int i = 0; i < reps; i++) {
	      gson.toJson(smallMap);
	    }
	  }

	  public void timeJacksonSmall(int reps) throws Exception {
	    for (int i = 0; i < reps; i++) {
	      mapper.writeValueAsString(smallMap);
	    }
	  }
	  public void timeJsonSmartSmall(int reps) throws Exception {
	    for (int i = 0; i < reps; i++) {
	      mapper.writeValueAsString(smallMap);
	    }
	  }

	  public void timeMessagePackLarge(final int reps) throws Exception {
	    for (int i = 0; i < reps; i++) {
	      msgpack.write(largeMap);
	    }
	  }

	  public void timeGoogleGsonLarge(int reps) throws Exception {
	    for (int i = 0; i < reps; i++) {
	      gson.toJson(largeMap);
	    }
	  }

	  public void timeJacksonLarge(int reps) throws Exception {
	    for (int i = 0; i < reps; i++) {
	      mapper.writeValueAsString(largeMap);
	    }
	  }
	}

First, we create our converter instances (for MessagePack, GSON and Jackson) and the HashMaps. All the runs in this benchmark are executed by Caliper. We need to modify our `main` class to load the wrapper:

	import business.MsgpackBenchmark;
	import com.google.caliper.runner.CaliperMain;

	public class Main {
	  public static void main(String[] args) throws Exception {
	    CaliperMain.main(EncodingBenchmark.class, args);
	  }
	}

Now if you run this in your IDE, you'll see some logging going on:

	Experiment selection: 
	  Instruments:   [allocation, micro]
	  User parameters:   {}
	  Virtual machines:  [default]
	  Selection type:    Full cartesian product

	This selection yields 12 experiments.
	Starting experiment 1 of 12: {instrument=allocation, method=GoogleGsonLarge, vm=default, parameters={}}
	Complete!
	Starting experiment 2 of 12: {instrument=allocation, method=GoogleGsonSmall, vm=default, parameters={}}
	Complete!
	Starting experiment 3 of 12: {instrument=allocation, method=JacksonLarge, vm=default, parameters={}}
	Complete!
	Starting experiment 4 of 12: {instrument=allocation, method=JacksonSmall, vm=default, parameters={}}
	Complete!
	...
	Starting experiment 11 of 12: {instrument=micro, method=MessagePackLarge, vm=default, parameters={}}
	Complete!
	Starting experiment 12 of 12: {instrument=micro, method=MessagePackSmall, vm=default, parameters={}}
	Complete!

	Execution complete: 7.058m.
	Collected 162 measurements from:
	  2 instrument(s)
	  1 virtual machine(s)
	  6 benchmark(s)
	Results have been uploaded. View them at: https://microbenchmarks.appspot.com/runs/f25820e0-08dd-43f8-833e-e44847400f19

Once finished, Caliper even uploaded your results to a webpage (so make sure to have internet connection)! Note that if you see "GC errors" during your runs, it may be good to retry and if they still persist, consider increasing your heap size a bit. I don't know for how long the data is held on the web page, but [here](https://microbenchmarks.appspot.com/runs/f25820e0-08dd-43f8-833e-e44847400f19#r:scenario.benchmarkSpec.methodName) are my results.

At least in my benchmarks I found that libraries like Jackson outperform MessagePack by a large amount when encoding `HashMap`s. Maybe it's different with other data types and sizes though.

## Summary
I hope this blog post was a fun introduction and showed what's possible with Couchbase Server 2.0 and its new View engine. I found that MessagePack really saves you a good amount of bytes on the wire (and on the server), but is much slower on the JVM when it comes to encoding data (here `HashMap`s).
