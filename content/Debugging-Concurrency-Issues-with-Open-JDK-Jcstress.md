+++
date = "2014-05-27"
draft = false
title = "Debugging Concurrency Issues with OpenJDK Jcstress"
description = "Learn about how to use the JCStress tool from the OpenJDK team to debug concurrency issues."
tags = ["java","jcstress","openjdk"]
slug = "Debugging-Concurrency-Issues-with-Open-JDK-Jcstress"
disqus_identifier = "a3564b8d59a2331a4c30acf6f0179b8f85001b7edca74d867e07ef6a77aa09d3654416020a9327c1ad0acc8e0e32de1d1f0e768da8a660024fb44cc0ca974a9d"
+++

I fell in love with the Java Microbenchmarking Harness ([JMH](http://openjdk.java.net/projects/code-tools/jmh/)) a few months ago since (in my opinion) it is the only sane way to do microbenchmarks of JVM code right now. I also poked around on their website for other tools they provide, and found that there is another very interesting tool called [jcstress](http://openjdk.java.net/projects/code-tools/jcstress/). It stands for Java Concurrency Stress tests and is used mainly by the OpenJDK people itself to make sure their code works correctly with regards to concurrency.

As always, you need a motivating factor so today [a ticket](https://www.couchbase.com/issues/browse/SPY-170) got opened which reported a race condition inside a utility class in the [spymemcached](https://code.google.com/p/spymemcached/) library (which is also used by the Couchbase Java SDK). Before coming up with complicated code that simulates multi-threaded consumers of the API I thought I could just try out `jcstress` first.

TL;DR: I managed to find, fix and verify the concurrency issue. If you want to learn how, read along.

A short disclaimer: as you can guess, I'm far away from an expert on the `jcstress` library and there could be information provided here that is plain wrong. The intent of this post is to show you how to set up `jcstress` and run your code against it. Also, I'd like to thank [Aleksey Shipilёv](https://twitter.com/shipilev) to help me verify/improve the test and look over it. He also told me that `jcstress` is not (yet) ready for a broad public consumption, so your mileage may vary.

# Setup
As [described](http://openjdk.java.net/projects/code-tools/jcstress/) on the website, you need to clone the repository and build the whole thing. Also, make sure to have Java 8 installed for compiling it (I think running with older versions afterwards works).

```
$ hg clone http://hg.openjdk.java.net/code-tools/jcstress/ jcstress
$ cd jcstress/
```

Now, go import the maven project in your favorite IDE ([IntelliJ, what else?](http://www.jetbrains.com/idea/)). We need to add our own code and tests so that it can be picked up later. You can either add your code under test as a dependency into the `pom.xml` file or just copy it over to the project if it is self contained (so it's easier to play and poke around). If you want to follow along, add the `StringUtils.java` and the `StringUtilsTest.java` files to the `com.couchbase.client.tests.util` package:

```java
package com.couchbase.client.tests.util;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * Utility methods on string objects.
 */
public final class StringUtils {

    /**
     * A pattern to match on a signed integer value.
     */
    private static final Pattern decimalPattern = Pattern.compile("^-?\\d+$");

    /**
     * The matcher for the decimal pattern regex.
     */
    private static final Matcher decimalMatcher = decimalPattern.matcher("");

    /**
     * Maximum supported key length.
     */
    private static final int MAX_KEY_LENGTH = 250;

    /**
     * Exception thrown if the input key is too long.
     */
    private static final IllegalArgumentException KEY_TOO_LONG_EXCEPTION =
        new IllegalArgumentException("Key is too long (maxlen = "
            + MAX_KEY_LENGTH + ")");

    /**
     * Exception thrown if the input key is empty.
     */
    private static final IllegalArgumentException KEY_EMPTY_EXCEPTION =
        new IllegalArgumentException("Key must contain at least one character.");

    /**
     * Preset the stack traces for the static exceptions.
     */
    static {
        KEY_TOO_LONG_EXCEPTION.setStackTrace(new StackTraceElement[0]);
        KEY_EMPTY_EXCEPTION.setStackTrace(new StackTraceElement[0]);
    }

    /**
     * Private constructor, since this is a purely static class.
     */
    private StringUtils() {
        throw new UnsupportedOperationException();
    }

    /**
     * Check if a given string is a JSON object.
     *
     * @param s the input string.
     * @return true if it is a JSON object, false otherwise.
     */
    public static boolean isJsonObject(final String s) {
        if (s == null || s.isEmpty()) {
            return false;
        }

        if (s.startsWith("{") || s.startsWith("[")
            || "true".equals(s) || "false".equals(s)
            || "null".equals(s) || decimalMatcher.reset(s).matches()) {
            return true;
        }

        return false;
    }

}
```

And the test:

```java
package com.couchbase.client.tests.util;

import org.openjdk.jcstress.annotations.Actor;
import org.openjdk.jcstress.annotations.JCStressTest;
import org.openjdk.jcstress.annotations.State;
import org.openjdk.jcstress.infra.results.IntResult2;

@JCStressTest
@State
public class StringUtilsTest {

    @Actor
    public void actor1(IntResult2 result) {
        try {
            result.r1 = StringUtils.isJsonObject("1234") ? 1 : 0;
        } catch(Exception ex) {
            result.r1 = -1;
        }
    }

    @Actor
    public void actor2(IntResult2 result) {
        try {
            result.r2 = StringUtils.isJsonObject("5678") ? 1 : 0;
        } catch(Exception ex) {
            result.r2 = -1;
        }
    }
}
```

The test uses `@Actor` annotations to define the different actors of the system. You can see that both actors access the same static method and put the result (which is abbreviated to a simple true (1) / false (0) pattern) in the `IntResult2` object. Both actors share their results and if an exception
is raised we store -1. Now what is that good for?

We also need to add a description file for our test which sets the expecations on allowed and disallowed value combinations. Note that I tried to add my own .xml file, but somehow it didn't get picked up, so I modified an existing xml (to be specific the `resources/org/openjdk/jcstress/desc/atomic-boolean.xml`) and added the following:

```xml
<test name="com.couchbase.client.tests.util.StringUtilsTest">
    <contributed-by>Michael Nitschinger</contributed-by>
    <description>
        Tests the thread-safeness of the StringUtil class.
    </description>
    <case>
        <match>[1, 1]</match>
        <expect>ACCEPTABLE</expect>
        <description>
            Acceptable to see true.
        </description>
    </case>
    <unmatched>
        <expect>FORBIDDEN</expect>
        <description>
            Other cases are not expected. -1 would mean an exception raised.
        </description>
    </unmatched>
</test>
```

You can see that the only thing which is acceptable is that both actors always get `true` returned, all other combinations are simply forbidden (that is, marked as failure).

Now that we've got that set up, we need to compile the thing on the command line through maven:

```
$ mvn clean install -pl tests-custom -am
```

Finally, we run the shaded jar and give it a regex so that only our own test gets picked up:

```
$ java -jar tests-custom/target/jcstress.jar -t ".*StringUtils.*"
```

The resulting output is as follows:

```
Java Concurrency Stress Tests
---------------------------------------------------------------------------------
Rev: a119fb6622e3+, built by michael with 1.8.0_05 at 20140527-1040

Burning up to figure out the exact CPU count....... done!

Non-fatal: VM support for online deoptimization is not enabled, tests might miss some issues.
Possible reasons are:
  1) unsupported JDK, only JDK 8+ is supported;
  2) -XX:+UnlockDiagnosticVMOptions -XX:+WhiteBoxAPI are missing;
  3) the jcstress JAR is not added to -Xbootclasspath/a

Non-fatal: VM support for @Contended is not enabled, tests might run slower.
Possible reasons are:
  1) unsupported JDK, only JDK 8+ is supported;
  2) -XX:-RestrictContended is missing, or the jcstress JAR is not added to -Xbootclasspath/a

FORKED MODE
  Test preset mode: "default"
  Writing the test results to "jcstress.1401180107287"
  Parsing results to "results/"
  Running each test matching ".*StringUtils.*" for 1 forks, 5 iterations, 1000 ms each
  Solo stride size will be autobalanced within [10, 10000] elements
  Hardware threads in use/available: 8/8, no yielding in use.


 (ETA:        n/a) (R: 5.90E+08) (T:   1/1) (F: 1/1) (I: 1/5)   [FAILED] com.couchbase.client.tests.util.StringUtilsTest
                     Observed state     Occurrences        Expectation Interpretation      
                             [1, 1] (    1,889,690)         ACCEPTABLE Acceptable to see true.                                     
                             [0, 1] (          150)          FORBIDDEN Other cases are not expected. -1 would mean an exception ...
                             [1, 0] (          130)          FORBIDDEN Other cases are not expected. -1 would mean an exception ...

 (ETA:   00:00:02) (R: 4.03E+06) (T:   1/1) (F: 1/1) (I: 2/5)       [OK] com.couchbase.client.tests.util.StringUtilsTest
 (ETA:   00:00:01) (R: 3.05E+06) (T:   1/1) (F: 1/1) (I: 3/5)       [OK] com.couchbase.client.tests.util.StringUtilsTest
 (ETA:   00:00:00) (R: 2.74E+06) (T:   1/1) (F: 1/1) (I: 4/5)       [OK] com.couchbase.client.tests.util.StringUtilsTest
 (ETA:        now) (R: 2.60E+06) (T:   1/1) (F: 1/1) (I: 5/5)       [OK] com.couchbase.client.tests.util.StringUtilsTest
Reading the results back...
Generating the report...
Look at results/index.html for the complete run results.

Will throw any pending exceptions at this point.
Exception in thread "main" java.lang.AssertionError: TEST FAILURES:
com.couchbase.client.tests.util.StringUtilsTest: Observed forbidden state: [0, 1]
com.couchbase.client.tests.util.StringUtilsTest: Observed forbidden state: [1, 0]

	at org.openjdk.jcstress.infra.grading.ExceptionReportPrinter.parse(ExceptionReportPrinter.java:119)
	at org.openjdk.jcstress.JCStress.run(JCStress.java:134)
	at org.openjdk.jcstress.Main.main(Main.java:85)
```

The command line output is very descriptive, but you can also open the `results/index.html` file for a nicer visualization. We can see that most of the time the output was acceptable, but 280 times we got non-consistent output. In those cases, one of the actors got `false` as a response instead of `true` pretty bad if you ask me!

You can look through the code if you want to find the race condition, but for those who are too lazy here is the code fix:

```
   /**
    * Exception thrown if the input key is empty.
@@ -112,7 +106,7 @@ public final class StringUtils {

     if (s.startsWith("{") || s.startsWith("[")
       || "true".equals(s) || "false".equals(s)
-      || "null".equals(s) || decimalMatcher.reset(s).matches()) {
+      || "null".equals(s) || decimalPattern.matcher(s).matches()) {
       return true;
     }
```

As it turns out, the `matcher` is not thread safe and there is a race condition between resetting the characters and matching them afterwards. If we fix it to always create a new `matcher`, the race condition should go away. So let's make the proposed changes and run the test again (don't forget to run `mvn clean install -pl tests-custom -am` again since we changed the code):

```
*snip*
 (ETA:        n/a) (R: 1.82E+09) (T:   1/1) (F: 1/1) (I: 1/5)       [OK] com.couchbase.client.tests.util.StringUtilsTest
 (ETA:   00:00:02) (R: 1.13E+07) (T:   1/1) (F: 1/1) (I: 2/5)       [OK] com.couchbase.client.tests.util.StringUtilsTest
 (ETA:   00:00:01) (R: 8.90E+06) (T:   1/1) (F: 1/1) (I: 3/5)       [OK] com.couchbase.client.tests.util.StringUtilsTest
 (ETA:   00:00:00) (R: 7.98E+06) (T:   1/1) (F: 1/1) (I: 4/5)       [OK] com.couchbase.client.tests.util.StringUtilsTest
 (ETA:        now) (R: 7.61E+06) (T:   1/1) (F: 1/1) (I: 5/5)       [OK] com.couchbase.client.tests.util.StringUtilsTest
Reading the results back...
Generating the report...
Look at results/index.html for the complete run results.

Will throw any pending exceptions at this point.
Done.
```

Look at that! Our results are now consistent. We can now be confident that our bugfix actually solved the race condition here.

Give it a shot if you also need to debug concurrency issues in your codebase, but keep in mind that your mileage may vary. Thanks again to [Aleksey Shipilёv](https://twitter.com/shipilev) for eyeballing the code and giving suggestions.
