+++
date = "2013-11-22"
draft = false
title = "Using JMH for Java Microbenchmarking"
description = "This post shows you how to write solid microbenchmarks using the new OpenJDK JMH library."
tags = ["java","jmh"]
slug = "Using-JMH-for-Java-Microbenchmarking"
disqus_identifier = "23a93829d88ddda2cac2f4d4c5ec7870a63530838c5e5c0d6cb5e74699602db0eb3c24d16f1649e984e828bcf2beb16e7782b2d7849239c53e057bd489088495"
+++

So before we dive in, let's rule two things out. First, I'm not a JVM expert and second, microbenchmarking is hard. The bigger problem is that it isn't only hard but also looks very easy if you start. You put your test code in a loop, use `System.nanoTime` or something similar to measure the total time of the run and divide it by the number of runs. Doing it that way, you could very well let your cat estimate the results (mine would do it for proper catnip).

Larger companies like Google started their own projects to remedy all kinds of pitfalls and libraries like [Caliper](https://code.google.com/p/caliper/) popped out of that. Now finally, some fine folks of the OpenJDK team thought they'll do their own and share it for everyone to use. The tool is called [jmh](http://openjdk.java.net/projects/code-tools/jmh/) or also known as "Java Microbenchmarking Harness".

In this post I just show you how to get started, but there are still lots of things you need to think about. I really recommend you to watch [the talk by Aleksey Shipilev](http://vimeo.com/78900556) which dives into concepts and pitfalls. You can also find the slides [here](http://shipilev.net/pub/talks/devoxx-Nov2013-benchmarking.pdf).

## Using JMH
There are a few ways to use it and all of them are explained on the official website linked above. In general, you want to use the self-contained jar and run them on the command line, but there is also an easy way to run them directly out of your IDE - this is what I'm showing you in a second. If you just want to run it from the command line, you do it like (but you need to build the jar first):

```
$ java -jar target/microbenchmarks.jar -h
```

Let's start a maven project in our IDE and add the following dependency (mad props to [Evgeny Mandrikov](https://twitter.com/_godin_/status/403668134206242817) for putting it on maven central recently):

```xml
<dependency>
  <groupId>org.openjdk.jmh</groupId>
  <artifactId>jmh-core</artifactId>
  <version>0.1</version>
</dependency>
```

Now let's write a main method that calls the underlying method of JMH so we can run it easily form our IDE:

```java
import org.openjdk.jmh.Main;

public class Start {
  public static void main(String... args) {
    Main.main(args);
  }
}
```

If we hit run, all we'll see is:

```
No matching benchmarks. Miss-spelled regexp? Use -v for verbose output.
```

Which is good, because we know JMH is working and looking for our benchmarks. Let's write an empty one to get a feeling of how it works.

```java
package benchmarks;

import org.openjdk.jmh.annotations.GenerateMicroBenchmark;

public class HelloWorld {

  @GenerateMicroBenchmark
  public void helloWorld() {

  }

}
```

JMH will automatically pick it up from the CLASSPATH and start to run our benchmark. You will see lots of output (forks, warmup runs), but be patient until the end and you'll see something like this:

```
# Fork: 1 of 1
# Warmup: 20 iterations, 1 s each
# Measurement: 10 iterations, 1 s each
# Threads: 1 thread, will synchronize iterations
# Benchmark mode: Throughput, ops/time
# Running: benchmarks.HelloWorld.helloWorld
# Warmup Iteration   1: 3034307.449 ops/ms
# Warmup Iteration   2: 3108187.599 ops/ms
# Warmup Iteration   3: 3117686.141 ops/ms
# Warmup Iteration   4: 3121382.926 ops/ms
# Warmup Iteration   5: 3100218.889 ops/ms
# Warmup Iteration   6: 3117158.492 ops/ms
*** snip ***
Iteration   7: 2906090.682 ops/ms
Iteration   8: 3053047.149 ops/ms
Iteration   9: 3055673.541 ops/ms
Iteration  10: 3113494.893 ops/ms

Result : 3073969.337 ±(95%) 45679.588 ±(99%) 65631.592 ops/ms
  Statistics: (min, avg, max) = (2906090.682, 3073969.337, 3120910.177), stdev = 63860.098
  Confidence intervals: 95% [3028289.749, 3119648.926], 99% [3008337.745, 3139600.930]


Benchmark                   Mode Thr     Count  Sec         Mean   Mean error    Units
b.HelloWorld.helloWorld    thrpt   1        10    1  3073969.337    65631.592   ops/ms
```

Note that both the output and the results may look different on your machine. To not fork as often and don't do so many measure runs (for demo purposes), you can apply the following flags for the command in your IDE: `-i 10 -f 1`. This will only run one fork and do 10 iterations after warming up. There are dozens of flags you can tune to your needs.

## Benchmarking real code
Now all is fun, but let's test some real code. In Spymemcached, before we put a document on the wire we check that the key is not longer than allowed and - for ascii based operations - also check if they do not contain whitespace and so on. The code in question currently looks like this:

```java
public static void validateKey(String key, boolean binary) {
  byte[] keyBytes = KeyUtil.getKeyBytes(key);
  if (keyBytes.length > MemcachedClientIF.MAX_KEY_LENGTH) {
    throw new IllegalArgumentException("Key is too long (maxlen = "
        + MemcachedClientIF.MAX_KEY_LENGTH + ")");
  }
  if (keyBytes.length == 0) {
    throw new IllegalArgumentException(
        "Key must contain at least one character.");
  }
  if(!binary) {
    // Validate the key
    for (byte b : keyBytes) {
      if (b == ' ' || b == '\\n' || b == '\\r' || b == 0) {
        throw new IllegalArgumentException(
            "Key contains invalid characters:  ``" + key + "''");
      }
    }
  }
}
```

Let's say we don't care about optimizing it yet, but we want to see how it performs if the "binary" flag is applied or not. Obviously less work is done but we want to see how that turns out in practice. Let's add the dependency so we can use the static method directly from the code:

```xml
<dependency>
  <groupId>net.spy</groupId>
  <artifactId>spymemcached</artifactId>
  <version>2.10.2</version>
</dependency>
```

And write a new benchmark that compares both approaches with variation.

```java
package benchmarks;

import net.spy.memcached.util.StringUtils;
import org.openjdk.jmh.annotations.GenerateMicroBenchmark;

public class KeyBench {

  /**
   * 12 chars long
   */
  private static final String SHORT_KEY = "user:michael";

  /**
   * 240 chars long
   */
  private static final String LONG_KEY = "thisIsAFunkyKeyWith_underscores_AndAlso334" +
      "3252545345NumberslthisIsAFunkyKeyWith_underscores_AndAlso3343252545345Numbe" +
      "rslthisIsAFunkyKeyWith_underscores_AndAlso3343252545345NumberslthisIsAFunkyK" +
      "eyWith_underscores_AndAlso3343252545345Numbersl";

  @GenerateMicroBenchmark
  public void validateShortKeyBinary() {
    StringUtils.validateKey(SHORT_KEY, true);
  }

  @GenerateMicroBenchmark
  public void validateShortKeyAscii() {
    StringUtils.validateKey(SHORT_KEY, false);
  }

  @GenerateMicroBenchmark
  public void validateLongKeyBinary() {
    StringUtils.validateKey(LONG_KEY, true);
  }

  @GenerateMicroBenchmark
  public void validateLongKeyAscii() {
    StringUtils.validateKey(LONG_KEY, false);
  }
}
```

We also see how the performance differs for long and short keys. The first run we do is with 10 iterations and 1 fork:

  Result : 18093.561 ±(95%) 65.824 ±(99%) 94.575 ops/ms
    Statistics: (min, avg, max) = (17920.967, 18093.561, 18240.197), stdev = 92.022
    Confidence intervals: 95% [18027.737, 18159.385], 99% [17998.986, 18188.136]


  Benchmark                             Mode Thr     Count  Sec         Mean   Mean error    Units
  b.KeyBench.validateLongKeyAscii      thrpt   1        10    1     1718.532        8.504   ops/ms
  b.KeyBench.validateLongKeyBinary     thrpt   1        10    1     3632.664       24.579   ops/ms
  b.KeyBench.validateShortKeyAscii     thrpt   1        10    1    13198.597       71.557   ops/ms
  b.KeyBench.validateShortKeyBinary    thrpt   1        10    1    18093.561       94.575   ops/ms

The first observation we can make is that - without surprise - shorter keys are faster than longer keys. But we can also see that the results are much more consistent for longer keys than shorter ones. So let's grab a coffee (this test took over 10 minutes to complete) and run the results with 20 iterations and 10 forks:

  Result : 17796.673 ±(95%) 56.110 ±(99%) 76.699 ops/ms
    Statistics: (min, avg, max) = (17441.465, 17796.673, 17947.702), stdev = 119.891
    Confidence intervals: 95% [17740.563, 17852.783], 99% [17719.974, 17873.371]


  Benchmark                             Mode Thr     Count  Sec         Mean   Mean error    Units
  b.KeyBench.validateLongKeyAscii      thrpt   1       200    1     1719.010        2.160   ops/ms
  b.KeyBench.validateLongKeyBinary     thrpt   1       200    1     3745.221       12.101   ops/ms
  b.KeyBench.validateShortKeyAscii     thrpt   1       200    1    13144.629       16.918   ops/ms
  b.KeyBench.validateShortKeyBinary    thrpt   1       200    1    17886.628       31.513   ops/ms

We get similar results, but our mean error rate went down which gives us further confidence in our numbers.

Now here comes the important part: while microbenchmarking definitely helps to find bottlenecks and shows you pointers where to go fix your code, it (I think more importantly) shows you code that you do not need to optimize. Say if the numbers shown above fit our application purpose we can "check them off" the list and go look somewhere else to optimize.

Also, if you are running those benchmarks on your notebook (and not on the servers, where you should too), make sure you don't enable power management so your CPUs work full steam during the full runs. Also, close all other programs that are not essential and leave it alone until the benchmark finishes. That way you reduce the probability of other tasks inferring with your benchmark results and helping you to get more consistent ones.

## Further Considerations
Funny enough, JMH not only makes your microbenchmarks more accurate, I think they are also much easier to write. You don't need to fight with things happening in the JVM and your operating system while the tests run. You can integrate those benchmarks into your CI runs when someone checks in a new piece of code and look at historically accurate performance numbers (for crucial pieces in your codebase).

One thing I need to tell you before finishing this post. Please go watch the video shown above, since there is still a lot to consider, even with JMH. Also a very good pointer is the examples section located [here](http://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/). You should especially be on the lookout for dead code elimination, and the proper use of black holes in JMH.

Please share your findings and additions in the comments section below!

## Additional Information
As [Aleksey Shipilёv](https://twitter.com/shipilev) points out in the comments, there is also a nice Java API that you can use, instead of just proxying "main" and emulating the command line. Here is an example that shows how to use it. Note that iterating the MAP is optional, but it gives you a code-readable way of the results. If you just call `new Runner(opts).run();`, you will see the identical output as above.

```java
public static void main(String... args) throws Exception {
  Options opts = new OptionsBuilder()
      .include(".*")
      .warmupIterations(10)
      .measurementIterations(10)
      .jvmArgs("-server")
      .forks(1)
      .outputFormat(OutputFormatType.TextReport)
      .build();

  Map<BenchmarkRecord,RunResult> records = new Runner(opts).run();
  for (Map.Entry<BenchmarkRecord, RunResult> result : records.entrySet()) {
    Result r = result.getValue().getPrimaryResult();
    System.out.println("API replied benchmark score: "
      + r.getScore() + " "
      + r.getScoreUnit() + " over "
      + r.getStatistics().getN() + " iterations");
  }
}
```

Also, the constant keys used in the above example are subject to constant optimizations. To work around that, there is the @State annotation for the class and the static fields need to be instance variables. Here is the updated code sample for clarity, the numbers don't change for the purpose of this post.

```java
package benchmarks;

import net.spy.memcached.util.StringUtils;
import org.openjdk.jmh.annotations.GenerateMicroBenchmark;
import org.openjdk.jmh.annotations.State;

@State
public class KeyBench {

  /**
   * 12 chars long
   */
  private String SHORT_KEY = "user:michael";

  /**
   * 240 chars long
   */
  private String LONG_KEY = "thisIsAFunkyKeyWith_underscores_AndAlso334" +
      "3252545345NumberslthisIsAFunkyKeyWith_underscores_AndAlso3343252545345Numbe" +
      "rslthisIsAFunkyKeyWith_underscores_AndAlso3343252545345NumberslthisIsAFunkyK" +
      "eyWith_underscores_AndAlso3343252545345Numbersl";

  @GenerateMicroBenchmark
  public void validateShortKeyBinary() {
    StringUtils.validateKey(SHORT_KEY, true);
  }

  @GenerateMicroBenchmark
  public void validateShortKeyAscii() {
    StringUtils.validateKey(SHORT_KEY, false);
  }

  @GenerateMicroBenchmark
  public void validateLongKeyBinary() {
    StringUtils.validateKey(LONG_KEY, true);
  }

  @GenerateMicroBenchmark
  public void validateLongKeyAscii() {
    StringUtils.validateKey(LONG_KEY, false);
  }
}
```

Thanks for pointing it out Aleksey!
