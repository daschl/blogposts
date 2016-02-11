+++
date = "2016-02-11"
draft = false
title = "Binding Threads And Processes to CPUs in Rust"
description = "Using the hwloc library and the rust binding you'll learn how to bind your threads and processes to CPU cores."
tags = ["rust", "hwloc", "cpu"]
slug = "Binding-Threads-And-Processes-to-CPUs-in-Rust"
+++

In the [previous post](http://nitschinger.at/Discovering-Hardware-Topology-in-Rust) I've introduced the [hwloc-rs](https://github.com/daschl/hwloc-rs) library, which allows you to discover and manage hardware topologies. Discovering the capabilities of a machine is insightful, but it gets more interesting if you can perform certain actions based on those insights.

Binding threads or processes to distinct CPU cores is very important in high performance applications to isolate workloads, keep inter-core messaging latency to a minimum and also to prevent the operating system from relocating your threads between cores as it sees fit. This becomes even more important in [NUMA](https://en.wikipedia.org/wiki/Non-uniform_memory_access) architectures, where the memory access latency depends on the memory location relative to the processors (binding memory chunks, while also supported by `hwloc` is not covered in this post).

[Example Benchmarks](https://www.open-mpi.org/projects/hwloc/tutorials/20150921-EuroMPI-hwloc-tutorial.pdf) with [OpenMPI](https://www.open-mpi.org/) and [Intel MPI](https://software.intel.com/en-us/intel-mpi-library) on a 12-core Xeon E5 show that the throughput and latency vary greatly when passing messages between cores. Between cores on the same NUMA node the latency is around 330ns with a throughput of 4220MiB/s, but once messages need to cross the boundaries between cores in different NUMA nodes the latency shoots up to 590ns and the throughput drops to 3410MiB/s. 

As always with such low-level concerns like this, the APIs differ across operating systems and some platforms don't even support CPU binding at all. This is where `hwloc` shines again - it provides us with easy to use abstractions that we can readily use in our rust code. The following blog post explains the different options (checking for support, process binding, thread binding) in greater detail.

The [docs](http://nitschinger.at/hwloc-rs/rustdoc/0.3.0/hwloc/index.html) provide helpful instructions to get started, but make sure you pick up at least version `0.3.0` if you want to try it out:

```
[dependencies]
hwloc = "0.3.0"
```

## Checking for Support
Before even thinking about binding your thread or process to a specific core you need to check whether your target platform supports it. Spoiler: if you are thinking about trying this on OSX, you are out of luck. But this gives us a chance to compare the output of the following code on Linux and OSX:

```rust
extern crate hwloc;

use hwloc::Topology;

/// Example on how to check for specific topology support of a feature.
fn main() {
    let topo = Topology::new();

    // Check if Process Binding for CPUs is supported
    println!("CPU Binding (current process) supported: {}", topo.support().cpu().set_current_process());
    println!("CPU Binding (any process) supported: {}", topo.support().cpu().set_process());

    // Check if Thread Binding for CPUs is supported
    println!("CPU Binding (current thread) supported: {}", topo.support().cpu().set_current_thread());
    println!("CPU Binding (any thread) supported: {}", topo.support().cpu().set_thread());

    // Debug Print all the Support Flags
    println!("All Flags:\n{:?}", topo.support());
}

```

On Linux:

```
CPU Binding (current process) supported: true
CPU Binding (any process) supported: true
CPU Binding (current thread) supported: true
CPU Binding (any thread) supported: true
All Flags:
TopologyDiscoverySupport { pu: 1 }, TopologyCpuBindSupport { set_thisproc_cpubind: 1, get_thisproc_cpubind: 1, set_proc_cpubind: 1, get_proc_cpubind: 1, set_thisthread_cpubind: 1, get_thisthread_cpubind: 1, set_thread_cpubind: 1, get_thread_cpubind: 1, get_thisproc_last_cpu_location: 1, get_proc_last_cpu_location: 1, get_thisthread_last_cpu_location: 1 }, TopologyMemBindSupport { (omitted) }
```

On OSX:

```
CPU Binding (current process) supported: false
CPU Binding (any process) supported: false
CPU Binding (current thread) supported: false
CPU Binding (any thread) supported: false
All Flags:
TopologyDiscoverySupport { pu: 1 }, TopologyCpuBindSupport { set_thisproc_cpubind: 0, get_thisproc_cpubind: 0, set_proc_cpubind: 0, get_proc_cpubind: 0, set_thisthread_cpubind: 0, get_thisthread_cpubind: 0, set_thread_cpubind: 0, get_thread_cpubind: 0, get_thisproc_last_cpu_location: 0, get_proc_last_cpu_location: 0, get_thisthread_last_cpu_location: 0 }, TopologyMemBindSupport { (omitted) }
```

As a result, the following sections use a (virtual) Linux machine with 4 cores to demonstrate the binding capabilities. To give you some context, here is the `lstopo` output of the VM:

```
$ uname -a
Linux vagrant-ubuntu-trusty-64 3.13.0-71-generic #114-Ubuntu SMP Tue Dec 1 02:34:22 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux

$ lstopo -p --no-io
Machine (490MB) + Socket P#0 + L2d (6144KB)
  L1d (32KB) + Core P#0 + PU P#0
  L1d (32KB) + Core P#1 + PU P#1
  L1d (32KB) + Core P#2 + PU P#2
  L1d (32KB) + Core P#3 + PU P#3  
```

## The CpuSet
One important type to know when performing CPU binding operations is the [CpuSet](http://nitschinger.at/hwloc-rs/rustdoc/0.3.0/hwloc/type.CpuSet.html). The `CpuSet` is just a type alias for a generic [Bitmap](http://nitschinger.at/hwloc-rs/rustdoc/0.3.0/hwloc/struct.Bitmap.html) which has its bits set according to CPU physical OS indexes.

You can create a `CpuSets` instance too, but in general you will retrieve them through the topology or its objects, then copy/modify it and finally use it for your custom CPU binding. Every bitmap implements the `Display` and the `Debug` trait (amongst others), so printing their values is often a good idea. The next examples will make heavy use of `CpuSets`, so make sure to browse around the API a bit and make yourself familiar with it.

## Process Binding
If your platform supports it as well, `hwloc` provides two different ways to bind a process. You can either bind the current process or an arbitrary process identified by its process ID (commonly referred to as `pid`).

### Binding the Current Process

Here is an example which binds the current process to the last core available:

```rust
extern crate hwloc;

use hwloc::{Topology, TopologyObject, ObjectType, CPUBIND_PROCESS};

fn main() {
    let mut topo = Topology::new();

    let mut cpuset = last_core(&mut topo).cpuset().unwrap();
    cpuset.singlify();

    println!("Cpu Binding before explicit bind: {:?}", topo.get_cpubind(CPUBIND_PROCESS));
    println!("Cpu Location before explicit bind: {:?}", topo.get_cpu_location(CPUBIND_PROCESS));

    match topo.set_cpubind(cpuset, CPUBIND_PROCESS) {
        Ok(_) => println!("Correctly bound to last core"),
        Err(e) => println!("Failed to bind: {:?}", e)
    }

    println!("Cpu Binding after explicit bind: {:?}", topo.get_cpubind(CPUBIND_PROCESS));
    println!("Cpu Location after explicit bind: {:?}", topo.get_cpu_location(CPUBIND_PROCESS));
}

/// Helper method to find the last core
fn last_core(topo: &mut Topology) -> &TopologyObject {
    let core_depth = topo.depth_or_below_for_type(&ObjectType::Core).unwrap();
    let all_cores = topo.objects_at_depth(core_depth);
    all_cores.last().unwrap()
}
```

This prints the following on the linux machine:

```
Cpu Binding before explicit bind: Some(0-3)
Cpu Location before explicit bind: Some(2)
Correctly bound to last core
Cpu Binding after explicit bind: Some(3)
Cpu Location after explicit bind: Some(3)
```

So let's break it apart a bit. The first thing we need to do is find the `CpuSet` for the last core so we have a reference to bind it to. Note that this `singlify` call here is useful so that the process does not have a chance of migrating between multiple logical CPUs in the original mask. 

```rust
let mut cpuset = last_core(&mut topo).cpuset().unwrap();
cpuset.singlify();

//...

fn last_core(topo: &mut Topology) -> &TopologyObject {
    let core_depth = topo.depth_or_below_for_type(&ObjectType::Core).unwrap();
    let all_cores = topo.objects_at_depth(core_depth);
    all_cores.last().unwrap()
}
```

Now that we have our "target", we can start binding the current process there. To visualise what's going on, we also print the binding and location for the current process before and after the explicit binding:

```rust
println!("Cpu Binding before explicit bind: {:?}", topo.get_cpubind(CPUBIND_PROCESS));
println!("Cpu Location before explicit bind: {:?}", topo.get_cpu_location(CPUBIND_PROCESS));

match topo.set_cpubind(cpuset, CPUBIND_PROCESS) {
    Ok(_) => println!("Correctly bound to last core"),
    Err(e) => println!("Failed to bind: {:?}", e)
}

println!("Cpu Binding after explicit bind: {:?}", topo.get_cpubind(CPUBIND_PROCESS));
println!("Cpu Location after explicit bind: {:?}", topo.get_cpu_location(CPUBIND_PROCESS));
```

The current `CpuSet` of the process (which you can retrieve through `get_cpubind(CPUBIND_PROCESS)`) contains all possibles cores where the operating system might dispatch the process on. In our case it prints `0-3` which means all four cores are possible. The call to `get_cpu_location()` gives us the current core location, but this can change between subsequent calls as the operating system moves the process around.

Finally we override the current binding with our custom one (the new `CpuSet` only contains the last core rather than all four) and apply some simple matching to make sure the binding didn't fail for some reason. The last `println!` calls are just there to visually validate the new binding.

### Binding an Arbitrary Process
Binding any process works very similar to binding the current process, but there is one difference - we need to find the `pid` of the process we want to bind. This is a little out of scope for this blog post, but since our own process also has a `pid` we can use that one for our examples. Unfortunately we need a little bit of unsafe `libc` magic there:

```rust
extern crate libc;

fn main() {
  //...
  let pid = unsafe { libc::getpid() };
  //...
}
```

Once we have our pid, we will reuse the code from the last example to get the last core where we want to bind the process to:

```rust
fn main() {
  //...
  let mut cpuset = last_core(&mut topo).cpuset().unwrap();
  //...
}

/// Find the last core
fn last_core(topo: &mut Topology) -> &TopologyObject {
    let core_depth = topo.depth_or_below_for_type(&ObjectType::Core).unwrap();
    let all_cores = topo.objects_at_depth(core_depth);
    all_cores.last().unwrap()
}
```

Now we can use the same methods as previously, but with the `for_process()` suffix. Here is the full example, again with some debug print statements to visualise what's going on:

```rust
extern crate hwloc;
extern crate libc;

use hwloc::{Topology, CPUBIND_PROCESS, TopologyObject, ObjectType};

fn main() {
    let mut topo = Topology::new();

    let pid = unsafe { libc::getpid() };
    println!("Binding Process with PID {:?}", pid);

    let mut cpuset = last_core(&mut topo).cpuset().unwrap();
    cpuset.singlify();

    println!("Before Bind: {:?}", topo.get_cpubind_for_process(pid, CPUBIND_PROCESS).unwrap());
    println!("Last Known CPU Location: {:?}", topo.get_cpu_location_for_process(pid, CPUBIND_PROCESS).unwrap());

    topo.set_cpubind_for_process(pid, cpuset, CPUBIND_PROCESS).unwrap();

    println!("After Bind: {:?}", topo.get_cpubind_for_process(pid, CPUBIND_PROCESS).unwrap());
    println!("Last Known CPU Location: {:?}", topo.get_cpu_location_for_process(pid, CPUBIND_PROCESS).unwrap());
}

/// Find the last core
fn last_core(topo: &mut Topology) -> &TopologyObject {
    let core_depth = topo.depth_or_below_for_type(&ObjectType::Core).unwrap();
    let all_cores = topo.objects_at_depth(core_depth);
    all_cores.last().unwrap()
}
```

If we run this on our linux box, this is the output:

```
Binding Process with PID 3034
Before Bind: 0-3
Last Known CPU Location: 3
After Bind: 3
Last Known CPU Location: 3
```

So as long as you have the process ID available and the operating system supports it, you can bind every process to any number of cores you want. This is especially helpful if you need to bind forked processes or if you need to write some kind of babysitter service that needs to keep track and orchestrate a number of processes.

## Thread Binding
In addition to bind the process as a whole, it is also possible to pin individual threads inside a process to cores. Every thread has a unique thread ID (`tid`) which is used to bind a thread to a core.

The following example will spawn one thread for each core in the system and then bind each thread to one of the cores. Here is the full code in its beauty, we'll break it apart afterwards:


```rust
extern crate hwloc;
extern crate libc;

use hwloc::{Topology, ObjectType, CPUBIND_THREAD, CpuSet};
use std::thread;
use std::sync::{Arc,Mutex};

fn main() {
    let topo = Arc::new(Mutex::new(Topology::new()));

    let num_cores = {
        let topo_rc = topo.clone();
        let topo_locked = topo_rc.lock().unwrap();
        (*topo_locked).objects_with_type(&ObjectType::Core).unwrap().len()
    };
    println!("Found {} cores.", num_cores);

    let handles: Vec<_> = (0..num_cores).map(|i| {
            let child_topo = topo.clone();
            thread::spawn(move || {
                let tid = unsafe { libc::pthread_self() };
                let mut locked_topo = child_topo.lock().unwrap();

                let before = locked_topo.get_cpubind_for_thread(tid, CPUBIND_THREAD);

                let bind_to = cpuset_for_core(&*locked_topo, i);

                locked_topo.set_cpubind_for_thread(tid, bind_to, CPUBIND_THREAD).unwrap();

                let after = locked_topo.get_cpubind_for_thread(tid, CPUBIND_THREAD);

                println!("Thread {}: Before {:?}, After {:?}", i, before, after);
            })
        }).collect();

        for h in handles {
            h.join().unwrap();
        }
}

fn cpuset_for_core(topology: &Topology, idx: usize) -> CpuSet {
    let cores = (*topology).objects_with_type(&ObjectType::Core).unwrap();
    match cores.get(idx) {
        Some(val) => val.cpuset().unwrap(),
        None => panic!("No Core found with id {}", idx)
    }
}
```

Before the binding we need to identify the number of cores - that's an easy task for `hwloc`:

```rust
let num_cores = {
    let topo_rc = topo.clone();
    let topo_locked = topo_rc.lock().unwrap();
    (*topo_locked).objects_with_type(&ObjectType::Core).unwrap().len()
};
println!("Found {} cores.", num_cores);
```

The code finds all cores through `objects_with_type(&ObjectType::Core)` and counts them. Note that we need to do use proper rust synchronization mechanisms around our `Topology` since we are accessing it from multiple threads in the code.

The next piece spawns one thread for each core and joins on the main thread to wait until they complete:

```rust
let handles: Vec<_> = (0..num_cores).map(|i| {
        let child_topo = topo.clone();
        thread::spawn(move || {
            // do our other stuff
        })
    }).collect();

    for h in handles {
        h.join().unwrap();
    }
```

Next up we load the current thread ID through some `libc` unsafe magic, lock the `Topology` for safety and then read the current `CpuSet` for the thread (to print it out later):

```rust
let tid = unsafe { libc::pthread_self() };
let mut locked_topo = child_topo.lock().unwrap();

let before = locked_topo.get_cpubind_for_thread(tid, CPUBIND_THREAD);

```

The next part is important:

```rust
let bind_to = cpuset_for_core(&*locked_topo, i);

locked_topo.set_cpubind_for_thread(tid, bind_to, CPUBIND_THREAD).unwrap();
```

The helper function `cpuset_for_core` accepts an integer which represents the thread number (not the `tid`) and loops through the cores available on the `Topology`. It then returns the right `CpuSet` for the given index, so the first thread will be pinned to Core 0, the second one to Core 1 and so forth. Then, we use the `set_cpubind_for_thread()` method to bass in the current thread id as well as the `CpuSet` to apply and the `CPUBIND_THREAD` identifier which is needed.

Finally we just collect the new binding and then print it out for visualisation:

```rust
let after = locked_topo.get_cpubind_for_thread(tid, CPUBIND_THREAD);

println!("Thread {}: Before {:?}, After {:?}", i, before, after);
```

Running this on our 4-core machine prints:

```
Found 4 cores.
Thread 0: Before Some(0-3), After Some(0)
Thread 1: Before Some(0-3), After Some(1)
Thread 2: Before Some(0-3), After Some(2)
Thread 3: Before Some(0-3), After Some(3)
```

## Conclusion
Which processes or threads to bind is purely an application concern, but the underlying mechanics are greatly simplified through the `hwloc` abstractions. Combining the topology discovery with the CPU binding support allows you to choose the most optimised deployment option at runtime and giving you reasonable fallback options if the most performant way is not supported on the target.

Looking ahead, the rust binding is pretty much complete on discovery and CPU binding (modulo some advanced APIs that are yet to come), but the big piece missing is memory binding. Since rust itself pretty much abstracts the whole memory management story, it's not as easy as exposing the custom memory allocation functions of `hwloc`. I'm currently trying to wrap my head around good abstractions for this, so every input is very much appreciated! Please also let me know of any bugs you find or enhancements/clarifications you'd like to see in the rust binding.