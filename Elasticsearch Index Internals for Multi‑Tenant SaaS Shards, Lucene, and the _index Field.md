# Elasticsearch Index Internal Architecture Research

**File:** `elastic_index_internal_architecture_research.md`  
**Last updated:** 2026-07-08  

---

## 1. Problem Statement (Multi‑Tenant SaaS Context)

For a large multi‑tenant SaaS platform with thousands of customers, one of the key architectural questions is:

> Does Elasticsearch internally create **physically separate data structures** for every index, or does it maintain a **single shared data pool** with the `_index` field acting as a logical partition?

This decision affects:

- **Isolation vs. consolidation**
- **Scalability** (CPU, RAM, I/O)
- **Operational complexity** (cluster state, index management)
- **Security and fault‑isolation**

Specifically, we want to understand:

1. **Internal storage of multiple indices**
   - Does each index map to its own on‑disk structures?
   - Or are documents for different indices co‑mingled in a shared storage pool?

2. **Per‑customer index design**
   - If we create **one index per customer**, does Elasticsearch:
     - Allocate separate shards per index?
     - Create truly independent Lucene data structures?
   - Or does it just tag documents with `_index` and keep them in a shared pool?

3. **Meaning of `_index`**
   - Is `_index` merely a **metadata field** returned in search results?
   - Or is `_index` analogous to a **SQL column** that is stored in every document and used for partitioning queries?

4. **Resource impact**
   - How do separate indices vs. a shared index affect:
     - CPU usage (search, merges, refresh)
     - Memory (heap, file cache)
     - Disk I/O
     - Cluster state size
     - Management overhead

We will rely on official sources from Elastic and Apache Lucene to avoid speculation and clearly separate **facts** from **assumptions**.

---

## 2. Elasticsearch Internal Architecture

### 2.1 Core Concepts

From the official Elasticsearch documentation:

- A **cluster** is a collection of one or more nodes that together hold all your data and provide federated indexing and search across all nodes.  
  Source: [Elasticsearch: Set up a cluster](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html)

- A **node** is a single Elasticsearch instance participating in a cluster.  
  Source: [Elasticsearch: Nodes](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html)

- An **index** is a logical namespace that maps to one or more **primary shards** and their **replicas**.  
  Source: [Elasticsearch: Getting started — Indexes and Shards](https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html)

- A **shard** is a **Lucene index** under the hood:  
  > “A shard is a **Lucene index** and is a low‑level worker unit that holds just a slice of your data.”  
  Source: [Elastic: Elasticsearch from the Bottom Up (blog)](https://www.elastic.co/blog/found-elasticsearch-from-the-bottom-up)

- A **Lucene index** is composed of multiple **segments**, which are immutable data files Lucene uses to store indexed documents.  
  Source: [Lucene: Index File Formats](https://lucene.apache.org/core/current/core/org/apache/lucene/codecs/lucene95/package-summary.html)

### 2.2 Hierarchy & Relationships

The effective hierarchy in an Elasticsearch cluster is:

```mermaid
graph TD
  A[Cluster] --> B[Node]
  B --> C[Index]
  C --> D[Shard (Primary / Replica)]
  D --> E[Lucene Index]
  E --> F[Segments]
  F --> G[Documents & Inverted Index Structures]
```

More explicitly:

1. **Cluster**
   - Contains multiple **nodes**.
   - Maintains **cluster state** including index metadata and shard routing.

2. **Node**
   - Runs one or more **shard** instances (primary and/or replicas).
   - Uses local disk for the Lucene indices of its shards.

3. **Index**
   - Logical grouping of documents.
   - Configured with:
     - Number of **primary shards**
     - Number of **replica shards**
     - **Settings**, **mappings**, **aliases**, lifecycle policies, etc.
   - Each shard is an independent **Lucene index**.

4. **Shard**
   - Primary shard: authoritative Lucene index.
   - Replica shard: copy of the primary shard’s Lucene index (for HA and search load distribution).
   - Lucene index is stored in a directory containing multiple **segment** files.  
     Source: [Elasticsearch: Index Modules](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html)

5. **Lucene Index**
   - Consists of multiple **segments**.
   - Each segment contains an inverted index, stored fields, doc values, etc.  
     Source: [Lucene: Core Overview](https://lucene.apache.org/core/9_10_0/core/overview-summary.html)

6. **Segments**
   - Immutable units of storage.
   - Created during indexing; periodically merged as documents accumulate or are deleted.
   - Search executes **per segment** and then merges results.  
     Source: [Lucene: IndexWriter](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/index/IndexWriter.html)

---

## 3. What Exactly Is an Elasticsearch Index?

### 3.1 What Happens When an Index Is Created?

When you create an index via `PUT /my-index`, Elasticsearch:

1. **Allocates shard routing** in cluster state:
   - Decides which nodes will host the **primary shards** and **replica shards**.
   - Adds index metadata to the global **cluster state**.  
     Source: [Elasticsearch: Index Modules – Overview](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html)

2. **Initializes Lucene indices** on the target nodes:
   - On each node hosting a shard, Elasticsearch:
     - Creates a **directory** on disk for that shard.
     - Initializes a new **Lucene IndexWriter** for that shard.  
       This translates to “a new Lucene index instance per shard”.

3. **Establishes index‑level configuration:**
   - Each index has:
     - **Settings** (e.g., numberOfShards, numberOfReplicas, analyzers, refresh interval)
     - **Mappings** (field types, analyzers, doc values, etc.)
     - **Aliases**, **index lifecycle policies**, and various index‑level feature flags.  
   - These are stored as part of the **index metadata** in **cluster state**.  
     Source: [Elasticsearch: Create Index API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html)

### 3.2 Are Shards Independent?

**Fact:**
- Each **shard** is an independent **Lucene index**; there is **no shared Lucene index** across different Elasticsearch indices or shards.  
  Source: [Elastic blog: “Elasticsearch from the bottom up”](https://www.elastic.co/blog/found-elasticsearch-from-the-bottom-up)

Implications:

- Shards:
  - Maintain their own Lucene **index directory**, **segments**, and internal structures.
  - Are opened with independent Lucene `IndexReader` and `IndexWriter` instances.
  - Run their own **refresh**, **merge**, and **segment management** cycles.

### 3.3 Per‑Index Metadata

**Per index:**

- **Mappings:**  
  Each index can define its own field mappings; there is no global shared schema.  
  Source: [Elasticsearch: Mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html)

- **Settings:**  
  Includes shard/replica counts, refresh interval, analysis chain, translog settings, etc.  
  Source: [Elasticsearch: Index Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html)

- **Metadata:**  
  Stored in cluster state under `metadata.indices`.

So:

> **Every Elasticsearch index has its own primary/replica shards, with independent Lucene indices, independent mappings, and independent settings and metadata.**

---

## 4. Understanding the `_index` Field

### 4.1 What Is `_index`?

- `_index` is a **meta field** representing the name of the index a document belongs to.  
  Source: [Elasticsearch: Mapping Meta-Fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-fields.html#mapping-index-field)

From the official docs:

> “The `_index` meta‑field contains the index name that the document belongs to.”  
> Source: [Mapping Meta-Fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-fields.html#mapping-index-field)

### 4.2 Is `_index` Stored Inside Every Document?

Key points:

- `_index` is not a **user‑defined field** that you add; it is part of Elasticsearch metadata.
- Lucene itself has no concept of “Elasticsearch index name”; it only knows the **Lucene index directory**.
- Internally, documents are written to specific **Lucene indices** (per shard); the index name is known at the Elasticsearch layer.

**Fact‑level statement:**

- Elasticsearch uses the index name to route a write request to a specific shard (Lucene index).
- For search responses, Elasticsearch **returns `_index` as metadata** about where the document came from.

The public documentation does not explicitly state whether `_index` is stored as a separate field in the Lucene document; however:

- `_index` is not a mappable field; it is similar to other meta fields like `_id`, `_score`.
- In searches across multiple indices, Elasticsearch includes `_index` in the hit metadata to distinguish which index each hit came from.

**Conclusion from docs:**

> `_index` is a **metadata view** over which index/shard the document was read from. It behaves more like an envelope/header than a user field.

### 4.3 Is `_index` Comparable to an SQL Column?

**No.** Differences:

- An SQL column (e.g., `index_name`) is a **stored attribute in every row** within a single physical table, used to partition or filter data.
- `_index` reflects the **index namespace**, and the data for each index is stored in **different Lucene indices (per shard)**.

Thus:

- There is **no shared underlying Lucene structure** where `_index` acts as a discriminator column.
- A query like `GET /index-a,index-b/_search` fans out to the shards of both indices, then merges results; it is not a scan of one giant table with `_index` filters.

### 4.4 Why Developers Misunderstand `_index`

Behaviors that cause confusion:

1. **Multi‑index search API:**
   - You can search `/index-1,index-2/_search`, get hits with `_index` in the response.
   - This looks like filtering different partitions by “index name”.

2. **Index aliases:**
   - Aliases can point to multiple indices, again giving the illusion of a logical partition.  
     Source: [Index Aliases](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html)

3. **Meta‑field semantics:**
   - `_index`, `_id`, `_type` (in older versions) are all meta fields; it’s easy to think they’re stored fields.

**Fact vs. assumption:**

- **Fact:** Each Elasticsearch index maps to its own shards and Lucene indices.
- **Assumption (to be treated cautiously):** `_index` is not stored as an explicit field per document in Lucene; it’s primarily metadata at the Elasticsearch layer. The exact Lucene storage of `_index` is not detailed in public docs.

---

## 5. Data Storage: Separate Lucene Indices vs. Single Shared Pool

We now answer the main question:

> Does Elasticsearch store data as (A) separate Lucene indices per Elasticsearch index, or (B) one giant storage pool with an `_index` column?

### 5.1 Option A – Separate Lucene Indexes Per Elasticsearch Index (Per Shard)

From Elastic’s own engineering content:

- “Each shard is a fully functional and independent **Lucene index** that can be hosted on any node in the cluster.”  
  Source: [Elastic: Elasticsearch from the Bottom Up](https://www.elastic.co/blog/found-elasticsearch-from-the-bottom-up)

Also from Elasticsearch docs:

- Shards are the “basic unit of a distributed index” and each shard is a Lucene index.  
  Source: [Elasticsearch: Index Modules](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html)

Therefore:

- **Each Elasticsearch index has N primary shards.**
- **Each primary shard is a separate Lucene index.**
- **Each replica shard is another Lucene index copy.**
- There is **no single shared Lucene index** storing multiple Elasticsearch indices together.

### 5.2 Option B – Single Shared Storage with `_index` Column

This would imply:

- All documents of all Elasticsearch indices are stored in **one Lucene index** (or a small fixed set), with `_index` as a discriminator field.

**There is no official Elasticsearch or Lucene documentation supporting this.**  
All official material emphasizes shards as **independent Lucene indices**.

### 5.3 Conclusion for Storage Model

**Fact-based conclusion:**

> Elasticsearch uses **Option A**: **Each index is comprised of one or more shards, and each shard is its own Lucene index with its own segments.** There is **no single shared storage pool** with `_index` acting as a logical column to partition documents.

---

## 6. Lucene Internals (Relevant to Shards)

### 6.1 What Is a Lucene Index?

From Apache Lucene:

- A Lucene index is a **directory of files** that together provide indexing and search over a set of documents.  
  Source: [Lucene: Core Overview](https://lucene.apache.org/core/9_10_0/core/overview-summary.html)

Characteristics:

- An index is built from **segments**.
- Each segment is read‑only once written.
- Indexing adds new segments; deletes are recorded in a deletes file.

### 6.2 What Is a Segment?

From Lucene docs:

- A **segment** is a **fully self‑contained mini‑index** with its own postings, stored fields, doc values, norms, etc.  
  Source: [Lucene: Index File Formats](https://lucene.apache.org/core/current/core/org/apache/lucene/codecs/lucene95/package-summary.html)

Properties:

- Immutable after creation.
- New documents are appended by creating new segments.
- Merges combine small segments into larger ones.

### 6.3 How Documents Are Stored

At a high level:

1. Documents are added via **IndexWriter**.
2. Documents are buffered and periodically flushed to disk as new **segments**.
3. Each document:
   - Is represented via inverted index structures (for searchable fields).
   - May have **stored fields** for retrieval.
   - May have **doc values** for aggregations/sorting.

Source: [Lucene: IndexWriter](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/index/IndexWriter.html)

### 6.4 How Searches Happen

- Searches in Lucene are executed via an **IndexSearcher**.
- The searcher:
  - Maintains an **IndexReader** that is typically a `DirectoryReader` composed of multiple segment readers.
  - Executes query logic **per segment**, then merges the results.  
  Source: [Lucene: IndexSearcher](https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/search/IndexSearcher.html)

In Elasticsearch:

- Every shard corresponds to a Lucene index with its own `IndexSearcher`.
- A cluster‑wide search:
  - Fans out to all relevant shards.
  - Executes Lucene search per shard (per segment).
  - Combines results (e.g., top‑K per shard → global top‑K).

### 6.5 How Merges Happen

- Lucene’s `IndexWriter` periodically merges smaller segments into larger ones to improve search performance and reclaim space.
- Elasticsearch uses a **merge policy** (e.g., TieredMergePolicy) configured per index.  
  Source: [Elasticsearch: Index Modules – Merge](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-merge.html)

Key property:

- **Merging is done per shard (per Lucene index)**; there is no cross-shard or cross-index merge.

### 6.6 Why Each Shard Is an Independent Lucene Index

From Elastic:

- “Each shard is a Lucene index.”  
  Source: [Elasticsearch from the Bottom Up](https://www.elastic.co/blog/found-elasticsearch-from-the-bottom-up)

Therefore:

- Different Elasticsearch indices do not share segments or a common Lucene `IndexWriter`.
- There is **no shared on‑disk segment** that contains documents from multiple Elasticsearch indices.

---

## 7. Resource Consumption When Creating New Indices

When you create a new Elasticsearch index (with its own shards), several resources are impacted.

### 7.1 JVM Heap

- Each shard allocates:
  - Lucene `IndexWriter` and `IndexReader` structures.
  - Query caches (filter cache, results cache, field data caches, etc., depending on configuration).
- Elasticsearch documentation and Elastic engineering blogs repeatedly warn that **too many shards** (and thus **too many indices**) will:
  - Increase heap usage.
  - Increase garbage collection pressure.

Source:  
- [Elastic blog: “How many shards should I have in my Elasticsearch cluster?”](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster)

Key takeaway:

> Every new index (with its shards) introduces additional heap overhead.

### 7.2 Cluster State

- Each index adds:
  - An `IndexMetadata` entry to **cluster state**.
  - Shard routing metadata.
- Large numbers of indices (e.g., thousands) inflate cluster state size, which:
  - Must be held in memory on the master node(s).
  - Must be transmitted to all nodes when updated.

Sources:  
- [Elasticsearch: Cluster State](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-state.html)  
- [Elastic blog: “Troubleshooting cluster state issues”](https://www.elastic.co/blog/troubleshooting-cluster-state-issues)

### 7.3 File Descriptors

- Each shard (Lucene index) opens **multiple files**:
  - Segment files.
  - Translog files.
  - Index metadata and commit files.
- More shards ⇒ more open file descriptors per node.

Source:  
- [Elasticsearch: File Descriptors](https://www.elastic.co/guide/en/elasticsearch/reference/current/setting-system-settings.html#setting-system-settings-fd)

### 7.4 Metadata

- Per‑index and per‑shard metadata:
  - Settings, mappings, index templates, ILM policies, etc.
  - Stored in cluster state, logs, and internal metadata indices (like `.security`, `.kibana`, etc.).

The **overhead per index** is not negligible; large index counts multiply this.

### 7.5 Search Threads & Execution Context

- Elasticsearch uses **thread pools** for search and index operations.  
  Source: [Elasticsearch: Thread Pools](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html)

Implications:

- Searching many small indices:
  - Increases the number of shards a query must hit.
  - Leads to more per‑shard search tasks and context switches.
- Each shard search may be executed by a separate thread, increasing scheduling overhead.

### 7.6 Segment Management, Refresh, and Merge

For each shard:

- **Refresh**:
  - Periodically opens a new `IndexSearcher` to make recent changes visible.
  - Per‑shard process; more shards ⇒ more frequent refresh cycles.  
    Source: [Elasticsearch: Near real-time search](https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html)

- **Merge**:
  - Per‑shard merges of Lucene segments.
  - Consumes CPU and I/O; overhead grows linearly with the number of shards.  
    Source: [Index Modules – Merge](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-merge.html)

### 7.7 Why Thousands of Indices Become Expensive

Official Elastic guidance:

- “Each shard carries overhead. Having too many small shards can cause performance issues and waste resources.”  
  Source: [How many shards should I have?](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster)

Combined effects:

- More index metadata → bigger cluster state.
- More shards → more heap usage, more open files, more refresh/merge operations.
- More shards per query → more thread overhead, more per‑shard search execution.

Therefore:

> **Creating thousands of small indices (and thus thousands of shards) leads to substantial overhead in heap, cluster state, file descriptors, and CPU/I/O due to per‑shard Lucene operations.**

---

## 8. Multi‑Tenant Design Analysis

We compare two approaches for a multi‑tenant SaaS.

- **Approach A:** Single (or few) shared index(es) with a `customerId` field + routing on `customerId`.
- **Approach B:** Separate index per customer.

### 8.1 Approach A – Single Index, `customerId` Field, Routing

Characteristics:

- All tenants share the same index (or a small fixed set of indices).
- Each document has a **tenant/`customerId` field**.
- Optionally, you use **routing** by `customerId` to control shard placement.  
  Source: [Elasticsearch: Routing](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html#index-routing)

Pros:

- **Fewer indices & shards:**
  - Lower cluster state overhead.
  - Fewer Lucene index instances, fewer merges/refresh cycles.
- **Better resource utilization:**
  - Large shards with many documents are typically more efficient than many tiny shards.
- **Simpler lifecycle management**:
  - ILM, rollover policies apply to a smaller number of indices.

Cons:

- **Security/Isolation needs extra care:**
  - Must enforce tenant filters (`customerId=X`) at the application layer or with features like **document‑level security** (in Elastic’s commercial features).
- **No hard physical isolation:**
  - A heavy tenant can impact shared shard performance.
- **Index mapping needs to be shared**:
  - You cannot easily give tenants completely different schemas in the same index.

### 8.2 Approach B – Separate Index per Customer

Characteristics:

- Each customer has their own index: `customer-123-index`, `customer-456-index`, etc.
- Each index has its own shards, mappings, and settings.

Pros:

- **Stronger logical/operational isolation:**
  - You can delete or snapshot a single tenant’s data independently.
  - Failures in one index (e.g., mapping explosion) do not directly affect others.
- **Potentially simpler security model:**
  - May be easier to restrict access by index (e.g., per‑index roles).
- **Per‑tenant tuning:**
  - Different mappings, analyzers, and shard counts per tenant.

Cons:

- **Many indices ⇒ many shards ⇒ large overhead** (heap, file descriptors, merges, cluster state).
- **Operational complexity:**
  - Managing templates, ILM policies, and rollover for thousands of indices is more complex.
- **Cluster state growth:**
  - Each index adds metadata; thousands of indices can stress the master node and make state updates slower.
- **Search across tenants gets expensive:**
  - Cross‑tenant reporting or analytics must fan out to many small indices.

### 8.3 Comparison Tables

#### Table 1 – Resource & Performance

| Aspect              | Approach A: Single Shared Index              | Approach B: Index per Customer                                |
|---------------------|----------------------------------------------|--------------------------------------------------------------|
| CPU (search)        | Fewer shards per query; efficient segment search; good cache utilization | Many small shards; high per‑shard overhead; less cache locality |
| CPU (indexing)      | Efficient batching into shared shards        | Many small translogs and merges; more per‑shard overhead     |
| JVM Heap            | Fewer shard contexts; lower overhead         | Per‑shard readers, writers, caches → much higher heap usage  |
| I/O (disk)          | Large segments; efficient merges and caching | Many small segment files; more random I/O per tenant         |
| Search Latency      | Generally lower and more predictable, provided per‑tenant filtering is efficient | May be good for per‑tenant search if each tenant is small and isolated; suffers on multi‑tenant aggregations |
| Cluster State Size  | Smaller – limited number of indices          | Large – thousands of index and shard metadata entries        |

#### Table 2 – Operational & Architectural

| Aspect              | Single Shared Index                               | Index per Customer                                       |
|---------------------|---------------------------------------------------|----------------------------------------------------------|
| Operational Complexity | Simple ILM, templates, maintenance for few indices | High – templates, ILM, mappings per tenant; more automation required |
| Scalability Pattern | Scale by adding nodes and tuning shard counts for shared index | Scale by splitting tenants into tiers or clusters; index count may become bottleneck |
| Recovery            | Restore fewer larger indices from snapshot       | Potentially restore single tenant only, but many indices to manage |
| Isolation           | Soft isolation only; noisy neighbors possible    | Better operational isolation per tenant                  |
| Security            | Needs document‑level constraints or app‑level filters | Per‑index auth easier (role to index mapping)           |
| Maintenance         | Easier – limited set of indices                  | Complex – index life cycle for thousands of tenants     |

#### Table 3 – When Each Approach Fits

| Scenario                                  | Recommended Approach  | Rationale |
|-------------------------------------------|-----------------------|-----------|
| Thousands of small tenants                | Shared index          | Avoid index/shard explosion and cluster state bloat |
| Few large enterprise tenants              | Per‑tenant index or cluster | Isolation and bespoke configurations per major customer |
| Regulatory isolation (data residency)     | Dedicated clusters or per‑cluster per region | Strongest isolation; index choice follows within clusters |
| Strong multi‑tenant reporting & analytics | Shared index          | Efficient cross‑tenant aggregations and queries |

---

## 9. Industry Practices (Public Information Only)

**Important disclaimer:**  
Production architectures of specific companies (GitHub, Salesforce, Shopify, Stripe) are generally **not fully disclosed**. Any conclusions here are based only on public material and should be treated as **indicative, not authoritative**.

### 9.1 Elastic

- Elastic’s own documentation and blogs emphasize:
  - Avoiding too many shards and indices.
  - Using shared indices with routing and multi‑tenant filtering where appropriate.  
  Source: [Elastic blog: How many shards should I have?](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster)

### 9.2 GitHub, Salesforce, Shopify, Stripe

Public patterns (from talks/blogs—not always Elasticsearch‑specific):

- **Stripe** (Postgres sharding, not Elasticsearch): heavy use of logical sharding with tenant identifiers, not table‑per‑customer.
- **Salesforce**: multi‑tenant architecture often uses **shared schemas** with tenant identifiers; in some cases separate pods/clusters for isolation.
- **Shopify/GitHub**: strong emphasis on **sharding by account/tenant** rather than isolated database per user.

While these are not Elasticsearch‑specific, they align with the common practice in large SaaS:

> Use **shared physical resources** with **logical tenant separation** (e.g., `tenantId` field + routing) and reserve per‑tenant physical isolation (index or cluster) for very large or high‑risk tenants.

Due to lack of explicit Elasticsearch internal cluster details in public documents from these companies, we must classify this as **informed analogy**, not a fact about their ES usage.

---

## 10. Best Practices According to Elastic

Elastic’s guidance (from documentation and blogs):

### 10.1 When to Use a Shared Index

Use a **shared index** (with tenant/`customerId` field) when:

- You have **many tenants**, each with relatively small data volumes.
- Schema is mostly shared.
- You need **cross‑tenant analytics**.
- You want to **minimize shard/index overhead**.

Sources:

- [How many shards should I have?](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster)
- [Index lifecycle management docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)

### 10.2 When to Use Index per Tenant

Use **index per tenant** when:

- You have **a moderate number of tenants** (e.g., hundreds, not tens of thousands).
- Tenants differ significantly in:
  - Schema requirements.
  - Retention/ILM policies.
  - Performance/SLAs.
- You want:
  - Easier **per‑tenant backup/restore**.
  - Simpler **per‑tenant deletion** (e.g., GDPR “right to be forgotten” by dropping entire index).

Elastic documentation does not prescribe exact tenant counts, but repeatedly warns against having **too many small indices**.

### 10.3 When to Use Dedicated Clusters per Tenant

Dedicated **cluster per tenant** (or per group of large tenants) is recommended when:

- A tenant’s data is **very large** or **highly sensitive**.
- You want:
  - Strong isolation (performance and security).
  - Independent upgrade and maintenance cycles.

Sources:

- [Cross-cluster search](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cross-cluster-search.html)

In practice:

- Large SaaS providers may combine strategies:
  - **Tier 1 tenants**: dedicated clusters or indices.
  - **Long tail**: shared indices with tenant fields.

---

## 11. Final Conclusion

### 11.1 Direct Answer to the Core Question

> **Does Elasticsearch internally create separate data structures for every index, or a single shared storage with `_index` acting as logical partitioning?**

**Based strictly on official documentation and Elastic engineering material:**

1. **Each Elasticsearch index is composed of one or more shards.**
2. **Each shard is a separate Lucene index with its own segments and on‑disk data.**  
   Source: [Elasticsearch from the Bottom Up](https://www.elastic.co/blog/found-elasticsearch-from-the-bottom-up)

Therefore:

> **Elasticsearch internally creates separate data structures (Lucene indices) per shard, and thus per index. It does *not* use a single shared storage pool where `_index` acts as a logical partition field.**

The `_index` meta‑field is **not** equivalent to a SQL column that partitions one shared table; instead, it indicates which index (and thus set of shards/Lucene indices) the document resides in.

### 11.2 Recommended Architecture for a Large Multi‑Tenant SaaS with Thousands of Customers

Based on:

- Elastic’s guidance on shard and index counts.
- Resource and complexity analysis in section 8.
- The reality that each index adds real overhead.

**Recommendation:**

1. **Default pattern: Shared index (Approach A) with `customerId` field + routing.**
   - Use one or a small number of indices per “data type” or per time period (for ILM/rollover), not per tenant.
   - Enforce **mandatory tenant filtering** in application code and/or via document‑level security.
   - Use **routing by `customerId`** to group each tenant’s data into a limited set of shards, improving locality.

2. **Selective elevation: Index per large or special tenant (Approach B) for a small subset.**
   - For very large or high‑SLA tenants, create dedicated indices (and possibly dedicated clusters).
   - This hybrid approach avoids index explosion while allowing per‑tenant tuning where it matters.

3. **Dedicated clusters for top‑tier tenants only.**
   - For tenants with strict regulatory or isolation requirements, move to a dedicated cluster with its own indices.

In short:

> For **thousands of customers**, the **primary architecture should be shared indices with tenant identifiers and routing**, not index‑per‑customer. Index‑per‑tenant should be reserved for a **relatively small subset** of large or special tenants where the operational and resource cost is justified by isolation and configurability benefits.

---

## 12. References

**Elasticsearch Official Documentation**

- Elasticsearch Setup & Clusters  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html
- Elasticsearch Nodes  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html
- Getting Started – Indices and Shards  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/getting-started.html
- Index Modules (shards, settings, merge, etc.)  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html
- Create Index API  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html
- Mapping  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html
- Mapping Meta-Fields (`_index`)  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-fields.html#mapping-index-field
- Near Real-Time Search (refresh)  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html
- Index Modules – Merge  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-merge.html
- Routing  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html#index-routing
- Index Aliases  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-aliases.html
- Cluster State  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-state.html
- Thread Pools  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html
- File Descriptors & System Settings  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/setting-system-settings.html#setting-system-settings-fd
- Index Lifecycle Management (ILM)  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html
- Cross-Cluster Search  
  https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cross-cluster-search.html

**Elastic Engineering Blogs & Talks**

- Elasticsearch from the Bottom Up  
  https://www.elastic.co/blog/found-elasticsearch-from-the-bottom-up
- How many shards should I have in my Elasticsearch cluster?  
  https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster
- Troubleshooting cluster state issues  
  https://www.elastic.co/blog/troubleshooting-cluster-state-issues

**Apache Lucene Documentation**

- Core Overview (Indexing & Searching)  
  https://lucene.apache.org/core/9_10_0/core/overview-summary.html
- Index File Formats (Segments)  
  https://lucene.apache.org/core/current/core/org/apache/lucene/codecs/lucene95/package-summary.html
- IndexWriter (Indexing & Merges)  
  https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/index/IndexWriter.html
- IndexSearcher (Searching)  
  https://lucene.apache.org/core/9_10_0/core/org/apache/lucene/search/IndexSearcher.html

---

## Principal Engineer Review – Critical Assessment

From a Principal/Architect perspective, here is a critical review of the document:

### Strengths

- **Clear conclusion** on the core question, based firmly on official Elastic and Lucene statements:  
  - Shards are Lucene indices.  
  - There is no single shared Lucene index with `_index` as a column.
- **Good explanation of hierarchy** (cluster → node → index → shard → Lucene index → segments).
- **Reasonable and aligned** with Elastic best practices for shard and index counts.
- **Explicit separation of facts vs. assumptions** around the internal storage of `_index`.

### Weak Assumptions / Areas of Uncertainty

1. **Exact internal representation of `_index` in Lucene**
   - The document notes that `_index` is a meta‑field but does not demonstrate:
     - Whether it is stored in Lucene as a hidden field.
     - Or purely derived from the index context.
   - Public docs are not explicit; we inferred behavior logically.
   - **Mitigation:** Marked explicitly as *assumption*, not fact. If critical, one would need to inspect Lucene source or Elastic source for `IndexShard`/`Engine` implementations.

2. **Industry practices for specific companies**
   - References to GitHub, Salesforce, Shopify, Stripe are based on **general SaaS patterns**, not explicit Elasticsearch disclosures.
   - **Mitigation:** Clearly framed as *analogy* and not factual about their ES usage.

3. **Tenant count thresholds**
   - The document implies that “thousands of indices” is problematic, consistent with Elastic guidance but doesn’t give an exact safe upper bound.
   - Elastic suggests general shard counts per node (not a strict formula).
   - **Mitigation:** Rephrased as “thousands of indices/shards are typically problematic” rather than prescribing hard limits.

4. **Hybrid architecture recommendation**
   - The recommendation (shared index for most tenants + index/cluster per large tenant) is common in practice but not formally prescribed in docs.
   - **Mitigation:** It is presented as a **best‑effort design recommendation**, not an official Elastic recipe.

### Missing Considerations / Future Extensions

1. **Security Model Details**
   - The document briefly references document‑level security but does not dive deeply into:
     - How field/role‑based security interacts with multi‑tenant designs.
     - Performance implications of document‑level security vs. app‑layer filters.
   - For a real design decision, this deserves deeper exploration.

2. **Cross‑Cluster Search / Federation**
   - For very large SaaS deployments, cross‑cluster search (CCS) and cross‑cluster replication (CCR) can be major design tools:
     - Tenant‑based clustering (country, region, tier).
     - Search across clusters.
   - These mechanisms are only touched on briefly via references.

3. **Index Lifecycle Management Strategies for Multi‑Tenant**
   - The document discusses ILM mostly at a high level; a real design would specify:
     - Rollover strategy for shared indices (e.g., daily vs. monthly).
     - How to avoid shard imbalance when tenants differ in traffic.

4. **Cost Modeling**
   - No quantitative numbers (heap per shard, file descriptors per shard, etc.) are given.
   - For an actual decision, it would be helpful to model:
     - Example cluster with X nodes, Y shards, Z tenants.
     - Approximate heap and FD usage under each model.

5. **Operational Tooling**
   - Real deployments will rely on:
     - Automation to create/manage indices.
     - Monitoring for shard count, cluster state size, and search latencies.
   - This is out of scope but worth mentioning as a next step.

### Overall Verdict

- The document is **technically sound** on the core question: **Elasticsearch uses separate Lucene indices per shard/index; `_index` is not a logical partition column over a shared pool.**
- The **recommended architecture** (shared indices with `customerId` + routing, selective per‑tenant elevation) is aligned with Elastic’s shard/index guidance and general SaaS design best practices.
- All major non‑obvious assumptions are either:
  - Called out as assumptions, or
  - Anchored to public Elastic/Lucene documentation.

For a production‑grade design decision, I would:

1. Validate resource assumptions by load‑testing both models on a representative cluster.
2. Review Lucene/Elasticsearch source if the exact storage of `_index` is business‑critical.
3. Extend this document with:
   - Concrete ILM and routing configuration examples.
   - A sizing guide (shards per node, indices per cluster).
   - Security model deep‑dive (document‑level vs. index‑level security).

With those caveats, the document is ready as a solid architectural reference for deciding between shared indices and index‑per‑tenant in Elasticsearch.
