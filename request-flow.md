# Search Request Flow Overview: Broker + Multiple Searcher Shards

The Argo search system uses a **scatter-gather pattern** where:
- **Broker** = Coordinator that routes and merges requests
- **Searchers** = Workers, each holding a shard of the index

## How Request Flow Works

```
Client Request
      ↓
   Broker (BrokerServiceImpl)
      ↓
SearcherClientSelector (determines which shards to query)
      ↓
   Fanout (parallel gRPC calls)
      ↓
[Searcher Shard 0] [Searcher Shard 1] ... [Searcher Shard N]
      ↓
   Merge Results
      ↓
  Return to Client
```

## Concrete Example from BrokerWithTwoShardsTest

### 1. Data Sharding During Indexing

```kotlin
// Build two separate index shards
val indexPathShard1 = IndexerFixtures.fullBuildSmallFoodItemIndex(
    shards = 2,  // Total number of shards
    shardId = 0   // This searcher gets shard 0
)

val indexPathShard2 = IndexerFixtures.fullBuildSmallFoodItemIndex(
    shards = 2,   // Total number of shards  
    shardId = 1   // This searcher gets shard 1
)
```

**How data gets distributed:**
- Documents are sharded using `micro_shard_id % numberOfShards`
- Example: If menu item has `micro_shard_id = 372`, with 2 shards: `372 % 2 = 0` → goes to Shard 0
- Example: If menu item has `micro_shard_id = 759`, with 2 shards: `759 % 2 = 1` → goes to Shard 1

**From your documents.db:**

```
-- Shard 0 gets these items (micro_shard_id % 2 == 0):
item_id: 687029697, micro_shard_id: 18    (18 % 2 = 0)
item_id: 690422833, micro_shard_id: 1187  (1187 % 2 = 1, so actually Shard 1)
item_id: 690422846, micro_shard_id: 1000  (1000 % 2 = 0)

-- Shard 1 gets these items (micro_shard_id % 2 == 1):
item_id: 420110344, micro_shard_id: 759   (759 % 2 = 1)
item_id: 687029698, micro_shard_id: 1073  (1073 % 2 = 1)
```

### 2. Broker Setup with Multiple Searcher Clients

```kotlin
// Create two searcher clients, one per shard
val client1 = SearcherClient(shardId = 0, searcher1Stub)
val client2 = SearcherClient(shardId = 1, searcher2Stub)

// Broker knows about both clients
val brokerServiceImpl = BrokerServiceImpl(
    listOf(client1, client2),  // List of all searcher clients
    indexMetadata,
    stackConfig,
    "generation_P"
)
```

### 3. Request Routing

When a request comes in, the broker decides which searchers to query:

```kotlin
fun select(namespace: String, route: RouteSearchQueryModel?): List<SearcherClient> {
    if (route == null) {
        return clients  // Query ALL shards (scatter to all)
    }

    // If route is specified, only query specific shards
    val shards = route.shards(numberOfShards, numberOfMicroShards)
    return shards.map { shardId -> clients[shardId] }
}
```

**Example scenarios:**

#### Scenario A: No routing (most common)

```kotlin
// Request has no routing info
broker.broker(BrokerFixtures.foodItemJoinWithStoreRequest())

// Broker queries ALL searchers: [Shard 0, Shard 1]
```

#### Scenario B: With routing (targeted query)

```kotlin
// Request specifies to only query specific micro-shards
route = RouteSearchQueryModel(microShardIds = [18, 372, 1000])

// With 2 shards and these micro_shard_ids:
// - 18 % 2 = 0   → Shard 0
// - 372 % 2 = 0  → Shard 0  
// - 1000 % 2 = 0 → Shard 0

// Broker only queries: [Shard 0]
```

### 4. Fanout to Searchers

```kotlin
private suspend fun fanout(
    searcherRequest: SearcherRequestModel,
    route: RouteSearchQueryModel?
): List<BrokerSearcherResponse> {

    // Select which searchers to query
    val responses = searcherSelector
        .select(searcherRequest.searchQuery.namespace, route)
        .map { searcherClient ->
            // Launch parallel async gRPC call to each searcher
            CoroutineScope(Dispatchers.IO).async {
                val searcherResponseProto = searcherClient.searcher(searcherRequestProto)
                BrokerSearcherResponse(
                    shard = searcherClient.shardId,
                    collection = searcherResponseProto.searchResultsFlatNormalized,
                    facets = searcherResponseProto.facets
                )
            }
        }
        .awaitAll()  // Wait for all parallel calls to complete

    return successfulResponses
}
```

**Example with 2 shards:**

```
Time 0ms:  Broker receives request "search for chicken tikka"
Time 1ms:  Broker fans out to BOTH searchers in parallel:
           ├─ gRPC call → Searcher Shard 0 (async)
           └─ gRPC call → Searcher Shard 1 (async)

Time 50ms: Shard 0 responds with 15 results from its data
Time 55ms: Shard 1 responds with 12 results from its data

Time 56ms: Broker receives both responses via awaitAll()
```

### 5. Each Searcher Processes Independently

```kotlin
override suspend fun searcher(request: SearcherRequest): SearcherResponse {
    // Each searcher only searches its local shard of data
    val searchResults = searcher.search(
        searchQuery = searchQuery,
        // Only searches local Lucene index for this shard
    )
    return response
}
```

**What each searcher sees:**

**Searcher Shard 0** (has items with `micro_shard_id % 2 == 0`):
```
Query: "tikka masala"
Local Lucene Index Search:
  - item_id: 687029707, name: "Tikka Masala Bowl"
  - item_id: 690422836, name: "Chicken Tikka Masala 16oz"
  - item_id: 690422839, name: "Tikka Masala Burrito"
Response: 3 results
```

**Searcher Shard 1** (has items with `micro_shard_id % 2 == 1`):
```
Query: "tikka masala"
Local Lucene Index Search:
  - (searches its own data, may find 0 or more matching items)
Response: 1 result
```

### 6. Broker Merges Results

```kotlin
// After fanout completes
val searcherResponses = fanout(searcherRequest, route)

// Merge results from all shards
val collections = searcherResponses.map { it.collection }

// Deduplicate across shards (if needed)
dedup(searchQuery, documents, collections)

// Merge facets (sum counts from all shards)
val mergedFacets = mergeFacets(searcherResponses.map { it.facets })

// Sort and paginate the global result set
// Return top K results to client
```

**Example merge:**

```
Shard 0 results: [doc1(score=10), doc2(score=8), doc3(score=6)]
Shard 1 results: [doc4(score=9), doc5(score=7), doc6(score=5)]

After merge + sort:
  [doc1(score=10), doc4(score=9), doc2(score=8), doc5(score=7), doc6(score=6), doc3(score=6)]

With limit=5:
  [doc1, doc4, doc2, doc5, doc6]
```

## Error Handling

The broker handles partial failures gracefully:

```kotlin
// If some searchers fail:
if (successfulResponses.isEmpty()) {
    throw IllegalStateException("All Searchers failed")
}

// Otherwise, continue with partial results
// This allows degraded operation if one shard is down
```

**Example:** If Shard 1 fails but Shard 0 succeeds:
- Broker returns results from Shard 0 only
- Logs error for Shard 1
- Client gets partial results (better than total failure)

## Key Takeaways

1. **Data is pre-sharded** during indexing using `micro_shard_id % numberOfShards`
2. **Broker fans out to ALL shards** by default (unless routing specified)
3. **Searchers work independently** on their local shard data
4. **Parallelism** via Kotlin coroutines (`async`/`awaitAll`)
5. **Broker merges results** - dedup, sort, paginate, merge facets
6. **Graceful degradation** - partial results if some shards fail

This architecture enables **horizontal scaling** - add more shards to handle larger datasets, and the broker automatically distributes load across them.
