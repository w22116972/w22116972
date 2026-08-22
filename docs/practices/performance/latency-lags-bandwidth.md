# Latency lags bandwidth

> David A. Patterson. 2004. Latency lags bandwith. Commun. ACM 47, 10 (October 2004), 71–75. https://doi.org/10.1145/1022594.1022596

## Abstract

Bandwidth improves too much more quickly than latency. Even latency is limited by the speed of light, we can **improve latency by cache, replication, and prediction**.

## Introduction

#### Moore's Law
- bandwidth improved by faster transistors, more transistors, more pins in parallel
- latency improved by faster transistors, but
  - The distances that signals need to travel between different parts of a chip can contribute to increased latency

#### Distance sets lower bound to latency

- Speed of light: 300 meters corresponds to a latency of at least 1 microsecond.
- In DRAM, _the row access time (the time from issuing a row address to accessing data) is significantly affected by delays on the word lines and bit lines_. These delays constitute the largest portion of the row access time, impacting overall DRAM performance.
    
#### Reducing latency can increase bandwidth, but increasing bandwidth doesn't reduce latency.

Examples:
- Lower DRAM latency allows more DRAM accesses per second, resulting in higher bandwidth.
- A faster spinning disk reduces rotational latency, leading to higher bandwidth.
- Increasing the linear density of bits per inch on a track raises bandwidth but doesn't lower latency.

#### Buffer improves bandwidth but not latency

How Buffers Help Bandwidth:
Buffers maintain high bandwidth by smoothing out bursts of data traffic. They store short, high-speed data bursts and release the data at a steady rate, ensuring smooth and consistent processing without losing packets.

How Buffers Hurt Latency:
Buffers introduce latency because data has to wait in line before being processed. When a buffer is full, incoming data packets must wait longer, increasing the overall travel time from source to destination.

Example: Video Streaming
When streaming a video, data packets are buffered to ensure smooth playback. This buffering prevents interruptions even if network speed temporarily dips, maintaining good bandwidth usage. However, the initial buffering when you start the video causes a brief delay (latency) before the video begins to play.

In summary, buffers help manage bandwidth by handling data bursts efficiently but increase latency because of the time data spends waiting in the buffer.

## Methods to reduce latency

#### Cache

- Utilize the principle of locality of reference to enable a small, fast memory to handle the majority of accesses efficiently.
  - e.g. L1 cache, L2 cache, L3 cache
- File systems use a large main memory to act as a file cache

Cons: May increase miss rate.

#### Replication

- Place replicas close to users.
- Create database replicas.
- Processors replicate registers.

Cons: Higher communication costs between replicas.

#### Prediction

- Predict then prefetch data

Cons: Requires a recovery mechanism

#### Suboptimal: Lower bandwidth

Example: DRAM blocks and interfaces should be designed like SCSI(Small Computer System Interface) disk to reduce bandwidth and increase latency.

#### Suboptimal: Recalculation instead of fetching data from memory

Because latency to memory is approaching the time to execute a thousand instructions


## Keywords

#Latency, #Performance
