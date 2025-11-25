# Apache Kafka Diskless KIPs summary and status update

## Why Diskless

The common goal of the Diskless KIPs is to save cross-AZ traffic cost.

Typical deployment of Kafka on the cloud is to deploy brokers onto several availability zones to reduce the chance of interruption by zone-wide outage. However, Kafka's traditional architecture is based on broker disks for persistence and replication. This creates cross-AZ traffic, which cloud providers charge for—typically $0.01-0.02 per GB on AWS and GCP.

Kafka's tiered storage (KIP-405) takes a step to reduce the cross-AZ traffic on older log data, aka the rolled segments, by moving them to cloud storage and relying on the cloud storage's built-in cross-AZ replication. This cost saving is great for batch historical workloads that require less than real-time latency.

Real-time workloads on latest events, however, mostly only touch the active segments, which still rely on the broker disk in the traditional architecture to replicate across zones. This presents an opportunity for further cost cutting, though it may come with some performance impact.

For cloud providers that do not charge cross-AZ traffic (like Azure), the other goal of diskless becomes important, which is to simplify operations and enable cloud-native scalability.

## Diskless KIPs summary

The Kafka community currently has three concurrent proposals addressing the same challenge, creating an unprecedented situation where discussions have moved gradually as of November 2025. Each KIP represents a fundamentally different architectural philosophy.

### KIP-1150: Diskless Topics

**Main Idea:** Replace broker local disks with object storage (like S3) as the primary durable storage for Kafka topics.

**Architecture Changes:**
- **Leaderless design** - all brokers can interact with all partitions (revolutionary change from traditional leader-follower model)
- Data stored solely in object storage, not on broker disks
- Batch-based write model: producers send data to any broker → broker accumulates requests in buffer → uploads complete batches to object storage → Batch Coordinator assigns offsets
- Pluggable Batch Coordinator (KIP-1164) for metadata management
- Delegated replication to object storage (leveraging S3's built-in cross-AZ replication)

**Implementation:** Open source prototype at https://github.com/aiven/inkless (led by Aiven team)

### KIP-1176: Tiered Storage for Active Log Segment

**Main Idea:** Extend existing KIP-405 (tiered storage for closed segments) to also upload active log segments to fast object storage.

**Architecture Changes:**
- **Incremental evolution** of existing tiered storage - preserves leader-follower model
- Creates three-tier storage: Local disk → Fast object store (S3 Express One Zone) → Traditional S3
- Leader writes to both local disk AND fast object storage simultaneously
- Followers fetch active segments from fast object storage (same AZ) instead of cross-AZ replication from leader
- Background tasks handle uploads via `RLMWalCombinerTask`
- Reuses existing page cache for performance

**Implementation:** Closed source (based on proprietary work at Slack/Salesforce)

### KIP-1183: Unified Shared Storage

**Main Idea:** Abstract the storage layer to support both traditional local disk and shared storage simultaneously.

**Architecture Changes:**
- Two-step approach: 1) Abstract log layer with `AbstractLog` and `AbstractLogSegment` classes, 2) Define pluggable `Stream` API
- Enables Kafka to transform from shared-nothing to shared storage architecture
- Supports both architectures simultaneously for gradual migration
- Maintains leader-based architecture (unlike KIP-1150)
- Flexible deployment on various storage backends (S3, HDFS, NFS, Ceph, MinIO, CubeFS)

**Implementation:** Primary implementation at https://github.com/AutoMQ/automq (AutoMQ/Alibaba)


A table summary of KIPs along several factors important to our customers, such as cost saving, and architecture pros and cons along scalability, availability, efforts to change.

| KIPs     | Cross-AZ Traffic Reduction | Cost Savings | Performance Impact | Scalability | Availability | Implementation Effort | Status (Nov 2025) |
|----------|---------------------------|--------------|-------------------|-------------|--------------|---------------------|-------------------|
| **KIP-1150** | **Complete** - Eliminates all cross-AZ replication (leaderless design) | **Maximum** - No cross-AZ costs for replication | **High latency**: P50 ~500ms, P99 ~1-2s (vs single-digit ms traditional) | **Best** - Stateless brokers, true cloud-native elasticity, data/metadata separation | **Strong** - Leverages S3's 11 9's durability and built-in cross-AZ replication | **High** - Revolutionary change, multiple sub-KIPs, but clean design | Under discussion, concerns about transaction/queue support |
| **KIP-1176** | **Partial** - Only follower replication path (producer→leader and consumer traffic unchanged) | **Moderate** - 43% overall cost reduction documented | **Low latency in some durability/storage settings** - Maintains single-digit ms for acks=1, near-traditional for acks=-1 with fast storage | **Limited** - Still broker-centric, no cloud-native elasticity benefits | **Weak** - AZ failure creates recovery challenges, no hot standby for active segments | **Medium-High** - Incremental changes to existing code, but complexity may grow | Under discussion, availability concerns raised |
| **KIP-1183** | **Moderate** - Eliminates cross-AZ replication with shared storage (RF=1) | **Moderate** - Similar to KIP-1150 but less optimized | **Not quantified** - Depends on Stream implementation quality | **Good** - Stateless brokers enable scaling, but not as optimized as KIP-1150 | **Concerns** - Failover latency with RF=1 (1-2 sec documented), no hot standby | **High** - Large plugin development burden, unclear Stream interface design | Under discussion, Stream interface design unclear |

**Implementation References:**
- KIP-1150: https://github.com/aiven/inkless (Aiven, open source)
- KIP-1176: Closed source (Slack/Salesforce proprietary)
- KIP-1183: https://github.com/AutoMQ/automq (AutoMQ/Alibaba)
 
## Diskless KIPs status update

The Kafka community faces an unprecedented challenge with three concurrent proposals all addressing cross-AZ replication costs. Discussions have evoloved gradually as of November 2025, with authors and the community taking time to proceed to consider pros and cons of the proposals.

### Timeline of Major Discussion Points

**April 16, 2025** - KIP-1150 Discussion Initiated
- Josep Prat (Aiven) introduces "diskless topics" concept
- Proposes leaderless architecture using object storage as primary storage
- Follows KRaft approach: meta-KIP with implementation sub-KIPs (KIP-1163, KIP-1164, KIP-1181)

**April 17-19, 2025** - Initial Community Feedback
- Questions about "zero local disk" claims
- Clarification: brokers don't store user data, but do store metadata and caching
- Concerns about multi-region support and latency expectations

**May 6, 2025** - KIP-1176 Introduced
- Henry Haiying Cai (Slack/Salesforce) presents "incremental evolution" approach
- Emphasizes simpler design building on existing KIP-405
- Claims 43% cost reduction while maintaining low latency performance in some durability/storage settings

**May-November 2025** - Discussion continued
- Jun Rao (Confluent) raises questions about KIP-1150's latency vs KIP-1176's simpler approach
- Community concern about three concurrent proposals creating review burden
- No clear consensus emerges on which direction to pursue

### Current Status by KIP

**KIP-1150: Diskless Topics**
- **Status:** Under discussion (continuing since May 2025)
- **Key Concerns:**
  - Expected latency ~100-500ms for writes (vs single-digit ms traditional)
  - No concrete design yet for transactions and queues support
  - Is the significant architectural change justified?
- **Open Questions:**
  - Which use cases can tolerate higher latency?
  - When will feature parity with traditional Kafka be achieved?

**KIP-1176: Tiered Storage for Active Segments**
- **Status:** Under discussion (last comment June 2025)
- **Key Concerns:**
  - Weak availability story for AZ failures
  - Only saves cross-AZ costs on follower replication path
  - Implementation complexity may grow beyond initial "small effort" estimate
- **Open Questions:**
  - How to handle AZ outages with S3 Express One Zone storage?
  - Multi-cloud deployment model for GCP/Azure unclear

**KIP-1183: Unified Shared Storage**
- **Status:** Under discussion (continuing since May 2025)
- **Key Concerns:**
  - Stream interface design remains unclear
  - Large plugin development burden
  - Potential multiple AbstractLog implementations
- **Open Questions:**
  - How does this relate to KIP-1150 and KIP-1176?
  - Should Kafka support both local disk and shared storage long-term?
  - Leaderless architecture vs abstraction layer approach?

### The Path Forward Dilemma

As documented in the [community summary page](https://cwiki.apache.org/confluence/display/KAFKA/The+Path+Forward+for+Saving+Cross-AZ+Replication+Costs+KIPs), the community must resolve:

1. **Philosophical Questions:**
   - Should Kafka embrace cloud-native shared storage architecture?
   - Revolutionary change vs incremental evolution?
   - Unified community approach vs vendor-specific extensions?

2. **Technical Trade-offs:**
   - Performance (KIP-1176) vs Cost Savings (KIP-1150)
   - Clean architecture (KIP-1150) vs Implementation effort (KIP-1176)
   - Flexibility (KIP-1183) vs Complexity

3. **Strategic Concerns:**
   - Risk of Apache Kafka losing market share to protocol-compatible alternatives already using object storage
   - Need to balance innovation with backward compatibility
   - Community capacity to review and maintain multiple approaches

## Conclusion: What This Means for Kafka Users

The three diskless KIPs represent different visions for Kafka's cloud-native future:

**KIP-1150** offers the most radical transformation—a leaderless, cloud-native architecture that maximizes cost savings but comes with higher latency and significant implementation complexity. This approach is ideal for use cases where cost optimization is paramount and 100-500ms latency is acceptable (analytics, logging, monitoring).

**KIP-1176** takes an incremental approach that preserves Kafka's performance characteristics while delivering moderate cost savings (43% documented). However, its weak availability story and limited scope (only follower replication) raise questions about whether it goes far enough.

**KIP-1183** attempts to bridge both worlds with a pluggable abstraction layer, but the unclear Stream interface design and high plugin development burden have led to community skepticism about multiple implementations.

### For Kafka Users

If you're currently evaluating Kafka for cloud deployments:

1. **Short term (2025-2026):** Continue with traditional Kafka or existing tiered storage (KIP-405). The diskless discussion is unlikely to resolve quickly.

2. **Medium term (2026-2027):** Monitor which KIP gains traction. Early indicators suggest the community may favor either KIP-1150's comprehensive approach or a simplified version of KIP-1176.

3. **Long term:** Kafka's architecture will likely evolve toward cloud-native patterns. Competitors like WarpStream and AutoMQ have already proven object storage viability, pressuring Apache Kafka to adapt.

The current stalemate reflects a healthy but challenging debate about Kafka's future. The community must balance backward compatibility with innovation, performance with cost, and unified direction with vendor flexibility. Whatever path emerges will shape Kafka's relevance in cloud-native architectures for years to come.

## Further reading

Links to several sources that are important to keep track of.
- [The community summary page](https://cwiki.apache.org/confluence/display/KAFKA/The+Path+Forward+for+Saving+Cross-AZ+Replication+Costs+KIPs)
- [KIP-1150](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1150%3A+Diskless+Topics)
- [KIP-1176](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1176%3A+Tiered+Storage+for+Active+Log+Segment)
- [KIP-1183](https://cwiki.apache.org/confluence/display/KAFKA/KIP-1183:+Unified+Shared+Storage)
