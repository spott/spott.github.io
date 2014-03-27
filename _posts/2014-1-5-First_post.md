---
layout: post-no-feature
title: Toward a more optimized GPU reduce
---

This is my own interpretation of [these slides](http://developer.download.nvidia.com/assets/cuda/files/reduction.pdf), and some other information I have picked up on the way.  A lot of this is likely to be wrong, please [let me know](mailto:andrew.spott@gmail.com) what I have gotten wrong.[^knowledge]

# Background:

*A GPU is just a whole bunch of CPU cores, right?*

Before we can start to talk about the reduce algorithm, we need to talk about what is so different between a typical x86 multicore CPU, and the nvidia GPU that we will be doing our computation on.

## Throughput vs. Latency cores:

GPUs can be thought of as *throughput* cores, versus the more natural *latency* cores that make up your typical CPU.  A *latency* core is designed to reduce latency in operations, reducing the amount of time that is spent on any specific calculation.  It is essentially designed to take a sequence of instructions and execute them as quickly as possible.  To do this, it has a few prominent tools:  the cache and the branch prediction.  The cache allows the CPU to spend less time waiting on memory by grabbing lots of it at the same time, before you need it.  Branch prediction allows the CPU to "guess" at the output of an if statement and start executing that branch before it has finished executing the if statement itself.  If it guesses wrong, then it has to flush the pipeline[^pipeline] and restart; but if it guesses right (which it does on modern processors > 90% of the time), then it got a head start.  The CPU is fast because it calculates quickly.

*Throughput* cores are a different beast entirely.  A *throughput* core is designed to increase the number of calculations being done, at the expense of the speed of doing one calculation.  This means lots and lots of parallelism.  The tools in a *throughput* core's toolbox are simpler - no branch prediction or multilevel caches - but instead a long pipeline, and many, many cores.  The long pipeline allows 

Code that is optimized for GPUs is typically optimized for a latency optimized cores:  something like a classic CPU.  These chips have lots of optimizations that make classic sequential code run faster.  These optimizations include things like cache prefetching (to pull desired data out of main memory and put it into the faster L2 and L1 caches) and branch prediction (to reduce the cost of an if statment, one of the branches is guessed to be true, and both the if statement and the guessed branch are run through the pipeline simultaneously).  Most latency optimized cores have only a couple hardware threads available per core:  they aren't designed to quickly switch between threads.  However, throughput optimized cores are designed to quickly switch between multiple threads.  They reduce memory latency by quickly switching away from threads that have stalled, allowing other threads to run to fill up "dead time".

This means that cores that are throughput optimized are designed for many threads to be available (to reduce latency), and for many instructions to be run on many threads: so called "Single Instruction, Multiple Thread" cores.

## Structure of a parallel reduce kernel for throughput devices:

Throughput devices aren't able to handle multiple `blocks` being synchronized together, from within a single kernel.  For example, in CUDA:

    __syncthreads();

Only sychronizes threads inside a block: not all the threads that are being run.  The reason for this is clear from above:  it is difficult to get all threads sychronized *across multiple cores*, because that loses the greatest advantage that throughput devices have: the ability to quickly switch between cores.

Because of this 

[^knowledge]: This is the internet, so this shouldn't have to be said, but do not trust anything in this document if love or money depends on it.

[^pipeline]: the pipeline is 
