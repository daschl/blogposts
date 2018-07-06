+++
date = "2014-01-29"
draft = false
title = "Bootstrapping from DNS SRV records in Java"
description = "See how to load hostnames in your Java application from DNS SRV records."
tags = ["java","couchbase","dns"]
slug = "Bootstrapping-from-DNS-SRV-records-in-Java"
+++

I know this topic has a very narrow audience, but I hope that one or two people out there scratching their heads will benefit from it.

Here's the itch we're trying to scratch: is there an easy way to determine hostnames for - let's say - a database connection? There are many ways to do this, like hardcoding it, providing them through a properties file and so on. All this techniques (maybe aside from fetching it over the network from a central storage) require some modifications on the server once one of the hostnames changes. If you need to maintain a lot of machines, this can get inefficient pretty quickly.

Now let's step back a second and think about what we're using anyway in our infrastructure: DNS. Until recently I haven't heard of the SRV record that you can use to "refer" to other hostnames. The nice thing about this is that you can provide more endpoints and even add weights and and priorities. Let's look at an example:

```
_cbnodes._tcp.example.com.  0  IN  SRV  20  0  8091  node2.example.com.
_cbnodes._tcp.example.com.  0  IN  SRV  10  0  8091  node1.example.com.
_cbnodes._tcp.example.com.  0  IN  SRV  30  0  8091  node3.example.com.
```

Here, based on the "_cbnodes._tcp.example.com." service information we get to know that there are three nodes configured that belong to this service. They all listen on port `8091`, but have priorities associated with them (10, 20, 30). Lower priorities are considered more important, so you can use that to your advantage. The `0` between the priority and the port is the weight. You can use different weights (which are probabilities) to generate some kind of load-balancing behaviour.

Once this is configured on the DNS server, we can make use of that in our application. Say we want to infer the seed nodes to bootstrap from in our `CouchbaseClient`. To make this happen, we need to make use of [JNDI](http://en.wikipedia.org/wiki/Java_Naming_and_Directory_Interface). Let's first create a simple class that will hold those records shown above:

```java
static class DnsRecord implements Comparable<DnsRecord> {
    private final int priority;
    private final int weight;
    private final int port;
    private final String host;

    public DnsRecord(int priority, int weight, int port, String host) {
      this.priority = priority;
      this.weight = weight;
      this.port = port;
      this.host = host.replaceAll("\\\\.$", "");
    }

    public int getPriority() {
      return priority;
    }

    public int getWeight() {
      return weight;
    }

    public int getPort() {
      return port;
    }

    public String getHost() {
      return host;
    }

    public static DnsRecord fromString(String input) {
      String[] splitted = input.split(" ");
      return new DnsRecord(
        Integer.parseInt(splitted[0]),
        Integer.parseInt(splitted[1]),
        Integer.parseInt(splitted[2]),
        splitted[3]
      );
    }

    @Override
    public String toString() {
      return "DnsRecord{" +
        "priority=" + priority +
        ", weight=" + weight +
        ", port=" + port +
        ", host='" + host + '\\'' +
        '}';
    }

    @Override
    public int compareTo(DnsRecord o) {
      if (getPriority() < o.getPriority()) {
        return -1;
      } else {
        return 1;
      }
    }

}
```

We provide a custom `compareTo` method in order to automatically sort each `DnsRecord` by priority. The next step is to write a method that allows us to fetch the actual information:

```java
String service = "_cbnodes._tcp.example.com";

Hashtable<String, String> env = new Hashtable<String, String>();
env.put("java.naming.factory.initial", "com.sun.jndi.dns.DnsContextFactory");
  env.put("java.naming.provider.url", "dns:");
DirContext ctx = new InitialDirContext(env);
Attributes attrs = ctx.getAttributes(service, new String[] { "SRV" });

  NamingEnumeration<?> servers = attrs.get("srv").getAll();
  Set<DnsRecord> sortedRecords = new TreeSet<DnsRecord>();
  while (servers.hasMore()) {
    DnsRecord record = DnsRecord.fromString((String) servers.next());
    sortedRecords.add(record);
  }
```

Now we have a `Set` of sorted `DnsRecords`, which we can use however we want to. For example, with [Couchbase](http://couchbase.com) we can turn them into `URI`s:

```java
List<URI> uris = new ArrayList<URI>();
for(DnsRecord record : sortedRecords) {
	uris.add(new URI("http://" + record.getHost() + ":" + record.getPort() + "/pools"));
}
```

If you want to play around with the code and don't have DNS SRV records set up, I recommend you to use `_xmpp-server._tcp.gmail.com`. It exposes Googles GMail XMPP servers and lets
you try the code without much effort. In case you wonder how to mock that thing correctly, I recommend you to mock the `DirContext` like so:

```java
String service = "_seeds._tcp.couchbase.com";

BasicAttributes basicAttributes = new BasicAttributes(true);
BasicAttribute basicAttribute = new BasicAttribute("SRV");
basicAttribute.add("20 0 8091 node2.couchbase.com.");
basicAttribute.add("10 0 8091 node1.couchbase.com.");
basicAttribute.add("30 0 8091 node3.couchbase.com.");
basicAttribute.add("40 0 8091 node4.couchbase.com.");
basicAttributes.put(basicAttribute);

DirContext mockedContext = mock(DirContext.class);
when(mockedContext.getAttributes(service, new String[] { "SRV" }))
  .thenReturn(basicAttributes);
```

The code shown here (and more advanced features) are part of the official Couchbase SDK, so check out the [codebase](https://github.com/couchbase/couchbase-java-client) if you want to learn more!
