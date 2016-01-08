+++
date = "2016-01-08"
draft = false
title = "Discovering Hardware Topology in Rust"
description = "This post shows how you can learn more about your hardware topology using the Rust programming language and the hwloc library."
tags = ["rust", "hwloc"]
slug = "Discovering-Hardware-Topology-in-Rust"
+++

Todays programming languages and operation systems provide a bunch of abstraction layers over our hardware. Most of the time this is great, since we can write code quickly and make it run on lots of different machines. The opportunity cost with abstraction is (most of the time) performance and a lack of understanding.

To get the best performance out of hour hardware, it is important to understand it. Concepts like cache locality matter a lot, especially in modern NUMA architectures. Modern hardware is complex, so understanding it is not easy - and making matters worse its even harder to take advantage of this in our software stacks. For example binding threads or processes to specific cores is different in every operating system, some don't even support it.

One library which helps with hardware topology discovery and management is [hwloc](https://www.open-mpi.org/projects/hwloc/). The description on the website gives a nice overview:

> *The Portable Hardware Locality (hwloc) software package provides a portable abstraction (across OS, versions,
> architectures,...) of the hierarchical topology of modern architectures, including NUMA memory nodes, sockets, shared
> caches, cores and simultaneous multithreading. It also gathers various system attributes such as cache and memory
> information as well as the locality of I/O devices such as network interfaces, InfiniBand HCAs or GPUs. It primarily aims
> at helping applications with gathering information about modern computing hardware so as to exploit it accordingly and
> efficiently.*

As a side project I'm working on a [Rust Binding for hwloc](https://github.com/daschl/hwloc-rs) which you'll get to see in this blogpost. It already supports topology discovery and CPU binding, but only the discovery bit will be covered in this post.

I assume that you have basic rust skills, if not you want to check out [the book](https://doc.rust-lang.org/stable/book/) first. All the examples shown here should work on Linux and OSX, I never tested rust binding on windows (hwloc itself supports windows). If you want to help out adding support for it, that would be awesome!

## Setup
You need to have rust (for example 1.5.0) installed, as well as the `hwloc` C library. Please refer to the [README](https://github.com/daschl/hwloc-rs#install-hwloc-on-os-x) for detailed instructions on how to install it. On linux, most of the time you can find recent packages in your package manager, on OSX you want to install it from source. For example, here is all you need to do on OSX:

 - [Download](https://www.open-mpi.org/software/hwloc/v1.11/downloads/hwloc-1.11.2.tar.gz) the artifact.
 - tar -xvzpf hwloc-1.11.2.tar.gz
 - cd hwloc-1.11.2
 - ./configure && make && sudo make install

The next step is to create a rust project which will be runnable from the command line:

```
~/rust $ cargo new hwloc-playground --bin
```

Go modify your `Cargo.toml` and add `hwloc` as a dependency:

```
[dependencies]
hwloc = "0.2.0"
```

Execute `cargo run` to make sure the dependency compiles correctly before we move on:

```
~/rust/hwloc-playground $ cargo run
   Compiling winapi-build v0.1.1
   Compiling libc v0.2.4
   Compiling rustc-serialize v0.3.16
   Compiling bitflags v0.3.3
   Compiling pkg-config v0.3.6
   Compiling winapi v0.2.5
   Compiling advapi32-sys v0.1.2
   Compiling kernel32-sys v0.2.1
   Compiling rand v0.3.12
   Compiling errno v0.1.5
   Compiling hwloc v0.2.0
   Compiling num v0.1.29
   Compiling hwloc-playground v0.1.0 (file:///Users/michael/rust/hwloc-playground)
     Running `target/debug/hwloc-playground`
Hello, world!
```

`hwloc-rs` also provides rustdoc which you can find [here](http://nitschinger.at/hwloc-rs/rustdoc) for the different versions.

### Cores and Processing Units
As a first simple example, we'll find out how many physical cores we have on the machine. This introduces some basic concepts of `hwloc`. The full code sample looks like this:


```rust
extern crate hwloc;

use hwloc::{Topology, ObjectType};

fn main() {
    // Create a new Topology
    let topology = Topology::new();

    // Get all objects with type "Core"
    let cores = topology.objects_with_type(&ObjectType::Core);

    // Match on the returned Result and print the length if successful.
    match cores {
        Ok(c) => println!("There are {} cores on this machine.", c.len()),
        Err(e) => panic!(format!("Could not load cores because of: {:?}", e))
    }
}
```

Let's break the code apart a bit. The first thing you always need to do is create a [Topology](http://nitschinger.at/hwloc-rs/rustdoc/0.2.0/hwloc/struct.Topology.html). The `Topology` is your logical representation of the actual mapped hardware.

```rust
let topology = Topology::new();
```

Now that we have the `Topology`, we can ask it to return all [TopologyObjects](http://nitschinger.at/hwloc-rs/rustdoc/0.2.0/hwloc/struct.TopologyObject.html) with a specific [ObjectType](http://nitschinger.at/hwloc-rs/rustdoc/0.2.0/hwloc/enum.ObjectType.html). One type is `Core` which maps to a computation unit on the physical hardware (most of the time a CPU Core).

```rust
let cores = topology.objects_with_type(&ObjectType::Core);
```

Since not every ObjectType is available on every hardware, it might not be possible to figure out the actual objects. Because of this, the `objects_with_type` method returns a `Result<Vec<&TopologyObject>, TypeDepthError>`. We can now utilize pattern matching to distinguish between success and error, and if it is successful print out the length of the [Vector](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.len):

 ```rust
match cores {
    Ok(c) => println!("There are {} cores on this machine.", c.len()),
    Err(e) => panic!(format!("Could not load cores because of: {:?}", e))
}
```

If you run the code you should see an output like this (My machine is equipped with an i7 quadcore):

```
~/rust/hwloc-playground $ cargo run
   Compiling hwloc-playground v0.1.0 (file:///Users/michael/rust/hwloc-playground)
     Running `target/debug/hwloc-playground`
There are 4 cores on this machine.
```

Hwloc allows you to differentiate between cores and actual processing units. For example if your CPU has hyperthreading enabled, you'll end up with more logical processing units than physical cores. Let's modify the code to print the number of logical processing units instead:

```rust
// Get all objects with type "PU"
let cores = topology.objects_with_type(&ObjectType::PU);

// Match on the returned Result and print the length if successful.
match cores {
    Ok(c) => println!("There are {} processing units on this machine.", c.len()),
    Err(e) => panic!(format!("Could not load processing units because of: {:?}", e))
}
```

My i7 indeed has hyperthreading enabled, so this prints:

```
~/rust/hwloc-playground $ cargo run
   Compiling hwloc-playground v0.1.0 (file:///Users/michael/rust/hwloc-playground)
     Running `target/debug/hwloc-playground`
There are 8 processing units on this machine.
```

The library also allows us to walk the topology in tree form, so the logical processing units ("PU") are "below" the actual cores. If we want to determine how many PUs every core has, we can print that as well:

```rust
extern crate hwloc;

use hwloc::{Topology, ObjectType};

fn main() {
    // Create a new Topology
    let topology = Topology::new();

    // Get all objects with type "PU""
    let pus = topology.objects_with_type(&ObjectType::PU).ok().expect("Could not load PUs!");

    // Iterate through each PU
    for pu in &pus {
        // Print the PU's logical index.
        print!("PU #{} is on Core ", pu.logical_index());

        // Walk up the parent chain until the Core is found and print its id.
        let mut parent = pu.parent();
        while let Some(p) = parent {
            if p.object_type() == ObjectType::Core {
                println!("{}", p.logical_index());
                parent = None;
            } else {
                parent = p.parent();
            }
        }
    }
}
```

First, we load all the processing units:

```rust
let pus = topology.objects_with_type(&ObjectType::PU).ok().expect("Could not load PUs!");
```

Next up, we iterate through each unit and print its logical index:

```rust
// Iterate through each PU
for pu in &pus {
    // Print the PU's logical index.
    print!("PU #{} is on Core ", pu.logical_index());

    // ...
}
```

Finally, we can walk up the tree of parents for each unit until we arrive at the Core level and print out its logical index as well. Notice how easy the walking is with the `while let` construct. This works because `TopologyObject#parent()` returns an `Option<&TopologyOption>`.

```rust
// Walk up the parent chain until the Core is found and print its id.
let mut parent = pu.parent();
while let Some(p) = parent {
    if p.object_type() == ObjectType::Core {
        println!("{}", p.logical_index());
        parent = None;
    } else {
        parent = p.parent();
    }
}
```

On my machine this prints:

```
 ~/rust/hwloc-playground $ cargo run
   Compiling hwloc-playground v0.1.0 (file:///Users/michael/rust/hwloc-playground)
     Running `target/debug/hwloc-playground`
PU #0 is on Core 0
PU #1 is on Core 0
PU #2 is on Core 1
PU #3 is on Core 1
PU #4 is on Core 2
PU #5 is on Core 2
PU #6 is on Core 3
PU #7 is on Core 3
```

You can see that each Core has two PUs attached to it.

## Walking The Tree
If you want to understand the topology as a whole, you can walk it in two ways: either level by level (by depth) or in tree form. Here is a full example to walk it by level:

```rust
extern crate hwloc;

use hwloc::Topology;

fn main() {
    // Create a new Topology
    let topology = Topology::new();

    // Loop through the complete topology depth.
    for i in 0..topology.depth() {
		println!("*** Objects at level {}", i);

        // Print each object at each level.
		for (idx, object) in topology.objects_at_depth(i).iter().enumerate() {
			println!("{}: {}", idx, object);
		}
	}
}
```

The code iterates through each level until the final topology depth is reached and on each level it prints all the TopologyObjects. This is the result on my machine:

```
*** Objects at level 0
0: Machine ()
*** Objects at level 1
0: NUMANode16GB (16GB)
*** Objects at level 2
0: L3 (6144KB)
*** Objects at level 3
0: L2 (256KB)
1: L2 (256KB)
2: L2 (256KB)
3: L2 (256KB)
*** Objects at level 4
0: L1d (32KB)
1: L1d (32KB)
2: L1d (32KB)
3: L1d (32KB)
*** Objects at level 5
0: Core ()
1: Core ()
2: Core ()
3: Core ()
*** Objects at level 6
0: PU ()
1: PU ()
2: PU ()
3: PU ()
4: PU ()
5: PU ()
6: PU ()
7: PU ()
```

Here you can see new ObjectTypes in action. My machine has one NUMANode with 16GB of RAM. Then you can see the L3, L2 and L1 caches, as well as the individual Cores and logical processing units.

A different way to visualize it is through a tree representation:

```rust
extern crate hwloc;

use hwloc::{Topology, TopologyObject};

fn main() {
    // Create a new Topology
	let topo = Topology::new();

    // Print the tree and start at the root
	println!("*** Printing overall tree");
	print_children(&topo, topo.object_at_root(), 0);
}

// Print children recursively
fn print_children(topo: &Topology, obj: &TopologyObject, depth: usize) {
    // some padding for the tree print
	let padding = std::iter::repeat(" ").take(depth).collect::<String>();
	println!("{}{}: #{}", padding, obj, obj.os_index());

	for i in 0..obj.arity() {
		print_children(topo, obj.children()[i as usize], depth + 1);
	}
}
```

We define a method called `print_children` which is called recursively:

```rust
fn print_children(topo: &Topology, obj: &TopologyObject, depth: usize)
```

Note the funky padding variable is just there to create a left-padding for the tree view:

```rust
let padding = std::iter::repeat(" ").take(depth).collect::<String>();
```

Running the code above gives the same information but in a different style:

```rust
 ~/rust/hwloc-playground $ cargo run
   Compiling hwloc-playground v0.1.0 (file:///Users/michael/rust/hwloc-playground)
     Running `target/debug/hwloc-playground`
*** Printing overall tree
Machine (): #0
 NUMANode16GB (16GB): #0
  L3 (6144KB): #0
   L2 (256KB): #0
    L1d (32KB): #0
     Core (): #0
      PU (): #0
      PU (): #1
   L2 (256KB): #1
    L1d (32KB): #1
     Core (): #1
      PU (): #2
      PU (): #3
   L2 (256KB): #2
    L1d (32KB): #2
     Core (): #2
      PU (): #4
      PU (): #5
   L2 (256KB): #3
    L1d (32KB): #3
     Core (): #3
      PU (): #6
      PU (): #7
```

What you can spot here immediately is that each core has its own L2 cache (as well as a L1 data cache) while at the same time they all share the same L3 cache.

## CPU Caches
Speaking of caches, it is often quite helpful to know how much memory and cache is available to each core/processing unit. This can be used to aid cpu and memory binding decisions which are concerned with data locality.

Let's say we want to know how much cache our first logical processing unit has available:

```rust
extern crate hwloc;

use hwloc::{Topology, ObjectType};

fn main() {
    // Create a new Topology
	let topo = Topology::new();

    // Get the first Logical Processing Unit
	let pu = topo.objects_with_type(&ObjectType::PU).unwrap()[0];

	let mut parent = pu.parent();
	let mut levels = 0;
	let mut size = 0;

    // Walk up the parents and if it is a cache, add up its capacity
	while let Some(p) = parent {
		if p.object_type() == ObjectType::Cache {
			levels += 1;
			size += p.cache_attributes().unwrap().size;
		}
		parent = p.parent();
	}

    // Print out the result
	println!("*** Logical processor 0 has {} caches totalling {} KB", levels, size / 1024);
}
```

We are using a similar technique as in the examples above, but this time we just add up all the capacity for each cache and then print it out.

```
 ~/rust/hwloc-playground $ cargo run
     Running `target/debug/hwloc-playground`
*** Logical processor 0 has 3 caches totalling 6432 KB
```

This output is not surprising given we have seen the topology before:

```
...
  L3 (6144KB): #0
   L2 (256KB): #0
    L1d (32KB): #0
     Core (): #0
      PU (): #0 <----
      PU (): #1
...
```

For a quick crosscheck, this is what OSX shows in its `System Report`:

```
Processor Name:	Intel Core i7
Processor Speed:	2,3 GHz
Number of Processors:	1
Total Number of Cores:	4
L2 Cache (per Core):	256 KB
L3 Cache:	6 MB
Memory:	16 GB
```

## Conclusion
Hwloc and the rust binding provide a very convenient way to identify hardware topology characteristics. While hwloc has much more to offer, this post should have given you an easy introduction and should motivate you discovering your own topologies. In a followup post I'll show you how you can utilize the binding API to perform CPU binding if your OS supports it.

The rust binding is still in the works and I appreciate all kinds of input. Bug reports, API enhancements or just questions in general are more than welcome!
