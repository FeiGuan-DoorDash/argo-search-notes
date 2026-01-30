# Request Flow: Client → Broker → Searcher

Let me trace through a concrete example of a search request flowing through the system.

**Example Request:** "Search for pizza stores near Times Square"

---

## 1. Client → API Service (Optional Entry Point)

For search queries, clients typically call the Broker directly. The API Service is primarily for write operations (document updates).

### Client Code Example

```kotlin
val brokerClient = BrokerClient(...)
val request = BrokerRequest.newBuilder()
    .setNamespace("retail_stores")
    .setSearchQuery(SearchQuery.newBuilder()
        .setKeywords(Keywords.newBuilder()
            .setQuery("pizza")
            .build())
        .setFilterQuery(FilterQuery.newBuilder()
            .setGeoDistance(GeoDistance.newBuilder()
                .setField("location")
                .setLat(40.758)
                .setLon(-73.985)
                .setDistanceKm(2.0)
                .build())
            .build())
        .setLimit(20)
        .build())
    .build()

val response = brokerClient.broker(request)
```

---

## 2. Broker: Request Reception & Planning

### Step 2a: Parse and Plan

**File:** [broker/src/main/kotlin/com/doordash/argo/broker/service/BrokerServiceImpl.kt:79-130](broker/src/main/kotlin/com/doordash/argo/broker/service/BrokerServiceImpl.kt#L79-L130)

```kotlin
suspend fun process(request: BrokerRequest): BrokerResponse {
    // Convert protobuf to internal model
    val modelRequest = request.toModelBrokerRequest()
    // Example after conversion:
    // BrokerRequestModel(
    //   namespace = "retail_stores",
    //   searchQuery = SearchQueryModel(
    //     keywords = KeywordsModel(query = "pizza"),
    //     filterQuery = GeoDistanceModel(field = "location", lat = 40.758, ...),
    //     limit = 20
    //   )
    // )

    // Plan query - applies optimizations, rewrites
    val plannedQuery = queryPlanner.plan(modelRequest.searchQuery)
    // Query planner might:
    // - Add dedup strategy if configured
    // - Configure pruning at searcher level
    // - Set up reordering after L1 ranking
}
```

### Step 2b: Shard Selection & Routing

**File:** [broker/src/main/kotlin/com/doordash/argo/broker/service/SearcherClientSelector.kt:47-83](broker/src/main/kotlin/com/doordash/argo/broker/service/SearcherClientSelector.kt#L47-L83)

```kotlin
fun select(namespace: String, route: RouteSearchQueryModel?): List<SearcherClient> {
    val metadata = brokerIndexModel.namespaces[namespace]
    // metadata.sharding = MicroShardingMetadata(
    //   numberOfMicroShards = 1320,
    //   microShardIdSource = "store_id"
    // )

    val totalShards = 45  // from StackInfo
    val totalMicroShards = 1320

    // If route is null → fan out to ALL shards
    // If route = RouteByKey("store_24679098") → calculate specific shard

    val shardIds = if (route != null) {
        route.shards(totalShards, totalMicroShards)
        // For key "store_24679098":
        // 1. microShardId = calculateMicroShardId("store_24679098", 1320) = 612
        // 2. Map to shard: shardId = 20 (using formula from previous answer)
        // Result: [20]
    } else {
        (0 until totalShards).toList()
        // Result: [0, 1, 2, ..., 44] - ALL shards
    }

    return shardIds.map { shardId -> clients[namespace]!![shardId] }
    // Returns: [SearcherClient(host="searcher-0"), SearcherClient(host="searcher-1"), ...]
}
```

**For our example (no specific routing):** All 45 searcher clients are selected.

### Step 2c: Fanout to Searchers

**File:** [broker/src/main/kotlin/com/doordash/argo/broker/service/BrokerServiceImpl.kt:167-242](broker/src/main/kotlin/com/doordash/argo/broker/service/BrokerServiceImpl.kt#L167-L242)

```kotlin
private suspend fun fanout(
    searcherRequest: SearcherRequestModel,
    clients: List<SearcherClient>
): List<BrokerSearcherResponse> {

    // Build protobuf request
    val searcherRequestProto = searcherRequest.toProtoSearcherRequest()

    // Launch parallel requests to all searchers
    return coroutineScope {
        clients.map { client ->
            async(Dispatchers.IO) {
                // gRPC call to searcher
                val response = client.searcher(searcherRequestProto)

                // Handle compression if used
                val documents = when (response.format) {
                    FLAT_NORMALIZED -> response.documentCollection
                    FLAT_NORMALIZED_COMPRESSED -> {
                        // Decompress LZ4
                        LZ4Factory.fastestInstance()
                            .fastDecompressor()
                            .decompress(response.documentCollectionCompressed.toByteArray())
                    }
                }

                BrokerSearcherResponse(
                    searchResults = documents.toModelSearchResults(...),
                    shardId = client.shardId
                )
            }
        }.awaitAll()  // Wait for all 45 searchers to respond
    }
}
```

#### Parallel Execution Example

```
Time 0ms:  Launch requests to searchers 0-44 in parallel
Time 15ms: Searcher 0 responds (5 results)
Time 18ms: Searcher 1 responds (12 results)
Time 20ms: Searcher 2 responds (3 results)
...
Time 45ms: All 45 searchers have responded
           Total results: 278 documents across all shards
```

---

## 3. Searcher: Query Execution

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/grpc/SearcherService.kt:72-138](searcher/src/main/kotlin/com/doordash/argo/searcher/grpc/SearcherService.kt#L72-L138)

Each searcher receives the request independently.

### Step 3a: Request Handling

```kotlin
override suspend fun searcher(request: SearcherRequest): SearcherResponse {
    // Track concurrency
    searchesInFlight.incrementAndGet()

    // Convert protobuf to model
    val searchQuery = request.toModelSearcherRequest()

    // Execute search with thread pool
    val results = withContext(threadPoolDispatcher) {
        argoSearcher.search(
            dispatcher = threadPoolDispatcher,
            searchQuery = searchQuery,
            squash = true
        )
    }

    // Build response
    return SearcherResponseBuilder.build(
        results = results,
        format = FLAT_NORMALIZED_COMPRESSED,  // Use LZ4 compression
        includeMetrics = request.includeMetrics
    )
}
```

### Step 3b: Query Building

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/query/LuceneQueryBuilder.kt:54-97](searcher/src/main/kotlin/com/doordash/argo/searcher/query/LuceneQueryBuilder.kt#L54-L97)

```kotlin
fun build(searchQuery: SearchQueryModel): Query {
    val booleanQuery = BooleanQuery.Builder()

    // 1. Build filter query (GeoDistance)
    val geoQuery = LatLonPoint.newDistanceQuery(
        field = "location",
        latitude = 40.758,
        longitude = -73.985,
        radiusMeters = 2000.0
    )
    booleanQuery.add(geoQuery, BooleanClause.Occur.FILTER)

    // 2. Build keyword query (BM25 scoring)
    val keywordQuery = when (searchQuery.keywords.matchType) {
        BEST_FIELDS -> {
            // Search "pizza" across multiple fields
            MultiFieldQuery(
                fields = ["name^2.0", "description^1.0", "categories^1.5"],
                query = "pizza",
                analyzer = StandardAnalyzer()
            )
        }
    }
    booleanQuery.add(keywordQuery, BooleanClause.Occur.SHOULD)

    return booleanQuery.build()
    // Result: (location:[within 2km of Times Square]) AND (pizza in name/description/categories)
}
```

### Step 3c: Lucene Search Execution

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt:120-180](searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt#L120-L180)

```kotlin
private suspend fun matchAndRank(
    searchQuery: SearchQueryModel
): MatchAndRankResults {

    // Build Lucene query
    val luceneQuery = luceneQueryBuilder.build(searchQuery)

    // Configure collector
    val topDocsCollector = TopScoreDocCollector.create(
        numHits = 20,  // limit from request
        totalHitsThreshold = 1000
    )

    // Execute search on Lucene index
    indexSearcher.search(luceneQuery, topDocsCollector)

    // Get top results
    val topDocs = topDocsCollector.topDocs()
    // topDocs.scoreDocs = [
    //   ScoreDoc(doc=12345, score=8.5),  // "Joe's Pizza"
    //   ScoreDoc(doc=45678, score=7.2),  // "Best Pizza Place"
    //   ScoreDoc(doc=78901, score=6.8),  // "Pizza Heaven"
    //   ...
    // ]

    return MatchAndRankResults(
        scoreDocs = topDocs.scoreDocs,
        totalHits = topDocs.totalHits.value
    )
}
```

### Step 3d: Document Hydration

```kotlin
private suspend fun hydrate(
    matchResults: MatchAndRankResults,
    requestedFields: List<String>
): List<ArgoDocument> {

    return matchResults.scoreDocs.map { scoreDoc ->
        val doc = indexSearcher.storedFields().document(scoreDoc.doc)

        ArgoDocument(
            primaryKey = doc.get("store_id"),  // "store_12345"
            fields = mapOf(
                "name" to "Joe's Pizza",
                "address" to "123 W 42nd St",
                "location" to GeoPoint(40.757, -73.986),
                "rating" to 4.5,
                "price_range" to "$$"
            ),
            score = scoreDoc.score,
            sortByValues = listOf(scoreDoc.score)  // For L1 ranking
        )
    }
}
```

### Step 3e: Build Response

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/grpc/SearcherResponseBuilder.kt:32-85](searcher/src/main/kotlin/com/doordash/argo/searcher/grpc/SearcherResponseBuilder.kt#L32-L85)

```kotlin
fun build(results: ArgoResults): SearcherResponse {
    val documentCollection = DocumentCollection.newBuilder()
        .addAllDocuments(results.documents.map { doc ->
            Document.newBuilder()
                .setPrimaryKey(doc.primaryKey)
                .putAllFields(doc.fields.toProto())
                .addAllSortByValues(doc.sortByValues)
                .build()
        })
        .build()

    // Compress with LZ4
    val compressor = LZ4Factory.fastestInstance().fastCompressor()
    val compressed = compressor.compress(documentCollection.toByteArray())

    return SearcherResponse.newBuilder()
        .setFormat(FLAT_NORMALIZED_COMPRESSED)
        .setDocumentCollectionCompressed(ByteString.copyFrom(compressed))
        .setTotalHits(results.totalHits)
        .build()
}
```

#### Response from Searcher #20

```kotlin
SearcherResponse {
  totalHits: 12
  documentCollectionCompressed: [LZ4 compressed bytes]
  // After decompression:
  // Document("store_12345", name="Joe's Pizza", score=8.5)
  // Document("store_45678", name="Best Pizza Place", score=7.2)
  // ...
}
```

---

## 4. Broker: Response Aggregation

**File:** [broker/src/main/kotlin/com/doordash/argo/broker/service/BrokerServiceImpl.kt:244-370](broker/src/main/kotlin/com/doordash/argo/broker/service/BrokerServiceImpl.kt#L244-L370)

### Step 4a: Deduplication

```kotlin
private fun dedup(
    searchResults: List<SearchResults>,
    dedupStrategy: DedupStrategy
): List<SearchResults> {

    // Collect all documents from all 45 shards
    val allDocuments = searchResults.flatMap { it.documents }
    // Total: 278 documents across all shards

    // Group by primary key
    val dedupedDocs = allDocuments
        .groupBy { it.primaryKey }
        .mapValues { (_, docs) ->
            // Keep highest scoring version
            docs.maxByOrNull { it.score }!!
        }
        .values.toList()

    // Result: 245 unique documents (33 duplicates removed)
}
```

### Step 4b: Merging & Sorting

```kotlin
private suspend fun merge(
    searchResults: List<SearchResults>,
    searchQuery: SearchQuery,
    limit: Int
): SearchResults {

    // All documents are already scored and sorted by each searcher
    // Now merge-sort across all shards

    val mergedDocs = searchResults
        .flatMap { it.documents }
        .sortedByDescending { it.score }  // Global sort
        .take(limit)  // Take top 20

    // Result:
    // Document("store_12345", name="Joe's Pizza", score=8.5)         [from shard 20]
    // Document("store_99999", name="Pizza Paradise", score=8.3)      [from shard 5]
    // Document("store_45678", name="Best Pizza Place", score=7.2)    [from shard 20]
    // ... (17 more)
}
```

### Step 4c: Reordering (L2 Ranking)

```kotlin
private suspend fun reordering(
    searchResults: SearchResults,
    reorderings: List<Reordering>
): SearchResults {

    // Apply post-processing reordering rules
    // Example: Boost documents with high ratings

    val reranked = searchResults.documents.map { doc ->
        var newScore = doc.score

        // Apply boosts
        if (doc.fields["rating"] as Double > 4.0) {
            newScore *= 1.2  // 20% boost
        }
        if (doc.fields["is_open_now"] as Boolean) {
            newScore *= 1.1  // 10% boost
        }

        doc.copy(score = newScore)
    }.sortedByDescending { it.score }

    // Result: Rankings adjusted based on business logic
}
```

### Step 4d: Facet Aggregation

```kotlin
private fun mergeFacets(
    searchResults: List<SearchResults>
): Map<String, FacetResult> {

    // Aggregate facet counts across all shards
    val facetCounts = mutableMapOf<String, MutableMap<String, Long>>()

    searchResults.forEach { result ->
        result.facets.forEach { (field, buckets) ->
            buckets.forEach { (value, count) ->
                facetCounts
                    .getOrPut(field) { mutableMapOf() }
                    .merge(value, count, Long::plus)
            }
        }
    }

    // Result:
    // "price_range" -> {"$": 45, "$$": 123, "$$$": 77}
    // "rating" -> {"4+": 156, "3-4": 89}
}
```

### Step 4e: Build Final Response

```kotlin
private fun buildResponse(
    searchResults: SearchResults,
    facets: Map<String, FacetResult>
): BrokerResponse {

    return BrokerResponse.newBuilder()
        .setTotalHits(245)
        .addAllDocuments(searchResults.documents.map { it.toProto() })
        .putAllFacets(facets.mapValues { it.value.toProto() })
        .setMetrics(Metrics.newBuilder()
            .setSearcherTimeMs(45)  // Max time across all searchers
            .setBrokerTimeMs(52)    // Total broker processing time
            .build())
        .build()
}
```

---

## 5. Final Response to Client

```kotlin
val response = brokerClient.broker(request)

// Response structure:
BrokerResponse {
  totalHits: 245
  documents: [
    Document {
      primaryKey: "store_12345"
      fields: {
        "name": "Joe's Pizza",
        "address": "123 W 42nd St",
        "location": {lat: 40.757, lon: -73.986},
        "rating": 4.5,
        "price_range": "$$"
      }
      score: 10.2  // After L2 reranking
    },
    Document {
      primaryKey: "store_99999"
      fields: {
        "name": "Pizza Paradise",
        ...
      }
      score: 9.96
    },
    ... (18 more documents)
  ]
  facets: {
    "price_range": {
      "$": 45,
      "$$": 123,
      "$$$": 77
    },
    "rating": {
      "4+": 156,
      "3-4": 89
    }
  }
  metrics: {
    searcherTimeMs: 45
    brokerTimeMs: 52
  }
}
```

---

## Summary Timeline

```
0ms:   Client sends BrokerRequest to Broker
1ms:   Broker parses request and plans query
2ms:   Broker selects 45 searchers (all shards)
3ms:   Broker fans out to all 45 searchers in parallel
├─ 18ms:  Searcher 0 executes Lucene search, returns 6 results
├─ 20ms:  Searcher 1 executes Lucene search, returns 8 results
├─ ...
└─ 45ms:  All 45 searchers have responded
48ms:  Broker dedups 278 docs → 245 unique docs
49ms:  Broker merges and sorts all results
50ms:  Broker applies L2 reranking
51ms:  Broker aggregates facets across shards
52ms:  Broker returns final BrokerResponse to client
```

---

## Key Insights

This demonstrates the complete **scatter-gather architecture** where the Broker orchestrates parallel queries across multiple Searchers and aggregates the results!

### Architecture Highlights

1. **Parallel Fanout**: Broker queries all 45 shards simultaneously
2. **Compression**: LZ4 compression reduces network overhead
3. **Deduplication**: Handles documents that may appear in multiple shards
4. **Two-Level Ranking**:
   - **L1**: Lucene BM25 scoring at the searcher level
   - **L2**: Business logic reranking at the broker level
5. **Facet Aggregation**: Combines facet counts across all shards
6. **Efficient Merging**: Merge-sort algorithm combines pre-sorted results from each shard
