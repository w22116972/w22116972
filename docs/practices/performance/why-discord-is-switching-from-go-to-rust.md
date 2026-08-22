# Why Discord is switching from Go to Rust

## Abstract

Discord replace Golang with Rust to reduce latency spikes caused by garbage collection.

## Introduction

The Read State service is responsible for tracking which channels and messages users have read.
- Read States are accessed every time you connect to Discord, send a message, or read a message.
- There is one Read State per user per channel. Each Read State has several counters that need to be updated atomically and often reset to 0.
Each Read States server has an _LRU_ cache of Read States
- Tens of millions of Read States in each cache.
- Hundreds of thousands of cache updates per second.

## Problem: GC on GO causes 2 minutes spikes

Despite very few allocations, the Read State service still experiences large latency spikes every few minutes due to the impact of garbage collection.

In Go, memory is not immediately freed upon cache key eviction:
- During garbage collection, Go has to do a lot of work to determine what memory is free, which can slow the program down.

#### Subproblem 1: Go Forces a Garbage Collection Run Every 2 Minutes at Least

#### Failed Method: Tune garbage collector to collect more often to prevent a large spike

Efforts to tune the garbage collector to collect more often and prevent large spikes were unsuccessful. The result is that allocating memory quickly enough to trigger the garbage collector is actually difficult.

#### Subproblem 2: Main Cause of Latency Spikes

The main reason for the latency spikes is that the GC needs to scan the entire LRU cache to determine which memory is free, not because of a massive amount of memory ready to be freed.

#### Suboptimal Solution: Using a Smaller LRU cache

Using a smaller LRU cache results in smaller spikes but leads to higher 99th percentile latency times due to an increased miss rate.

## Main method: Rust

Rust frees memory immediately upon cache key eviction.

#### Other Methods

- Changing to a BTreeMap instead of a HashMap in the LRU cache to optimize memory usage.
- Reducing the number of memory copies when writing Rust codes

## References

- https://discord.com/blog/why-discord-is-switching-from-go-to-rust

## Keywords

#Rust, #Performance, #GC, #GarbageCollection
