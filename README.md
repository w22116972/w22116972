# Tech Stack

### Distributed systems
- Implemented retry and timeout strategies with exponential backoff to ensure service reliability in the face of transient failures, resulting in fault tolerance
- Minimized response times by introducing multi-tier caching for local Caffeine and remote Redis for frequently accessed data in high-traffic endpoints, resulting in improved latency
- Improved system observability and reliability by implementing the Prometheus stack to collect metrics, visualize performance, and proactively detect anomalies in distributed services

### Graph API Development
- **RESTful API** and **GraphQL** by **Spring** framework
- Implement text auto-complete, search, query to **graph database Neo4j**

### Java Experiences
- Diagnosed and resolved \textbf{OutOfMemoryError} issues by analyzing heap dumps, optimizing object allocation, and fine-tuning JVM parameters, improving system stability under high-load conditions
  - Resolved excessive I/O threads retention issue by releasing driver resources cached in memory based on eviction policy, resulting in reducing heap memory usage
  - Optimized memory usage by caching results of Java internal methods, reducing pressure from frequent `StringBuilder` allocations
  - Resolved 10M+ nodes returned from database by using pagination and limit
- Tuned JVM configurations (heap size, GC, thread pool) to reduce latency and prevent memory leaks in long-running backend services

### Cloud Infrastructure
- Led cloud migration by containerizing legacy applications with Docker and deploying them to **Kubernetes** (**AWS EKS**), using **Helm** for application deployment and AWS CDK to provision cloud infrastructure — enabling scalable, portable, and fully automated environments
- Optimized Docker image size by applying multi-stage builds, minimizing dependencies, and selecting minimal, vulnerability-free base images — reducing image size by over 60%, accelerating CI/CD pipeline performance, and improving container security in production environments
- Integrated CI/CD pipelines into cloud-deployed workloads using GitLab and Helm

### Code Quality
- Refactored legacy code by applying SOLID principles and design patterns to improve code maintainability
- Applied Oracle Secure Coding Guidelines by enforcing safe class design, input validation, overflow and floating-point error handling, robust exception handling, secure logging, and defensive copying, resulting in improved codebase security
  - [Oracle Secure Coding Guideline Note](https://github.com/w22116972/wiki/blob/main/docs/best-practices/Oracle%20Secure%20Coding%20Guidelines%20for%20Java.md)
- Strengthened security across codebase, container images, and CI/CD pipelines by leveraging Aqua Trivy to identify and fix Java vulnerable dependencies, insecure Dockerfile patterns, vulnerable base images, Helm chart misconfigurations.
  - TODO: dockerfile non-root solution and why
  - Adopt Amazon Corretto to replace Eclipse Temurin on alpine image
  
---

### Technical Writing

[Leetcode Pattern for Interview](https://github.com/w22116972/coding-interview-pattern)

[Google Technical Writing](https://medium.com/@w22116972/google-technical-writing-21a89129bfbc)

[Implementing Strategy Pattern on Spring Framework](https://medium.com/@w22116972/implementing-strategy-pattern-on-spring-framework-1a9760831ee5)
