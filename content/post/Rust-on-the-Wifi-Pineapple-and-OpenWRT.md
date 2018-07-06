+++
date = "2017-01-02"
draft = false
title = "Rust on the WiFi Pineapple (and OpenWrt)"
description = "Learn how to compile Rust for the WiFi Pineapple and similar OpenWrt platforms."
tags = ["rust", "openwrt", "wifipineapple"]
slug = "Rust-on-the-Wifi-Pineapple-and-OpenWRT"
+++

Over the holidays I wanted to get a very simple [Rust](https://www.rust-lang.org) application running on my [WiFi Pineapple Nano](https://wifipineapple.com/). Since I don't have much experience with embedded systems and cross-compilation, it sounded like something fun to do and I was sure I might learn a thing or two. Many hours later and lots of frustration, I ended up with a simple solution that I'd like to share in this post.

The major obstacle to overcome is building your own standard library, since Rust doesn't ship with one for our target out of the box. I first tried it "the old way" by checking out the source and running through all the steps described in [rust-cross](https://github.com/japaric/rust-cross). While I got it to build in the end it still didn't work because of some version issues (I compiled with `rustc 1.16.0-dev (4ecc85beb 2016-12-28)` and the same version but `rustc 1.16.0-nightly (4ecc85beb 2016-12-28)` [rejected running it](https://gist.github.com/daschl/b4f87ae707ecff297a1602135f3f940b)).

Close to giving up at this point after spending hours on it, I found the [xargo](https://github.com/japaric/xargo) wrapper around `cargo`, an awesome project by [Jorge Aparicio](https://github.com/japaric). With this tool I got it to work fairly quickly and this is also the approach we'll be following in this post. We still need to handle a couple of extra steps not outlined in the `xargo` documentation, so I hope this post provides extra value to some of you.

## Which target?
Before we get into the weeds we need to do our homework first and figure out the compilation target for cross compilation. Once you SSH into your pineapple and run `uname -a` you'll see something like this:

```
root@Pineapple:~# uname -a
Linux Pineapple 3.18.36 #40 Fri Oct 28 05:42:22 UTC 2016 mips GNU/Linux
```

This is the output from my WiFi Pineapple NANO running the 1.1.3 firmware. The important part here is **mips**, which gives us a clue which platform the pineapple is running on. The device is built on top of [OpenWRT](https://openwrt.org/), a linux distribution for embedded devices.

The `/etc/openwrt_release` file provides more information about the OpenWrt release itself:

```
root@Pineapple:~# cat /etc/openwrt_release
DISTRIB_ID='OpenWrt'
DISTRIB_RELEASE='Chaos Calmer'
DISTRIB_REVISION='r49403'
DISTRIB_CODENAME='chaos_calmer'
DISTRIB_TARGET='ar71xx/generic'
DISTRIB_DESCRIPTION='OpenWrt Chaos Calmer 15.05.1'
DISTRIB_TAINTS='no-all no-ipv6 busybox'
```

Our target is ‘ar71xx/generic’, running OpenWrt ChaosCalmer 15.05.1. Looking for more info [here](https://dev.openwrt.org/wiki/platforms), we can see that its a [big endian](https://en.wikipedia.org/wiki/Endianness#Big-endian) platform based on the Atheros AR71xx/AR724x/913x chipset.

That gives us enough information to locate [this download page](https://downloads.openwrt.org/chaos_calmer/15.05.1/ar71xx/generic/) for our chipset and especially we need to download [the correct SDK](https://downloads.openwrt.org/chaos_calmer/15.05.1/ar71xx/generic/OpenWrt-SDK-15.05.1-ar71xx-generic_gcc-4.8-linaro_uClibc-0.9.33.2.Linux-x86_64.tar.bz2).

OpenWrt only provides the SDK for Linux as a host platform, so if you are running on Windows or OSX you'll need a virtual machine to follow the steps.

## OpenWRT SDK Setup
To make sure our toolchain works we can compile and run a simple "Hello World" from C code by using the SDK directly. This will make sure that we have everything set up properly, since Rust builds on those binaries at a later stage.

Unpack the archive and add these environment variables to your `.bashrc` (or equivalent) so that you don't run into path issues.

```
export PATH=$PATH:~/openwrt/staging_dir/toolchain-mips_34kc_gcc-4.8-linaro_uClibc-0.9.33.2/bin/
export STAGING_DIR=~/openwrt/staging_dir/
```

We are adding the SDK `gcc` binaries to our `PATH` and also provide the `STAGING_DIR` environment variable that the SDK needs while building. Note that I've renamed the `OpenWrt-SDK-15.05.1-ar71xx-generic_gcc-4.8-linaro_uClibc-0.9.33.2.Linux-x86_64` (wow...) directory after extraction simply to `openwrt` so its easier to read the command line. Note: make sure to reload/source your `.bashrc` file if you want to stay in the current terminal or open a new one so the changes take effect.

Here is a very simple hello world in C (`hello.c`):

```c
#include<stdio.h>

main() {
        printf("Hello, World!\n");
}
```

You can compile it with:

```
$ mips-openwrt-linux-gcc hello.c -o hello
```

If it compiles without issues, copy it to the device and run it!

```
$ scp hello root@172.16.42.1:~
$ ssh root@172.16.42.1

root@Pineapple:~# ./hello
Hello, World!
root@Pineapple:~#
```

## Cross-Compiling Rust
At this point we know that the SDK is set up correctly and we can compile and run C code on the pineapple just fine.

To proceed, we need a nightly rust version, ideally set up through [rustup](https://rustup.rs/) which is
the preferred way to manage your rust environments by now.

```
$ rustup install nightly
$ rustup default nightly
```

Don't forget to add the source component, otherwise it won't work down the road:

```
$ rustup component add rust-src
```

Finally, you should end up with a version similar to mine:

```
$ rustc -Vv
rustc 1.16.0-nightly (4ecc85beb 2016-12-28)
binary: rustc
commit-hash: 4ecc85beb339aa8089d936e450b0d800bdf580ae
commit-date: 2016-12-28
host: x86_64-unknown-linux-gnu
release: 1.16.0-nightly
LLVM version: 3.9
```

Now, install `xargo` and wait for it to compile:

```
$ cargo install xargo
```

We are pretty much set now, but we haven't figured out which `rustc` target to use. Looking at the long output of `rustc --print target-list`, the one we need to use is **mips-unknown-linux-uclibc** (since we are compiling against the `mips` architecture and this OpenWRT version is compiled with [uclibc](https://uclibc.org/))

Let's create a hello world project with cargo:

```
$ cargo new hello --bin
     Created binary (application) `hello` project
```

Before we can build it, there are a couple modifications we need to make. First, we need to adapt the `Cargo.toml` to `abort` on `panic`. I don't know why this is the case, but the `xargo` README says we
should do it and it also fails to compile otherwise.

```
$ cat Cargo.toml
[package]
name = "hello"
version = "0.1.0"
authors = ["you"]

[dependencies]

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

Next, we need to tell `cargo` to use a different linker for our `mips` target since the one on our host doesn't work for our target. This is done by providing a custom config within the `.cargo/config` file:

```
$ cat .cargo/config
[build]
target = "mips-unknown-linux-uclibc"

[target.mips-unknown-linux-uclibc]
linker = "mips-openwrt-linux-gcc"
```

Finally, we need to add a `Xargo.toml` in order to tell `xargo` to also build the standard library for us:

```
$ cat Xargo.toml
[target.mips-unknown-linux-uclibc.dependencies.std]
features = []
```

Now we are all set! Run `xargo build` (not `cargo build`) and watch the magic happen. The first time `xargo` will build the standard library for you, but subsequent builds will just reuse the compiled libraries so its even quicker. The second time it will look like this:

```
$ xargo build
   Compiling hello v0.1.0 (file:///home/vagrant/src/hello)
    Finished debug [unoptimized + debuginfo] target(s) in 0.17 secs
```

Looking into our `target/` directory we can see that there is now a `hello` binary
available for our `mips-unknown-linux-uclibc` target!

```
~/src/hello$ tree target/
target/
|-- debug
|   |-- build
|   |-- deps
|   |-- examples
|   `-- native
`-- mips-unknown-linux-uclibc
    `-- debug
        |-- build
        |-- deps
        |   `-- hello-e7c80b1b0618ec37
        |-- examples
        |-- hello           <----------- this one ------
        `-- native
```

Of course, running it on our host won't work:

```
$ ./target/mips-unknown-linux-uclibc/debug/hello
-bash: ./target/mips-unknown-linux-uclibc/debug/hello: cannot execute binary file: Exec format error
```

As a final step, copy the file over to your pineapple and run it (here I'm copying it to the mounted SD card just for fun):

```
$ scp target/mips-unknown-linux-uclibc/debug/hello root@172.16.42:/sd
$ ssh root@172.16.42.1 '/sd/hello'
Hello, world!
```

That's it! We just cross-compiled and ran a Rust program on a WiFi Pineapple.

## One last thing...
While researching this topic I was made aware that there is still an open issue for the [libc](https://github.com/rust-lang/libc) crate to fully support the `mips-unknown-linux-uclibc` target. It seems to work mostly but until [the issue](https://github.com/rust-lang/libc/issues/361) is resolved keep in mind that you might run into some weird issues.

Let me know if I missed anything or if you are also interested in running Rust on the WiFi Pineapple or similar OpenWrt devices! I'd like to close this post with a special thanks to [Jorge Aparicio](https://github.com/japaric) who is tirelessly working on making the cross compilation and embedded experience with Rust better every day ([xargo](https://github.com/japaric/xargo), [trust](https://github.com/japaric/trust), [cross](https://github.com/japaric/cross),...).
