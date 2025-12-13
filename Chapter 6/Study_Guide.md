Study Guide: Database Partitioning (Sharding)

📝 Key Concepts & Revision Notes

Core Concepts

Concept	Definition
Partitioning (Sharding)	The process of intentionally breaking a large database into smaller, more manageable parts, called partitions, to improve scalability by distributing data and query load across multiple nodes.
Skew	An unfair distribution of data or query load across partitions, where some partitions have significantly more data or receive more queries than others, reducing the effectiveness of partitioning.
Hot Spot	A partition that experiences a disproportionately high load, becoming a bottleneck for the entire system.
Key Range Partitioning	A method of partitioning where each partition is assigned a continuous range of keys, allowing for efficient range scans but risking hot spots if access patterns target adjacent keys.
Hash Partitioning	A method where a hash function is applied to each key to determine its partition, which helps distribute data more evenly but loses the inherent ordering of keys, making range queries inefficient.
Rebalancing	The process of moving data and request load from one node to another within a cluster, typically triggered by adding or removing nodes, to ensure the load remains evenly distributed.
Secondary Index	An index based on a non-primary key field (e.g., color, make) used to efficiently search for records with a particular value, which presents unique challenges in a partitioned environment.

Hierarchical Outline

* 1.0 Introduction to Partitioning
  * 1.1 Definition: Breaking up data into partitions (shards).
  * 1.2 Primary Goal: Scalability (distributing data and query load across a shared-nothing cluster).
  * 1.3 Terminology:
    * Partition (Standard term)
    * Shard (MongoDB, Elasticsearch, SolrCloud)
    * Region (HBase)
    * Tablet (Bigtable)
    * vnode (Cassandra, Riak)
    * vBucket (Couchbase)
  * 1.4 Combination with Replication: Each partition is replicated for fault tolerance, with leaders and followers distributed across nodes.
* 2.0 Partitioning Key-Value Data
  * 2.1 Goal: Spread data and query load evenly to avoid skew and hot spots.
  * 2.2 Partitioning by Key Range:
    * Mechanism: Assigns a continuous range of keys to each partition.
    * Advantages: Efficient range scans are possible.
    * Disadvantages: Can lead to hot spots (e.g., using timestamps as keys for writes).
    * Example Systems: Bigtable, HBase, RethinkDB.
  * 2.3 Partitioning by Hash of Key:
    * Mechanism: A hash function determines the partition for a given key.
    * Advantages: Distributes skewed data uniformly, reducing hot spots.
    * Disadvantages: Loses key ordering, making range queries on the primary key inefficient (requires scatter/gather).
    * Compromise (Cassandra): Use a compound primary key; hash the first part for partitioning and use subsequent parts for sorting within the partition.
  * 2.4 Mitigating Skewed Workloads (Hot Spots):
    * Problem: A single, extremely popular key (e.g., a celebrity's user ID) directs all traffic to one partition.
    * Application-level solution: Append a random suffix to the hot key to split writes across multiple keys and partitions, but this complicates reads.
* 3.0 Partitioning and Secondary Indexes
  * 3.1 The Challenge: Secondary indexes don't map neatly to the primary key's partitions.
  * 3.2 Document-Partitioned Indexes (Local Indexes):
    * Mechanism: Each partition maintains its own secondary indexes, covering only the documents within that partition.
    * Write Path: Simple; only a single partition is updated.
    * Read Path: Complex and expensive; queries must be sent to all partitions ("scatter/gather").
    * Example Systems: MongoDB, Riak, Cassandra, Elasticsearch.
  * 3.3 Term-Partitioned Indexes (Global Indexes):
    * Mechanism: The secondary index is itself partitioned, separate from the primary key, based on the indexed term (value).
    * Write Path: Slower and more complicated; a single document write may require updating multiple index partitions. Often asynchronous.
    * Read Path: More efficient; a client query can be routed directly to the single partition containing the desired term.
* 4.0 Rebalancing Partitions
  * 4.1 Rationale: Needed when nodes are added/removed or a machine fails.
  * 4.2 Core Requirements:
    * Fair distribution of load after rebalancing.
    * Database remains operational during the process.
    * Minimal data movement to ensure speed and low overhead.
  * 4.3 Rebalancing Strategies:
    * Problematic Approach: hash mod N. Changing the number of nodes (N) requires most keys to be moved.
    * Fixed Number of Partitions: Create far more partitions than nodes initially. When a new node is added, it "steals" a few partitions from each existing node.
    * Dynamic Partitioning: Partitions are split when they grow too large and merged when they shrink. The number of partitions adapts to data volume. Used by HBase and RethinkDB.
    * Partitions Proportional to Nodes: The number of partitions per node is fixed. Adding a node causes existing partitions to be split. Used by Cassandra and Ketama.
  * 4.4 Operations: Rebalancing can be fully automatic or require manual administrator approval, with the latter preventing unpredictable performance degradation from expensive rebalancing operations.
* 5.0 Request Routing
  * 5.1 The Problem (Service Discovery): How does a client know which node owns the partition for its request?
  * 5.2 Routing Approaches:
    1. Any Node Gateway: Client connects to any node, which either handles the request or forwards it to the correct node.
    2. Routing Tier (Proxy): A dedicated layer acts as a partition-aware load balancer, routing client requests to the appropriate node.
    3. Client-Aware: The client itself knows the partition-to-node mapping and connects directly.
  * 5.3 Discovering Topology:
    * Coordination Service: Systems like ZooKeeper maintain the authoritative mapping of partitions to nodes (used by HBase, SolrCloud, Kafka).
    * Gossip Protocol: Nodes communicate changes in cluster state among themselves (used by Cassandra, Riak).

💡 In-Depth Explanation & Technical Deep Dive

Mechanism/Process Breakdown: Rebalancing with a Fixed Number of Partitions

This strategy is a common and effective way to manage cluster growth and shrinkage. It decouples the number of partitions from the number of physical nodes, providing operational simplicity. The process for adding a new node is as follows:

1. Initial Setup: The database is configured with a large, fixed number of partitions (e.g., 1,000) from the outset, significantly more than the initial number of nodes (e.g., 4). These partitions are distributed as evenly as possible among the available nodes. For example, with 4 nodes and 1,000 partitions, each node would hold approximately 250 partitions.
2. Node Addition: A new node (e.g., Node 5) is added to the cluster. This new node is initially empty and is not serving any traffic.
3. Partition "Stealing": To rebalance the load, the new node needs to take ownership of some partitions from the existing nodes. The cluster management logic determines that the new node will "steal" a certain number of partitions from each of the existing nodes. For example, it might take 50 partitions from Node 1, 50 from Node 2, and so on, until the partitions are once again fairly distributed across all 5 nodes.
4. Data Transfer: The actual data for the stolen partitions is copied over the network from the old nodes to the new node. This is the most resource-intensive part of the process.
5. Routing Update: While the data transfer is in progress, the old partition assignment remains active. Any reads and writes for the partitions being moved are still directed to their original nodes.
6. Cutover: Once the data transfer for a partition is complete and the new node is ready, the cluster's routing information is updated. This update, often managed by a coordination service like ZooKeeper or through a gossip protocol, redirects all future reads and writes for the moved partitions to the new node.
7. Completion: The process is complete when the new node has taken ownership of its share of partitions and the old nodes have relinquished them. The cluster is now in a new, balanced state with the load spread across all five nodes.

This mechanism avoids the hash mod N problem entirely, as the assignment of a key to a specific partition never changes; only the assignment of the partition to a physical node changes.

Reference: The process described is illustrated in the source document's Figure 6-6. Adding a new node to a database cluster with multiple partitions per node.

The "Why": Why Hashing a Key is a Superior Partitioning Strategy for Load Balancing

Hashing a key to determine its partition solves a critical problem inherent in simpler distribution methods like key-range partitioning or the naive hash mod N approach: the prevention of skew and hot spots, especially during rebalancing.

* Problem Solved by Hashing (vs. Key Range): Key-range partitioning is vulnerable to access patterns. If an application frequently writes data with sequential keys (like timestamps or auto-incrementing IDs), all write traffic will be directed to a single partition—the one containing the end of the range. This creates a severe write hot spot. A good hash function takes skewed input (like sequential keys) and produces uniformly distributed output. Even keys that are very close together (e.g., "2024-10-26-12:00:01" and "2024-10-26-12:00:02") will be mapped to completely different, seemingly random hash values, placing them in different partitions and spreading the write load evenly across the cluster.
* Problem Solved by Hashing with Ranges (vs. mod N): The simplest way to use a hash might seem to be hash(key) mod N, where N is the number of nodes. This works to distribute keys evenly at first. However, its fundamental flaw is that it tightly couples the key's location to the total number of nodes.
  * The Flaw of mod N: If the number of nodes N changes, the result of the mod operation changes for the vast majority of keys. For example, if a key with hash(key) = 123456 is on node 6 (123456 mod 10 = 6), adding a new node to make N=11 would require moving it to node 3 (123456 mod 11 = 3). This means that a small change in cluster size triggers a massive, expensive reshuffling of data across almost all nodes.
  * The Hashing with Ranges Solution: A better approach, used by modern systems, is to divide the entire possible output range of the hash function into a fixed number of partitions. A key is assigned to the partition whose hash range it falls into. This assignment of key-to-partition is permanent. Rebalancing only involves moving entire partitions between nodes. This is vastly more efficient, as only a fraction of the data (the partitions being moved) needs to be transferred, not a majority of individual keys.

🎯 Interview Preparation: Q&A and Comparisons

Top 3 Interview Questions

1. An e-commerce platform uses key-range partitioning for its orders, using order_timestamp as the primary key. During a flash sale, the system experiences severe performance degradation. What is the likely cause, and how would you redesign the partitioning strategy to mitigate this?

Answer:

* Definition of the Problem: The likely cause is a hot spot created by the key-range partitioning scheme combined with the access pattern. During a flash sale, a massive volume of new orders is created in a very short time. Since the primary key is order_timestamp, all new writes have keys that are close together and sequential. In a key-range partitioned system, this means all write requests are being directed to a single partition—the one that owns the current time range. This one partition and its host node become a bottleneck, while all other nodes in the cluster sit idle.
* Mechanism of Failure: This failure mode is a classic downside of key-range partitioning. While it excels at range queries (e.g., "get all orders from last Tuesday"), it is highly susceptible to skew when keys are monotonic or sequential. The system's write throughput is limited to the capacity of a single node, defeating the purpose of a distributed, scalable database.
* Proposed Redesign: To solve this, the partitioning strategy should be changed to one that distributes writes more evenly.
  1. Hash Partitioning: The most direct solution is to switch to hash partitioning on a primary key. Instead of order_timestamp, we could use a unique order_id (like a UUID). Applying a hash function to the order_id would distribute new orders uniformly across all partitions, eliminating the write hot spot and leveraging the full capacity of the cluster. The trade-off is that efficient range queries on the primary key are no longer possible.
  2. Compound Primary Key (Hybrid Approach): A more nuanced solution, as seen in systems like Cassandra, is to use a compound primary key. The key could be (customer_id, order_timestamp). We would then partition by the hash of the customer_id. This distributes the write load by customer, which is likely to be much more even than by time. Within each partition, orders would still be sorted by order_timestamp, allowing for efficient range queries to retrieve all orders for a specific customer within a given time frame. This approach balances load distribution with the ability to perform common, targeted range queries.

2. Explain the fundamental trade-off between Document-Partitioned (Local) and Term-Partitioned (Global) secondary indexes. Provide a use case where you would choose one over the other.

Answer:

* Core Concepts:
  * A Document-Partitioned (Local) Index means each partition maintains its own secondary index, which only contains entries for the documents stored on that specific partition.
  * A Term-Partitioned (Global) Index means the secondary index is itself partitioned by the indexed term (the value) and is stored separately from the primary data. An entry in a global index (e.g., for color:red) can point to documents located in any of the primary data partitions.
* The Fundamental Trade-off: The choice between these two methods represents a classic database design trade-off between write efficiency and read efficiency.
  * Write Path: Document-partitioned indexes have a simple, fast write path. When a document is written, only one node needs to be updated: the node containing the primary data partition, which also updates its local secondary index. In contrast, global indexes have a complex and slower write path. A single document write might require updates to multiple different index partitions, which could reside on different nodes, potentially requiring distributed transactions or leading to asynchronous index updates.
  * Read Path: Term-partitioned indexes offer a highly efficient read path. To find all documents with color:red, the query is sent directly to the single partition responsible for that index term. In contrast, document-partitioned indexes have an expensive read path. The same query requires a scatter/gather operation: the query must be broadcast to all partitions, each partition searches its local index, and the results from all partitions must be collected and aggregated at a routing tier or the client. This is slower and prone to tail latency amplification.
* Use Case Examples:
  * Choose Document-Partitioned (Local): An application with a very high write throughput and where secondary index queries are less frequent or can tolerate higher latency. For example, a logging system that ingests a massive volume of event data. Writes must be extremely fast. Searches for specific logs (e.g., by user_id) might happen occasionally for debugging but are not the primary, performance-critical workload.
  * Choose Term-Partitioned (Global): An application where read performance on the secondary index is critical. For example, a used car sales website where users must be able to quickly search and filter inventory by attributes like make, model, and color. These search queries are the core feature of the application and must be fast. The rate of new car listings (writes) is significantly lower than the rate of searches (reads), so optimizing for read efficiency by using a global index is the correct choice.

3. A database cluster uses a proxy/routing tier to direct client requests. Describe how this tier discovers and adapts to changes in the cluster topology, such as a partition being moved during rebalancing.

Answer:

* The Problem (Service Discovery): In a partitioned database, the assignment of partitions to physical nodes is not static; it changes during rebalancing, node additions/removals, or failovers. The routing tier must maintain an up-to-date map of this topology to correctly direct client requests. If its map is stale, it will send requests to the wrong node.
* Mechanism for Discovery and Adaptation: There are two primary architectural patterns for keeping the routing tier's knowledge current.
  1. Using a Coordination Service (e.g., ZooKeeper): This is a very common and robust approach.
    * Registration: Each node in the database cluster registers itself with a central, highly available coordination service like ZooKeeper.
    * Authoritative Mapping: ZooKeeper maintains the authoritative, ground-truth mapping of which partition is assigned to which node's IP address.
    * Subscription/Watches: The routing tier subscribes to this information in ZooKeeper. It establishes a "watch" on the data.
    * Notification: When a change occurs—for instance, a rebalancing operation moves Partition 5 from Node A to Node B—the cluster management logic updates the mapping in ZooKeeper. ZooKeeper then immediately notifies all subscribed components, including the routing tier.
    * Update: Upon receiving the notification, the routing tier updates its internal routing table to reflect the new topology. All subsequent requests for keys in Partition 5 are then sent to Node B. Systems like HBase, SolrCloud, Kafka, and MongoDB (with its config servers) use this model.
  2. Using a Gossip Protocol: This is a decentralized approach that avoids dependency on an external service.
    * Peer-to-Peer Communication: Nodes in the cluster periodically exchange ("gossip") information about their state and their knowledge of other nodes' states.
    * State Dissemination: When a change occurs (e.g., a node takes ownership of a new partition), it gossips this update to a few other random nodes. Those nodes then gossip the information to others, and the change eventually propagates throughout the entire cluster.
    * Routing Logic: In this model, requests can often be sent to any node. If the receiving node is not the correct owner for the partition, its up-to-date knowledge (from the gossip protocol) allows it to forward the request to the correct node. This is approach 1 in Figure 6-7. The routing tier could also participate in the gossip protocol to learn the topology directly. Systems like Cassandra and Riak use this approach.

Key Comparisons/Contrasts

Feature	Key Range Partitioning	Hash Partitioning
Data Organization	Keys are stored in sorted order. Each partition holds a continuous block of keys (e.g., A-F, G-M).	Keys are distributed based on the output of a hash function. Adjacent keys are scattered across different partitions.
Primary Use Case	Best for applications that require frequent range scans on the primary key (e.g., "fetch all sensor data from last month").	Best for applications that require even load distribution and primarily perform single-key lookups.
Risk of Hot Spots	High. Sequential or monotonic keys (timestamps, auto-incrementing IDs) can cause all writes to target a single partition.	Low. A good hash function provides uniform distribution, spreading the load from skewed or sequential keys evenly.
Rebalancing Strategy	Typically uses dynamic partitioning. Partitions are split when they grow too large and merged when they shrink.	Typically uses a fixed number of partitions. Rebalancing involves moving entire partitions between nodes.

🛠️ Practical Application and Implementation Tricks

Real-World Use Cases

1. Social Media Timelines: In a social media application, a user's posts need to be retrieved efficiently, often sorted by time. Cassandra's hybrid partitioning model is a perfect fit. A table for posts could use a compound primary key of (user_id, post_timestamp). The data is partitioned by the hash of the user_id, ensuring that data for different users is spread across the cluster. Within a single partition, all posts for a single user are stored sorted by post_timestamp. This allows the application to efficiently query for "the 20 most recent posts by user X" with a single, fast range scan on one partition, avoiding a costly full-table scan or scatter/gather.
2. IoT Sensor Data Ingestion: An application collecting time-series data from a vast network of sensors (e.g., temperature, pressure readings) needs to handle a very high write throughput. Using the timestamp alone as the key would create a massive hot spot with key-range partitioning. A better approach is to use a compound key like (sensor_name, timestamp). By partitioning on the hash of the sensor_name, the write load from thousands of concurrently active sensors is evenly distributed across the cluster, preventing bottlenecks. This still allows for efficient range queries to fetch all data for a specific sensor over a period of time.
3. Search Functionality (E-commerce): A website selling products like used cars needs to allow users to filter by various attributes (e.g., color, make, price). This requires secondary indexes. A global, term-partitioned index is ideal. The index for "color" would be partitioned so that color:red might be on Partition A and color:blue on Partition B. When a user searches for red cars, the query is routed directly to Partition A, which returns a list of all matching document IDs. This makes the primary user-facing search functionality very fast, justifying the increased complexity and latency of writes (when a new car listing is added).

Implementation Pitfalls/Tricks

1. Pitfall: The "Empty Database" Bottleneck: With key-range partitioning, an empty database starts with a single partition. Until the dataset grows large enough to trigger the first split, all writes are processed by a single node, while the other nodes sit idle. Trick: Use pre-splitting. Manually configure an initial set of partition boundaries on the empty database based on an expected distribution of keys. This ensures that load is distributed from the very first write.
2. Pitfall: Celebrity Hot Spots on Write: Hashing a key (e.g., user_id) distributes load well on average but fails in the extreme case where all activity is on a single key (e.g., millions of users commenting on a celebrity's post). Hashing the same ID always yields the same result. Trick: Use an application-level fan-out. For known hot keys, append a random suffix (e.g., from 1 to 100) to the key before writing: celebrity_id_54. This splits the writes for that single logical key across 100 different physical keys, distributing them to different partitions. The downside is that reads become more complex, as they must now query all 100 possible keys and combine the results.
3. Pitfall: Expensive Cross-Shard Joins: In a sharded database, a join operation that requires data from tables located on different shards can be extremely slow, as it may involve network round trips to fetch data from multiple nodes and perform the join logic on a proxy or the application server. Trick: Design the sharding scheme to co-locate related data. Ensure that data that is frequently joined together lives on the same shard. For instance, in an e-commerce database sharded by customer_id, ensure that the customer's orders and shipping addresses are also sharded by customer_id, placing them all on the same partition. This allows join queries for a specific customer to be executed entirely on a single node.
4. Pitfall: Rebalancing Storms: Fully automated rebalancing, especially when coupled with automatic failure detection, can be dangerous. A temporary network issue or a node overloaded by a query can cause it to respond slowly. If other nodes incorrectly flag it as "dead," they may trigger an automatic rebalancing process to move partitions away from it. This adds massive network and I/O load to the already struggling node and the rest of the cluster, potentially causing a cascading failure. Trick: Keep a human in the loop. For critical operations like rebalancing, many systems (e.g., Couchbase, Riak) generate a suggested rebalancing plan but require an administrator to review and approve it before execution. This prevents unpredictable performance hits and allows for better capacity planning.
5. Pitfall: Inefficient Scatter/Gather Queries: When using document-partitioned (local) secondary indexes, every query must be sent to every partition. This is inefficient and prone to tail latency amplification, where the overall query is only as fast as its slowest partition. Trick: Design the partitioning scheme around the most critical queries. If possible, structure the primary key such that the most common secondary index queries can be satisfied from a single partition. For example, if users frequently search for other users in their city, consider including city as part of the primary key that determines the partition, even if it feels redundant.
