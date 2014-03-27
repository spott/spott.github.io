---
layout: post-no-feature
title: GPU Architecture
---

 A lot of this is likely to be wrong, please [let me know](mailto:andrew.spott@gmail.com) what I have gotten wrong.

*A GPU is just a whole bunch of CPU cores, right?*

What is the difference between a typical x86 multicore CPU, and the nvidia GPU that we will be doing our computation on?

## Throughput vs. Latency cores:

GPUs can be thought of as *throughput* cores, versus the more natural *latency* cores that make up your typical CPU.  A *latency* core is designed to reduce latency in operations, reducing the amount of time that is spent on any specific calculation.  It is essentially designed to take a sequence of instructions and execute them as quickly as possible.  To do this, it has a few prominent tools:  the cache and the branch prediction.  The cache allows the CPU to spend less time waiting on memory by grabbing lots of it at the same time, before you need it.  Branch prediction allows the CPU to "guess" at the output of an if statement and start executing that branch before it has finished executing the if statement itself.  If it guesses wrong, then it has to flush the pipeline[^pipeline] and restart; but if it guesses right (which it does on modern processors > 90% of the time), then it got a head start.  The CPU is fast because it calculates quickly.

*Throughput* cores are a different beast entirely.  A *throughput* core is designed to increase the number of calculations being done, at the expense of the speed of doing one calculation.  This means lots and lots of parallelism.  The tools in a *throughput* core's toolbox are simpler - no branch prediction or multilevel caches - but instead a lot of hardware threads, a long pipeline and many, many cores.  The long pipeline allows for the many different threads to be running at the same time.  The lots of threads allows for the mitigation of memory latency by switching away from a thread when the data it needs hasn't arrived yet.  The GPU is fast because it calculates lots of things at the same time.

## Structure of a GPU:

In CUDA enabled devices, the device is logically broken up into three different *hierarchies*. These are, from largest to smallest:

* Grid: this is the sum total of all the threads that are being run for this kernel
* Block: a subset of those threads are run as a "block".  `__synchthreads()` can be used as a barrier between different threads inside a block.  
* Warp: usually 32 threads that are run in lockstep.  These threads are essentially running the same instruction at the same time on different pieces of data[^simd].

These different hierarchies share different levels of memory.

* Warp:  threads on the same warp share registers.
* Block: threads in the same block have fast shared memory in common.  This block of memory is also shared with the L1 cache, which can be be shared between different threads on the SM ("core" of the GPU)
* Grid:  all threads on the grid have access to main GPU memory.  They also share L2 cache, if it exists.

## What does it all mean?

This information allows us to think about performance on the GPU.  There are a few things that we can gleam from our limited information.

The GPU is best at executing the same instruction on lots of pieces of data.   For example, if we have an if statement that executes different branches for different threads on the same warp, we know that this will not be ideal:  approximately half the threads will sit idle while the others execute their branch.  We also know that data coalescing (accessing data in contiguous blocks) is important.  It allows the data to be put in the much faster shared memory and L1 cache, rather than  having to go all the way to main memory.  Since different blocks can't share memory, it is important to group your computations in such a way that threads in a block share data.  However, because all threads in a block are generally executed on the same SM, we want to split up our execution between different blocks to use the whole device.

[^pipeline]: The pipeline can be thought of as a bunch of instructions being executed back to back on different data.  For example, if there are two instructions that take 3 cycles to complete, a proper pipeline allows them both to complete in 5 cycles because they are running back to back.

[^simd]:  single instruction multiple data:  the "core" of the GPU takes multiple pieces of data but only one instruction at a time for that data.
