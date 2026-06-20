# Hazelcast Distributed Cache Integration

## 1. Executive Summary
A Spring Boot microservices-based ERP/LMS platform experienced recurring latency spikes and rising database pressure caused by repeated reads of low-volatility reference and configuration data. The engineering team introduced Hazelcast as a distributed in-memory cache layer shared across services. The solution reduced repetitive database calls, improved response times for read-heavy workflows, and created a scalable caching foundation aligned with the platform’s horizontal growth model.

## 2. Business Context
The platform supports academic operations such as admissions, fee workflows, enrollment, timetable access, and compliance reporting across multiple institutions. Core API workflows repeatedly consume the same lookup datasets (academic session metadata, fee rules, policy flags, branch settings, and role mappings). During admission windows and exam cycles, concurrent traffic rises sharply, making database efficiency directly tied to user experience, operational cost, and SLA reliability.

## 3. Existing System Architecture
Before caching, the architecture consisted of:

- Spring Boot microservices for ERP and LMS domains.
- REST-based inter-service communication.
- Shared relational database clusters for transactional and reference data.
- Stateless service instances deployed across multiple pods/VMs.
- Standard ORM/repository access patterns at service level.

The services were horizontally scalable, but read amplification on shared reference tables became a central bottleneck under peak load.

## 4. Problem Statement
### Repeated DB Queries
Frequently accessed but infrequently changing datasets were fetched repeatedly by multiple service instances, often within the same request chains.

### Increased Latency
Each extra round trip to the database added network and query overhead, creating cumulative latency in user-facing APIs.

### Database Load Concerns
Reference-data traffic consumed a meaningful portion of DB connection pool capacity and IOPS, reducing headroom for critical transactional workloads.

### Scalability Limitations
Adding more service instances increased total read pressure on the same DB layer, weakening horizontal scale efficiency and increasing infrastructure cost.

## 5. Why Caching Was Needed
Caching was required to decouple read-heavy reference access from the transactional database path. The objective was to:

- Offload repeated reads from the primary database.
- Lower p95/p99 response latency for common endpoints.
- Improve resilience during demand spikes.
- Enable service scale-out without linear database load growth.

## 6. Alternative Solutions Evaluated

### Option A: No Cache
**Pros**
- No additional infrastructure or operational overhead.
- Strong consistency from direct DB reads.

**Cons**
- Persistent repeated queries for static or slowly changing data.
- Higher latency and DB contention under peak traffic.
- Poor cost-efficiency as load grows.

**Scalability Implications**
Scale is bounded by database throughput; service-level horizontal scaling translates into disproportionate DB pressure.

### Option B: Local In-Memory Cache (Per Service Instance)
**Pros**
- Fastest local read latency.
- Easy initial implementation.
- No external cache dependency.

**Cons**
- Data duplication across instances.
- Cache inconsistency risk between nodes.
- Cold-start penalties on deployments/autoscaling.
- Difficult global invalidation.

**Scalability Implications**
Works at small scale but becomes operationally fragile as instance count and deployment frequency increase.

### Option C: Redis
**Pros**
- Mature ecosystem and strong performance.
- Rich data structures and observability tooling.
- Widely adopted operational patterns.

**Cons**
- Introduces an additional standalone cache tier.
- Potential architectural disconnect from JVM-native service stack.
- Additional integration and failover design effort for cluster behavior.

**Scalability Implications**
Scales well as a centralized/distributed cache, but requires disciplined capacity planning and high-availability topology management.

### Option D: Hazelcast
**Pros**
- Native JVM/Spring integration.
- Distributed in-memory data grid with partitioning and replication.
- Embedded or client-server deployment flexibility.
- Supports distributed data structures and event-driven invalidation patterns.

**Cons**
- Cluster management complexity compared with simple local cache.
- Requires careful memory governance and eviction strategy.
- Additional operational knowledge for partition health and rebalancing.

**Scalability Implications**
Horizontal cache scaling aligns well with microservice growth; data and load distribution improve read scalability without overloading the database.

## 7. Why Hazelcast Was Selected
Hazelcast was selected because it matched both technical and organizational constraints:

- Strong fit for Spring Boot and Java-based services.
- Distributed architecture with built-in partitioning/replication for high availability.
- Lower integration friction versus introducing an entirely separate non-JVM-centric caching model.
- Ability to evolve from basic key-value caching toward broader in-memory data grid use cases.

## 8. Hazelcast Architecture Overview
### Cluster Nodes
Multiple Hazelcast-enabled service nodes formed a logical cluster, with each node contributing memory and compute capacity.

### Data Distribution
Cached entries were partitioned across cluster members, balancing ownership and reducing hotspot concentration.

### Replication
Backup replicas were configured for partitions to ensure cached data survivability on node failure.

### Failover
When a node dropped, partition ownership was reassigned and backups promoted, maintaining cache availability with minimal disruption.

### In-Memory Data Grid Concepts
The implementation used distributed maps as shared cache regions and leveraged cluster-wide data locality, partition awareness, and distributed coordination semantics.

## 9. Integration Approach
### Spring Boot Integration
Hazelcast integration was implemented via Spring Cache abstraction and Hazelcast-backed cache manager configuration, minimizing code-level disruption.

### Cache Configuration
Cache regions were defined with TTL, max-idle, and eviction policies tuned to data volatility and memory budget.

### Cache Regions
Separate named regions were established to isolate workloads and policy needs (for example, configuration cache vs. institutional metadata cache).

### Cacheable Data Categories
Initial caching scope prioritized low-churn, high-read data:

- Master/reference tables
- Tenant/institution configuration
- Feature and policy flags
- Read-mostly lookup mappings used across request paths

## 10. Engineering Tradeoffs
### Memory Consumption
Caching improved speed but increased RAM demand. Region limits and eviction policies were required to prevent memory pressure.

### Cache Consistency
Short-lived staleness was accepted for selected categories to gain performance. Strong consistency remained database-backed for transactional paths.

### Cache Invalidation
Invalidation strategy combined TTL expiry with targeted eviction on configuration updates to balance correctness and operational simplicity.

### Cluster Complexity
Distributed caching introduced new operational concerns: membership stability, split-brain protection strategy, monitoring, and capacity forecasting.

## 11. Performance Benefits
### Reduced Database Load
Reference-data queries shifted significantly from the primary DB to cache hits, reducing connection pool saturation risk.

### Faster Response Times
High-frequency read endpoints observed meaningful improvements in average and tail latency due to reduced DB round trips.

### Better Horizontal Scalability
As service instances scaled out, cache distribution absorbed additional read demand, improving scalability efficiency versus DB-only reads.

## 12. Lessons Learned
- Start with narrow, high-impact cache domains rather than broad uncontrolled caching.
- Define ownership for cache invalidation early to avoid stale-data incidents.
- Instrument hit ratio, eviction rate, and per-region memory usage from day one.
- Align TTL values to business tolerance for staleness, not arbitrary defaults.
- Treat distributed cache rollout as an architectural change, not only a performance tweak.

## 13. Future Improvements
- Introduce event-driven invalidation from domain update events for finer consistency.
- Expand observability with cache-region SLO dashboards and anomaly alerts.
- Evaluate near-cache patterns for ultra-low-latency read paths where appropriate.
- Add resilience drills for node failure and partition rebalancing scenarios.
- Reassess cache key design and serialization formats for memory efficiency.

## 14. Key Backend Engineering Takeaways
- Distributed caching is most valuable when traffic is read-heavy and data volatility is low to moderate.
- Platform scalability depends on reducing shared database amplification, not only scaling stateless services.
- Technology selection should balance ecosystem fit, operational maturity, and long-term architecture direction.
- Caching strategy must include consistency and invalidation design as first-class engineering concerns.
- Performance gains are durable only when paired with observability and operational discipline.
