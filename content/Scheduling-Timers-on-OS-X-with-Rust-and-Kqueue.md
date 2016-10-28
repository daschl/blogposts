+++
date = "2016-06-15"
draft = false
title = "Scheduling Timers on OS X with Rust and Kqueue"
description = "Scheduling Timers on OS X is quite easy with Kqueue - this post shows you how."
tags = ["rust", "kqueue", "osx"]
slug = "Scheduling-Timers-on-OS-X-with-Rust-and-Kqueue"
+++

As a more or less POSIX compatible system I would've expected
[timer_create](http://man7.org/linux/man-pages/man2/timer_create.2.html) and
friends to be available on OS X, but it turns out those functions are not
available (at least I couldn't find them after hours of research).

Looking into alternatives (spoiler: there are not many I think if you want to
work from C/Rust) I settled on [Kqueue](https://en.wikipedia.org/wiki/Kqueue).
It doesn't have all the features that the `timer_` functions provide, but for
what I need it seems to be good enough. Also, there are a bunch of crates like
[mio](https://crates.io/crates/mio) and [nix](https://crates.io/crates/nix)
available that either provide abstractions or use Kqueue already so I had
something to refer to.

I didn't find documentation or blog posts on this topic so I decided it is
time to write one. This doesn't go into all the details since, frankly, I don't
know all of them yet too. It should be enough to get you started though.

## A Kqueue primer
Kqueue is a event notification system originally introduced in FreeBSD and
subsequently supported in many more BSD variants as well as OS X. It can be used
for similar tasks (like handling network connections) as the
[epoll](https://en.wikipedia.org/wiki/Epoll) system on Linux or the
[IOCP](https://en.wikipedia.org/wiki/Input/output_completion_port) framework on
Windows. You can also schedule timers with it, and this is what we are doing in
this blog post.

There are two main functions you need to work with:

```c
int kqueue(void);

int kevent(int kq, const struct kevent *changelist, int nchanges,
    struct kevent *eventlist, int nevents, const struct timespec *timeout);
```

which translate to the following rust signatures (from `nix`):

```rust
fn kqueue() -> Result<RawFd>
fn kevent(kq: RawFd, changelist: &[KEvent], eventlist: &mut [KEvent], timeout_ms: usize) -> Result<usize>
```

The `kqueue` function translates into a system call and creates a	new kernel
event queue and returns a descriptor. The `kevent` function is used to both
register new `KEvents` as well as check if any of them are currently pending.

The `KEvent` is a generic struct that describes the type of event to monitor
and looks like this in rust (also from `nix`):

```rust
pub struct KEvent {
    pub ident: uintptr_t,
    pub filter: EventFilter,
    pub flags: EventFlag,
    pub fflags: FilterFlag,
    pub data: intptr_t,
    pub udata: usize,
}
```

The `ident` holds a value which is used to identify the event. Depending on the
configured filter its type and meaning can change (but is very often a file
descriptor). The `filter` is important since it determines the kernel filter to
process this event. For our timers we'll use `EventFilter::EVFILT_TIMER`, but
there are many more available. The `flags` define which actions to perform
on the given event. The `fflags` allow you to configure filter-specific flags,
in our example we won't use them. Finally `data` allows to set filter-specific
values and `udata` is opaque user-defined data that is passed through the kernel
unchanged.

Check out [the docs in nix](http://rustdoc.s3-website-us-east-1.amazonaws.com/nix/master/osx/nix/sys/event/index.html)
for all the different values on `EventFlag`, `EventFilter` and `FilterFlag`. Also,
the original documentation on [kqueue](https://www.freebsd.org/cgi/man.cgi?query=kqueue)
provides lots of insights into the flags and their functionality.

The last thing you need to know before diving into the actual code is the difference
between `changelist` and `eventlist`: Both take a slice of `KEvents`, but only
`eventlist` is mutable.

 - The `changelist` is used to register events with kqueue.
 - The `eventlist` contains all the events which are currently active at the
   time of polling.

 Note that the same slice underlying container (like a `Vec`) can be used to
 maintain both lists.

 With the basics covered, let's dive into the code.

## Scheduling Timers

Create a new (`--bin`) project through `cargo` and add `nix` as a dependency:

```toml
[dependencies]
nix = "0.6.0"
```

Add this to the top of your `main.rs` file so we have the imports out of the
way:

```rust
extern crate nix;

use nix::sys::event::{KEvent, kqueue, kevent, EventFilter, FilterFlag};
use nix::sys::event::{EV_ADD, EV_ENABLE};
```

Next, we add a helper function that encapsulates the `KEvent` creation:

```rust
fn event(id: usize, timer: isize) -> KEvent {
    KEvent {
        ident: id,
        filter: EventFilter::EVFILT_TIMER,
        flags: EV_ADD | EV_ENABLE,
        fflags: FilterFlag::empty(),
        data: timer,
        udata: 0,
    }
}
```

Here the caller passes in the event id as well as the timer in milliseconds. Since
we want to get a timer event we need to use the `EventFilter::EVFILT_TIMER` filter.
The `EV_ADD | EV_ENABLE` indicates we want to add and enable the timer at the same
time. No flag filters are needed and the `data` payload for our timer event is the
time provided by the caller. We also don't set any opaque user data here.

Inside our `main` function we first need to grab a `kqueue` and then register
events. We make use of our `event` function here and create one event that
runs each second and one that runs every 1.5 seconds:

```rust
// Initialize the Kqueue
let kq = kqueue().expect("Could not get kqueue");

// Create a Vec<KEvent> with both events
let mut changes = vec![event(1, 1000), event(2, 1500)];

// Register the events in the `changelist`.
kevent(kq, changes.as_slice(), &mut [], 0).unwrap();
```

Kqueue now knows about the events we are interested in, so it's time to run
a loop and poll until they happen:

```rust
loop {
  match kevent(kq, &[], changes.as_mut_slice(), 0) {
    Ok(v) if v > 0 => {
      println!("---");
      for i in 0..v {
        println!("Event with ID {:?} triggered", changes.get(i).unwrap().ident);
      }
    }
    Err(e) => panic!("{:?}", e), // Panic on Errors
    _ => () // Ignore Ok(0),
  }
}
```

We poll `kevent` and the `changes` slice is updated with the results on each
poll with the `eventlist`. `kevent` returns either an `Err` or `Ok` with the
number of events that are available now. Note that `Ok(0)` is a special case
that nothing is available, so we move on. If we have at least one event pending
we iterate through all pending events and print their ID.

So if you run this example what you'll see is:

```
$ cargo run
       Fresh bitflags v0.4.0
       Fresh semver v0.1.20
       Fresh void v1.0.2
       Fresh cfg-if v0.1.0
       Fresh libc v0.2.12
       Fresh rustc_version v0.1.7
       Fresh nix v0.6.0
       Fresh kqueue-samples v0.1.0
     Running `target/debug/kqueue-samples`
---
Event with ID 1 triggered
---
Event with ID 2 triggered
---
Event with ID 1 triggered
---
Event with ID 1 triggered
Event with ID 2 triggered
---
Event with ID 1 triggered
---
Event with ID 2 triggered
---
^C
```

Since our timers fire at different intervals (and occasionally fire together) you
can see that every time the different events are available to process by your
application.

Of course this is barely scratching the surface of what you can do with `kqueue`
but I'd like to mention one final thing: as you can see even if you just
registered the events once they fire over and over again. If you want to fire
them just once you can use `flags: EV_ADD | EV_ENABLE | EV_ONESHOT` instead.
