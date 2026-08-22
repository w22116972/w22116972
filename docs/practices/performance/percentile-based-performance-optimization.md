# Percentile-Based Performance Optimization

#### Abstract
This guide provides a practical framework for understanding and optimizing service latency across key percentiles — P50, P95, and P99 — each representing different performance challenges and tuning strategies.

## P50 (Median Latency)

Measures the latency experienced by the middle 50% of requests — half of all requests are faster, and half are slower. Reflects the typical user experience and is a baseline for system responsiveness under normal conditions.

#### Optimization Goal

Improve critical path

#### Application-level optimizations

- Efficient algorithm and data structures
- Caching at code level
- Avoid synchronous I/O in critical path
- Compressed API payload
- API Pagination
- Large, uncompressed API Payloads: Payload Optimization - Compression, Field Filtering, Pagination)

#### Database-level optimizations

- Query Optimization: Simplify complex queries, reduce the use of expensive joins or aggregations.
- Efficient Query Writing & Specific Column Selection (for typical queries): Makes routine database interactions fast
- Basic Indexing for Common Queries: Ensures that typical lookups and filtering operations are performant
- Schema Optimization (Normalization, appropriate data types): Improves overall database efficiency for standard operations.
- Stored Procedures (for common, complex database logic): Can improve performance for routine complex tasks executed within the database.
- Prepared Statements: Reuse compiled query plans to reduce overhead.

#### Network-level

- Excessive Microservice Chattiness / Inefficient Protocol:  Efficient Network Protocols like gRPC, Payload Optimization

## P95 (Under Bursty Load Latency)

Measures the latency experienced by the slowest 5% of requests under load surges. Useful for identifying bottlenecks not visible in typical traffic (P50) but occurring more frequently than rare tail spikes (P99).

#### Optimization Goal
Handle increased concurrency and throughput spikes without degradation

#### Application-level optimizations

- Asynchronous Processing (including Async/Non-blocking Database Access): Essential for handling a high number of concurrent requests during load surges without exhausting resources (like threads). Using asynchronous or reactive database clients specifically prevents application threads from being blocked by database I/O, directly improving throughput and the ability to handle increased concurrency.
- Parallelization: Can improve the throughput of individual requests that involve multiple independent tasks, helping the system process more work overall during a spike.
Batching (especially for writes or similar operations that come in bursts): Reduces per-operation overhead, significantly improving throughput when handling a surge of many small operations.
- API Design - Payload Optimization (Compression, Selective Field Filtering, Pagination/Streaming for large responses): Minimizing data transfer and processing per request becomes critical during load spikes. This includes not only compressing data and selecting necessary fields but also avoiding the return of overly large result sets in a single API response. Implementing pagination or streaming results ensures that requests for large amounts of data don't overwhelm the network or server resources, which is crucial for P95.
- API Design - Efficient Network Protocols (e.g., gRPC, HTTP/3 with multiplexing): These protocols are designed for high concurrency and reduced overhead, making them more resilient and efficient during traffic surges.

#### Database-level optimization

- Eliminate N+1 query patterns: These patterns, where one query triggers N subsequent queries, often don't drastically affect median latency (P50) if individual queries are fast. However, under load or for entities with many related items, they can dominate higher percentiles (P95/P99) due to the multiplied effect of numerous small database calls. Removing them (e.g., via eager loading or batch fetching) is critical for P95
- Avoid full table scans
- Pagination / Streaming for Database Result Sets: Similar to API responses, avoid returning excessively large result sets from a single database query. Use database-level cursors, `LIMIT`/`OFFSET` for pagination, or streaming mechanisms where appropriate to manage large data volumes efficiently and prevent individual queries from becoming P95 outliers due to data size.
- Advanced Caching Strategies (with high hit ratios): A well-implemented cache acts as a shock absorber during load surges, serving a large fraction of requests without hitting the underlying, more resource-intensive database.
- Data Partitioning, Sharding, and Scale-Out Architectures (e.g., TiDB, CockroachDB): These are fundamental for database scalability. Sharding distributes the load across multiple database instances.
- Separate Read/Write Paths (e.g., Using Read Replicas): Offload read-heavy traffic from the primary database instance to read replicas. This reduces contention on the primary, allowing it to handle writes and more complex transactions efficiently, especially during load spikes. This improves overall throughput and resilience, directly benefiting P95.
- Scale-out DBMS allow for adding more nodes to handle increased throughput and data volume during sustained or bursty high-traffic periods.

#### Infra and network level optimizations

- CDN: absorb a massive amount of traffic during surges. Regional deployments help distribute load
- Network Traffic Management (Compression, Efficient Protocols): Helps manage increased data flow during bursts.
- Effective Load Balancing Algorithms: Essential for efficiently distributing sudden spikes in traffic.
- Autoscaling: This is a direct mechanism for handling bursty loads by dynamically adjusting resources.

## P99 (Tail latency)

Measures the latency experienced by the slowest 1% of requests. Highlights rare but impactful performance outliers, often caused by resource contention, long GC pauses, cold caches, or other edge-case conditions. Critical for maintaining system reliability and meeting strict SLAs/SLOs.

#### Optimization Goal

Improve the slowest requests that occur under edge-case conditions

#### JVM Challenge: GC Pauses, "stop-the-world"

- Heap Sizing: Optimize the initial and maximum heap sizes. An adequately sized heap reduces GC frequency, but an overly large heap can increase pause durations
- GC Algorithm Tuning: Select an appropriate garbage collector for the application's workload. Modern collectors like G1GC (Garbage-First Garbage Collector) and ZGC (Z Garbage Collector) are designed for low-latency applications, aiming to keep pause times minimal even with large heaps. Generational ZGC, available in newer JDK versions like JDK 21, can further reduce P99 latencies by minimizing the amount of work that needs to be done by application threads during GC cycles
- Code Optimizations: Minimize unnecessary object creation to reduce memory pressure and GC frequency. Reuse objects where possible (e.g., using StringBuilder for string concatenations).
- Profiling and Analysis: Use profiling tools like VisualVM or JProfiler to analyze memory usage patterns, identify memory leaks, and understand GC behavior

#### Application-level

- Default or Missing Timeouts for External Calls: Resilience Patterns (Strategic Timeouts, Circuit Breaker)

#### Cloud Challenge: "Noisy Neighbor" Phenomenon[^3]

- Monitoring: Track metrics like memory bandwidth utilization and LLC usage per workload to identify potential interference.
- Resource Isolation: Cloud providers and virtualization platforms often offer mechanisms for better resource isolation (e.g., dedicated instances, CPU pinning, memory bandwidth allocation controls). Production datacenters may use throttling or "busy" flags to protect higher-priority services.
- Intelligent Scheduling: Implement pod or VM scheduling policies that co-locate compatible workloads or isolate latency-sensitive applications from known noisy ones.
- Hardware-Assisted Mitigation: Technologies like Intel Resource Director Technology (RDT) can provide finer-grained control over shared resources like cache and memory bandwidth, potentially allowing for throttling of offending threads.
- Tenant Isolation Paradigms: System designs focusing on "noisy neighbor mitigation" aim to ensure predictable performance for tenants despite shared infrastructure. This "noisy neighbor" problem underscores a fundamental challenge in shared environments: balancing the cost-efficiency of resource pooling with the performance predictability required by latency-sensitive applications. Effective mitigation often requires a combination of platform-level isolation capabilities and application-level awareness and observability.

#### Network Challenge: Congestion[^1], Jitter[^2], and Packet Loss

- Implement robust congestion control mechanisms
- Tune network device buffers
- Design balanced network topologies to minimize delays
- Configure QoS policies on network devices to prioritize latency-sensitive traffic over bulk traffic
- Use jitter buffers at the receiving end of a connection (e.g., in VoIP clients or streaming applications). These buffers collect incoming packets and release them at a steady rate, smoothing out variations in arrival times. This can introduce a small amount of constant latency but significantly reduce the perceived impact of jitter
- Ensure sufficient network bandwidth is available to handle peak loads without causing congestion
  

#### Hardware Challenge: Silent Hardware Degradation (Healthy but degraded performance)

Identify
- Tail Latency Monitoring: Consistently high P99 or P99.9 latencies, especially if localized to specific nodes or services, can be an indicator of underlying hardware degradation.
- Proactive Health Monitoring: Implement comprehensive hardware health monitoring that goes beyond simple up/down checks, looking for performance counters, error rates, and other indicators of degradation.

Resolve
- Isolation and Replacement: Once a fail-slow component is identified, it should be isolated from production traffic and replaced
- Resilient System Design: Architect systems to be resilient to such partial failures, for example, by having sufficient redundancy and mechanisms to route traffic away from poorly performing nodes

#### Solutions

- Query Timeouts: Set query execution time limits to prevent long-running queries from affecting system performance.
- Background Preloading: Warm up cache periodically to avoid first-access delays.
- Connection Queuing: Use bounded queues or backpressure mechanisms to avoid overloading the database.
- Isolate Heavy Queries: Run batch reports or analytics workloads separately from real-time traffic.


[^1]: Congestion & Transient Oversubscription: Short bursts of synchronized traffic, common in distributed systems, can temporarily overwhelm network links or device buffers, leading to packet queuing and delays for some requests.
[^2]: Jitter & Variability: Jitter refers to the variation in packet arrival times. Inconsistent packet delays, caused by factors like faulty network equipment, Quality of Service (QoS) misconfigurations, wireless interference, or microbursts overwhelming buffers, can unpredictably increase tail latency. Even if average latency is low, high jitter means some packets arrive much later than others, impacting P99
[^3]: In multi-tenant cloud environments or shared hosting platforms, workloads co-exist on the same physical hardware. A "noisy neighbor" is a workload that excessively consumes shared resources such as CPU cycles, memory bandwidth, or Last Level Cache (LLC), thereby degrading the performance of other co-located workloads. This interference can lead to significant spikes in P95 and P99 latency for affected applications. For example, Google has reported 5x to 14x increases in P95/P99 latency due to memory subsystem interference
