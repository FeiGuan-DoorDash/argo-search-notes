# How Search Requests Interact with Indexes

## Complete Search Flow with Example

Let's trace a search query: **"Find all pizza products in stores near downtown"**

---

## Step 1: Client Request

The client application initiates a gRPC call to the broker.

### BrokerRequest (proto)

```protobuf
namespace: "product"
search_query:
  filter:
    and:
      - keyword: {query: "pizza", field: "name"}
      - join_query:  # Search in parent namespace
          namespace: "store"
          filter:
            keyword: {query: "downtown", field: "location"}
  limit: 20
  offset: 0
  sort_by: [{field: "price", order: ASC}]
  routing:  # Optional shard routing
    route_by_key: "S001"  # Query specific shard
```

---

## Step 2: Broker Receives Request

### BrokerGrpcHandler
- **Port**: 50052
- **Method**: `broker(request: BrokerRequest)`

### BrokerServiceImpl.process()

1. **Convert proto → BrokerRequestModel**

2. **Determine result format:**
   - Check original format (FULL_DOCUMENTS)
   - Random rollout: Use FLAT_NORMALIZED? No

3. **Plan query:**
   - Validate query structure
   - Apply transformations
   - Create execution plan

---

## Step 3: Searcher Selection & Routing

### SearcherClientSelector.select()

```kotlin
select(
  namespace = "product",
  route = RouteByKeyModel(key = "S001")
)
```

**Process:**

1. **Get sharding config from BrokerIndexModel:**
   - `numberOfShards = 4`
   - `numberOfMicroShards = 1024`

2. **Calculate target shard:**
   ```
   microShardId = calculateMicroShardId("S001", 1024)
                = hash("S001") % 1024 = 157

   microShardsPerShard = 1024 / 4 = 256
   shardId = 157 / 256 = 0
   ```

3. **Return:** `[SearcherClient for Shard 0]`

### Available Searcher Clients

| Shard | Host | Port | Status |
|-------|------|------|--------|
| **Shard 0** | searcher-0 | 50051 | ✓ **SELECTED** |
| Shard 1 | searcher-1 | 50051 | Not queried (due to routing) |
| Shard 2 | searcher-2 | 50051 | Not queried |
| Shard 3 | searcher-3 | 50051 | Not queried |

> **Note:** Without routing parameter, ALL 4 shards would be queried!

---

## Step 4: Prepare Searcher Request

### BrokerServiceImpl.prepareSearcherRequest()

**Create SearcherRequest:**
- Add primary key to return fields: `product_id`
- Add sort fields to return: `price`
- Include all requested fields: `name`, `price`
- Maintain filter structure with join queries
- Set limit, offset, sort parameters

---

## Step 5: Fanout to Searchers (Parallel Execution)

### BrokerServiceImpl.search()

```kotlin
For each selected SearcherClient:
  Launch coroutine {
    searcherClient.searcher(searcherRequestProto)
  }

awaitAll() // Wait for all shards to respond
```

In this case, only 1 shard is queried:

```
SearcherClient (Shard 0)
└─> gRPC call to searcher-0:50051
```

---

## Step 6: Searcher Processes Request

### SearcherService.searcher() @ Shard 0

1. **Increment metrics** (`searches_in_flight`)

2. **Acquire semaphore permit** (concurrency control)
   - `maxPermits = 4 * num_processors`

3. **Call ArgoSearcher.search()**

### ArgoSearcher.search()

#### PHASE 1: Match and Rank

**1. Build Lucene Query from filter:**

```
BooleanQuery:
  MUST:
   - TermQuery(name: "pizza")
  FILTER:
   - TermQuery(_namespace: "product")
  MUST: (join query)
   - JoinQuery(
       fromIndex: "store",
       fromField: "store_id",
       toField: "store_id",
       query: TermQuery(
         location: "downtown"
       )
     )
```

**2. Create MatchingAndRankingCollectorManager**
   - Manages result collection

**3. Execute search across index slices:**

```
IndexSearcher.search(query, collector)
└─> Searches Lucene index segments in parallel
    (one coroutine per slice)
```

**Results:** Top document IDs with scores

#### PHASE 2: Hydrate (Fetch Field Values)

For each matched document ID:
```
DocumentAccessor.load(docId, fields)
└─> Read stored fields from Lucene
```

**Results:** Complete documents with all fields

### Return: ArgoResults
```json
{
  "documents": [...matched docs...],
  "totalMatchedDocuments": 45,
  "facets": {...}
}
```

---

## Step 7: Index Interaction (Lucene Level)

### Lucene Index Structure (Shard 0)

```
/data/searcher/repository/index/
```

#### Segment _0 (older):

**Inverted Index:**
```
name:
  "pizza" → [doc5, doc12, doc27, ...]
  "burger" → [doc3, doc8, doc19, ...]
  "salad" → [doc1, doc15, doc31, ...]

_namespace:
  "product" → [doc1, doc3, doc5, doc8, ...]
  "store" → [doc2, doc4, doc6, ...]

store_id: (join field)
  "S001" → [doc5, doc12]
  "S002" → [doc27]
```

**DocValues (for sorting/faceting):**
```
price:
  doc5 → 12.99
  doc12 → 8.99
  doc27 → 15.50
```

**Stored Fields (for retrieval):**
```json
doc5: {
  "product_id": "P001",
  "name": "Pizza",
  "price": 12.99,
  "store_id": "S001"
}

doc12: {
  "product_id": "P002",
  "name": "Pizza Deluxe",
  "price": 8.99,
  "store_id": "S001"
}
```

#### Segment _1 (newer, from incremental updates):
Similar structure with updated/new documents

### Query Execution Flow

1. Check inverted index for "pizza" → candidate docs
2. Apply join filter (only docs with stores matching query)
3. Collect matching doc IDs with relevance scores
4. Sort by price using DocValues
5. Retrieve stored fields for top K documents

### Query Matches:
```json
[
  {"product_id": "P001", "name": "Pizza", "price": 12.99},
  {"product_id": "P002", "name": "Pizza Deluxe", "price": 8.99}
]
```

---

## Step 8: Searcher Returns Response

### SearcherResponse (from Shard 0)

```json
{
  "documents": [
    {
      "product_id": "P002",
      "name": "Pizza Deluxe",
      "price": 8.99,
      "_score": 1.5
    },
    {
      "product_id": "P001",
      "name": "Pizza",
      "price": 12.99,
      "_score": 1.2
    }
  ],
  "totalMatchedDocuments": 2,
  "matchedDocumentsPerNamespace": {
    "product": 2
  },
  "metrics": {
    "cpuTimeNs": 5234000,
    "responseSizeBytes": 512
  }
}
```

---

## Step 9: Broker Merges Results

In this example: Only 1 shard queried, so minimal merging needed.

### If ALL shards were queried (no routing):

#### BrokerServiceImpl.mergeSearchResults()

**Input:** `Map<ShardId, SearchResults>`
- Shard 0: 2 docs (sorted by price ASC)
- Shard 1: 5 docs (sorted by price ASC)
- Shard 2: 3 docs (sorted by price ASC)
- Shard 3: 4 docs (sorted by price ASC)

#### K-Way Merge Algorithm

Uses a **PriorityQueue (MinHeap by price)**:

**Initial state** (first doc from each shard):
```
[Shard1Doc1(8.50), Shard0Doc1(8.99),
 Shard2Doc1(9.75), Shard3Doc1(10.20)]
```

**Iteration 1:**
- Poll: `Shard1Doc1(8.50)` → Add to results
- Push: Next from Shard1 → `Shard1Doc2(11.00)`

**Iteration 2:**
- Poll: `Shard0Doc1(8.99)` → Add to results
- Push: Next from Shard0 → `Shard0Doc2(12.99)`

Continue until 20 docs collected.

**Aggregate metadata:**
```
totalMatchedDocuments = 2+5+3+4 = 14
facets = merge(facets from all shards)
```

---

## Step 10: Return Response to Client

### BrokerResponse (proto)

```json
{
  "documents": [
    {
      "product_id": "P002",
      "name": "Pizza Deluxe",
      "price": 8.99
    },
    {
      "product_id": "P001",
      "name": "Pizza",
      "price": 12.99
    }
  ],
  "totalMatchedDocuments": 2,
  "matchedDocumentsPerNamespace": {
    "product": 2
  },
  "metrics": {
    "totalLatencyMs": 45,
    "brokerProcessingMs": 2,
    "searcherLatencyMs": 43
  }
}
```

Response is sent back to the **Client Application**.

---

## System Architecture Diagram

### Argo Search System Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                   DATA INGESTION PATH                         │
└───────────────────────────────────────────────────────────────┘

┌──────────────┐      ┌──────────────┐      ┌─────────────────┐
│ Data Source  │─────▶│ API Service  │─────▶│ High-Priority   │
│ (CRDB/etc)   │      │ (gRPC)       │      │ Kafka Queue     │
└──────────────┘      └──────────────┘      └────────┬────────┘
                                                      │
                                                      │ Consume
                                                      v
┌───────────────────────────────────────────────────────────────┐
│                      INDEXING PATH                            │
└───────────────────────────────────────────────────────────────┘

┌──────────────┐      ┌──────────────┐      ┌─────────────────┐
│ Data Source  │◀────▶│   Indexer    │─────▶│  Local Index    │
│              │ Read │              │Write │  (Lucene)       │
│ - CRDB       │      │ - Full Build │      └────────┬────────┘
│ - Snowflake  │      │ - Incremental│               │
│ - Cassandra  │      │   Updates    │               │ Push
└──────────────┘      └──────────────┘               │
                                                      v
                                              ┌─────────────────┐
                                              │  S3 Origin      │
                                              │  Repository     │
                                              └────────┬────────┘
                                                       │
                                                       │ Pull/Checkout
                                                       v
┌───────────────────────────────────────────────────────────────┐
│                       SEARCH PATH                             │
└───────────────────────────────────────────────────────────────┘

┌──────────────┐      ┌──────────────┐      ┌─────────────────┐
│   Client     │─────▶│    Broker    │─────▶│  Searcher 0     │
│  (gRPC)      │Search│  (gRPC)      │Fanout│  (Local Index)  │
└──────────────┘      │              │      └─────────────────┘
                      │ - Route      │      ┌─────────────────┐
                      │ - Scatter    │─────▶│  Searcher 1     │
                      │ - Gather     │      │  (Local Index)  │
                      │ - Merge      │      └─────────────────┘
                      └──────────────┘      ┌─────────────────┐
                                      └────▶│  Searcher 2     │
                                            │  (Local Index)  │
                                            └─────────────────┘
                                            ┌─────────────────┐
                                      └────▶│  Searcher 3     │
                                            │  (Local Index)  │
                                            └─────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                      CONTROL PLANE                            │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│ Control Plane                                                 │
│ - Manages deployments across generations                     │
│ - Coordinates index lifecycle                                │
│ - Monitors index readiness                                   │
└───────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

### Indexing

1. **Two-phase approach:** Full build + incremental updates
2. **Hierarchical indexing:** Parent-child relationships via foreign keys
3. **Sharding:** Data distributed across multiple physical shards via micro-sharding
4. **Durability:** S3-backed origin repository for disaster recovery
5. **Continuous updates:** Kafka queue for real-time data changes

### Search

1. **Broker pattern:** Central broker routes to distributed searchers
2. **Scatter-gather:** Parallel queries to multiple shards
3. **Smart routing:** Optional shard-specific queries to avoid fanout
4. **K-way merge:** Priority queue merges sorted results from shards
5. **Join queries:** Cross-namespace searches for complex relationships

### Performance

The system achieves:
- **High throughput** through horizontal scaling
- **Low latency** through parallel execution and smart routing
