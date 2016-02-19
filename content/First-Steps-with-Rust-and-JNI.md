+++
date = "2016-02-19"
draft = false
title = "First Steps with Rust and JNI"
description = "This post explains how to call into Rust code from Java (JNI) - simple but just enough to get you started."
tags = ["rust", "java", "jni"]
slug = "First-Steps-with-Rust-and-JNI"
+++

The first steps are always the hardest, at least thats how the saying goes. But it turns out that calling into [Rust](http://rust-lang.org/) from Java is easier than I originally thought. 

The following blog post shows you how to setup and compile a Rust library which can be called from Java userland. Note that everything you see in this post, while being functional, is very simplistic. Real world JNI has lots of nitty gritty details and pitfalls, but we need to start somewhere right?

Recently on Hacker News there has been rumor that [adding two integers is slow in java](https://www.youtube.com/watch?v=dQw4w9WgXcQ), so lets try to offload this complex operation into Rust.

Note that all steps performed in this blog post were done on OSX, but with little adaption they should also work on Linux. Maybe even on Windows, but I'm not so sure there since my experience with Windows is very limited. Any recent Rust version should suffice, I'm using 1.6.0 stable.

The first step is to create a `cargo` library project:

```
~/rust $ cargo new highperf-adder
```

Open the generated `Cargo.toml` and add `libc` as a dependency. When interacting with [JNI](https://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/jniTOC.html), Rust needs to dress up a bit to look like C, and `libc` helps with that.

While you're in there, tell `cargo` that you want to build the crate as a `dylib` and also give it an explicit name:

```
[lib]
name = "hpa"
crate-type = ["dylib"]

[dependencies]
libc = "0.2.7"
```

If you don't specify that you want a `dylib`, it will build a `rlib` which is not intended for external use. Also don't use `staticlib`, since this fail to link as well.

To make sure all is well so far we can build the project:

```
$ cargo build
Updating registry `https://github.com/rust-lang/crates.io-index`
Compiling libc v0.2.7
Compiling highperf-adder v0.1.0 (file:///Users/michael/rust/highperf-adder)
```

Instead of diving straight into Rust code, let's work on the Java code first. To keep it simple, create a `Adder.java` file in the root of the project directory and add the following code:

```java
import java.nio.file.Path;
import java.nio.file.Paths;

class Adder {

  static {
    Path p = Paths.get("target/debug/libhpa.dylib");
    System.load(p.toAbsolutePath().toString());
  }

  public static native int add(int v1, int v2);

  public static void main(String... args) {
    System.out.println("2 + 3 = " + Adder.add(2, 3));
  }

}
```

Let's break the code up into digestible chunks.

```java
static {
    Path p = Paths.get("target/debug/libhpa.dylib");
    System.load(p.toAbsolutePath().toString());
}
```

We need to tell the JVM to pick up our shared library that got built by Rust. If you are running `cargo build` with the `--release` flag to optimize, make sure to point it towards `target/release/libhpa.dylib` instead.

```java
public static native int add(int v1, int v2);
```

Next up we define all our native methods that we want to call through JNI. It works a little bit like implementing an abstract class, but you are implementing the actual code in Rust intead of Java userland. The name of the method and its calling class will become important in a bit, so if you want to follow along make sure you keep the same names.

In our case we define one `add` method which takes two integers as arguments and returns the result as an integer.

```java
public static void main(String... args) {
    System.out.println("2 + 3 = " + Adder.add(2, 3));
}
```

The `main` method calls our static JNI method and prints the result to `stdout`.

Let's run it straight away:

```
$ javac Adder.java 
$ java Adder
Exception in thread "main" java.lang.UnsatisfiedLinkError: Adder.add(II)I
    at Adder.add(Native Method)
    at Adder.main(Adder.java:14)
```

It compiles without errors, but at runtime it breaks apart. This doesn't come as a huge surprise since we didn't write a single line of Rust code yet. The JVM just tells us it can't link the method we're looking for. 

To fix that, open the `src/lib.rs` file and insert the following Rust code:

```rust
extern crate libc;

use libc::{c_void, c_int};

#[repr(C)]
pub struct JNINativeInterface {
    reserved0: *mut c_void,
    reserved1: *mut c_void,
    reserved2: *mut c_void,
    reserved3: *mut c_void,
    // much more actually in here for practical JNI code, but not
    // relevant for this very simple example...
}

pub type JNIEnv = *const JNINativeInterface;

#[no_mangle]
pub extern fn Java_Adder_add(jre: *mut JNIEnv, class: *const c_void, v1: c_int, v2: c_int) -> c_int {
    v1 + v2
}
```

Alright, so what's going on here? 

```rust
#[repr(C)]
pub struct JNINativeInterface {
    reserved0: *mut c_void,
    reserved1: *mut c_void,
    reserved2: *mut c_void,
    reserved3: *mut c_void,
    // ...
}

pub type JNIEnv = *const JNINativeInterface;
```

The JVM exports a `jni.h` header file which contains its primary interfaces when interacting through JNI. Normally we'd write a proper Rust C FFI binding here and keep our code idiomatic, but for now all we need is a pointer to the `JNIEnv` (even if we don't actually use it, it gets passed in to our `add` method as an argument). If you take a look at the [actual jni.h](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/javavm/export/jni.h#l214) you can see that we get away with lots of handwaving for now.

Btw, how on earth did we know that the JVM expects a method with the signature of `fn Java_Adder_add(jre: *mut JNIEnv, class: *const c_void, v1: c_int, v2: c_int) -> c_int`? The answer lies in the `javah` command and some conversion of C to Rust. If you run `javah Adder` you get a `Adder.h` file which looks like:

```c
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class Adder */

#ifndef _Included_Adder
#define _Included_Adder
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     Adder
 * Method:    add
 * Signature: (II)I
 */
JNIEXPORT jint JNICALL Java_Adder_add
  (JNIEnv *, jclass, jint, jint);

#ifdef __cplusplus
}
#endif
#endif
```

The important part is this:

```c
JNIEXPORT jint JNICALL Java_Adder_add(JNIEnv *, jclass, jint, jint);
```

The type `jint` is defined per plattform. For OSX you can find it [here](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/macosx/javavm/export/jni_md.h#l33) and it maps to an `int`. The rust `libc` exports this type through `libc::c_int`, which we can utilize in our code. `jclass` is a reference type which we ignore for now, since we don't need it.

All we need to do is add the two numbers and return the result:

```rust
#[no_mangle]
pub extern fn Java_Adder_add(jre: *mut JNIEnv, class: *const c_void, v1: c_int, v2: c_int) -> c_int {
    v1 + v2
}
```

Make sure you don't forget the `#[no_mangle]` which tells Rust to turn off its name mangling so that there are no issues while linking (try it out and you'll see the `UnsatisfiedLinkError` again). Finally its time to compile the `crate` once more and run the Java class again. Note that you don't need to recompile the Java code every time you make a change to the Rust code. Just rebuild with cargo and run the Java file again, it will pick up the freshly created `dylib`.

```
$ cargo build
   Compiling libc v0.2.7
   Compiling highperf-adder v0.1.0 (file:///Users/michael/rust/highperf-adder)
src/lib.rs:18:30: 18:33 warning: unused variable: `jre`, #[warn(unused_variables)] on by default
src/lib.rs:18 pub extern fn Java_Adder_add(jre: *mut JNIEnv, class: *const c_void, v1: c_int, v2: c_int) -> c_int {
                                           ^~~
src/lib.rs:18:48: 18:53 warning: unused variable: `class`, #[warn(unused_variables)] on by default
src/lib.rs:18 pub extern fn Java_Adder_add(jre: *mut JNIEnv, class: *const c_void, v1: c_int, v2: c_int) -> c_int {
                                                             ^~~~~
src/lib.rs:18:1: 20:2 warning: function `Java_Adder_add` should have a snake case name such as `java_adder_add`, #[warn(non_snake_case)] on by default
src/lib.rs:18 pub extern fn Java_Adder_add(jre: *mut JNIEnv, class: *const c_void, v1: c_int, v2: c_int) -> c_int {
src/lib.rs:19     v1 + v2
src/lib.rs:20 }
```

```
$ java Adder
2 + 3 = 5
```

It works! The warnings from `rustc` are there because the rust compiler is as always super correct and tells us we are neither making use of the `jre` nor of the `class` function arguments. Also, our method signature is not named like idiomatic rust code is. You can turn off those warnings by adding the attributes the compiler suggests.

I think a crate which abstracts the JNI C FFI would be pretty awesome to abstract the nitty gritty details and to expose a safe, idiomatic Rust API. I'm planning to work on that as time permits, let me know if you also want to hack on it. Finally, thanks to [Vladimir Mateev](https://twitter.com/netvlm) who [answered](http://stackoverflow.com/questions/30258427/calling-rust-from-java) a related question on stackoverflow last year which got me motivated to dive in further.