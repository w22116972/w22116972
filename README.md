# Tech Stack

Specializing in **Java performance engineering** and **cloud infrastructure operations**. Proficient in resolving high-load performance bottlenecks, designing scalable distributed graph APIs, and delivering secure, reliable cloud-native applications. Hands-on experience in containerized deployments, infrastructure as code, and observability solutions for production-grade systems.

### Java Performance Optimization & Tuning
- Diagnosed and resolved OutOfMemoryError issues by analyzing heap dumps, optimizing object allocation, and fine-tuning JVM parameters, improving system stability under high-load conditions
  - Resolved excessive I/O threads retention issue by releasing driver resources cached in memory based on eviction policy
  - Reducing pressure from frequent Java internal `StringBuilder` allocations by caching results of Java internal methods
  - Resolved 10M+ nodes returned from database by using pagination and limit
- Tuned JVM configurations (heap size, GC, thread pool) to reduce latency and prevent memory leaks in long-running backend services

### Scalable Distributed Graph API Architecture
- Implemented fault tolerance by retry and timeout strategies with exponential backoff
- Reduced response times by introducing multi-tier caching for local Caffeine and remote Redis for frequently accessed data in high-traffic endpoints
- Implemented multi-tenant RESTful API and GraphQL of graph database Neo4j by Spring framework
  - Implemented queries on finding nodes, links, conditional search,   
  - Implemented search keywords on nodes, links, labels, node name, link name
  - Implemented text auto-complete when search
- Resolved large data nodes by using mega node as group to represent large nodes

### Cloud-native Infrastructure Engineering
- Led cloud migration by containerizing legacy applications with Docker and deploying them to Kubernetes (AWS EKS), using Helm for application deployment and AWS CDK to provision cloud infrastructure, enabling scalable, portable, and fully automated environments
- Optimized Docker images by applying multi-stage builds, minimizing dependencies, and selecting minimal, vulnerability-free base images. This reduced image size by over 60%, accelerated CI/CD pipeline performance, and enhanced container security in production environments
- Integrated CI/CD pipelines into cloud-deployed workloads using GitLab and Helm
- Improved system observability and reliability by implementing the Prometheus stack to collect metrics, visualize performance, and proactively detect anomalies in distributed services

### Code Quality

#### Code Secuirty
- Applied Oracle Secure Coding Guidelines[^10] on enforcing safe class design, input validation, overflow and floating-point error handling, robust exception handling, secure logging, and defensive copying
- Used Aqua Trivy to identify and fix Java vulnerable dependencies, insecure Dockerfile patterns[^11], vulnerable base images[^12], Helm chart misconfigurations

#### Maintainability
- Applied SOLID principle and design patterns to refactor legacy code

#### Reliability
- Used 

- Improved code maintainability by refactoring legacy code based on SOLID principles and design patterns
- Improved codebase security by applying Oracle Secure Coding Guidelines[^10] on enforcing safe class design, input validation, overflow and floating-point error handling, robust exception handling, secure logging, and defensive copying
- Strengthened security across codebase, container images, and CI/CD pipelines by leveraging Aqua Trivy to identify and fix Java vulnerable dependencies, insecure Dockerfile patterns[^11], vulnerable base images[^12], Helm chart misconfigurations

## Projects

### High traffic reservation system
> Designed a high-concurrency inventory reservation system handling over 100,000 QPS

- use Redis + Lua script to handle reservation atomically
- use Virtual thread to handle high traffic

### 1 billion rows challenge
> Compute 1 billion rows for grouped operations by java application less than 2 seconds

- use `mmap`


---

# Technical Writing

## Performance

- [Percentile-Based Performance Optimization](https://github.com/w22116972/wiki/blob/main/docs/performance-engineering/Percentile-Based%20Performance%20Optimization.md)

# Solutions

- Deduplicate
- Distributed transaction

## Devops, Cloud

- [Use service account to replace hardcode access key](https://github.com/w22116972/wiki/blob/main/docs/devops/Use%20IAM%20roles%20for%20Service%20Account.md)
- [Use CI/CD variables to replace declaring secret](https://github.com/w22116972/wiki/blob/main/docs/devops/Secret%20in%20k8s.md)
- [Docker Best Practices](https://github.com/w22116972/wiki/blob/main/docs/devops/Dockerfile%20Best%20Practices.md)



## Java, Spring

- [Implementing Strategy Pattern on Spring Framework](https://medium.com/@w22116972/implementing-strategy-pattern-on-spring-framework-1a9760831ee5)
- [Oracle Secure Coding Guideline Note](https://github.com/w22116972/wiki/blob/main/docs/java/Oracle%20Secure%20Coding%20Guidelines%20for%20Java.md)

## Classic Paper

- [David A. Patterson. 2004. Latency lags bandwith](https://github.com/w22116972/wiki/blob/main/docs/classic-paper/Latency%20lags%20bandwidth.md)

## Style Guide

- [Google Technical Writing](https://medium.com/@w22116972/google-technical-writing-21a89129bfbc)
- [Json Style Guide](https://github.com/w22116972/wiki/blob/main/docs/style-guide/JSON%20Style%20Guide.md)

## Case Study

- [Why Discord is switching from Go to Rust by Discord](https://github.com/w22116972/wiki/blob/main/docs/case-study/why-discord-is-switching-from-go-to-rust.md)


#### Miscellaneous

- [Download Resume](./Ander%20Wang.pdf)
- [Leetcode Pattern for Interview](https://github.com/w22116972/coding-interview-pattern)


[^10]: [Oracle Secure Coding Guideline Note](https://github.com/w22116972/wiki/blob/main/docs/java/Oracle%20Secure%20Coding%20Guidelines%20for%20Java.md)

[^11]: [Dockerfile Best Practices](https://github.com/w22116972/wiki/blob/main/docs/devops/Dockerfile%20Best%20Practices.md)

[^12]: Adopt `amazoncorretto:21.0.6-alpine3.21` to replace `eclipse-temurin:21.0.6_7-jdk-alpine-3.21` on alpine image for free vulnerability
