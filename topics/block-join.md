# Block Join in Argo Search

> **Important Note:** Argo Search does NOT use traditional Lucene BlockJoin (ToParentBlockJoin/ToChildBlockJoin queries). Instead, it implements a **foreign key-based parent-child relationship system** that achieves similar results with more flexibility.

---

## 1. What is "Block Join" in Argo?

In Argo, "block join" refers to the ability to:

1. Define parent-child relationships between documents in different namespaces
2. Execute queries that filter parents based on child criteria
3. Return parent documents with their matching children attached

This is achieved through **foreign keys** rather than Lucene's block structure.

---

## 2. Key Concepts

### The ArgoDocument Structure

```kotlin
data class ArgoDocument(
    val docId: Int,              // Global document ID (across all segments)
    val leafDocId: Int,          // Document ID within a Lucene segment
    val leafOrd: Int,            // Segment/leaf reader ordinal
    val primaryKey: StringFieldValue,
    val children: List<ListOfDocumentsFieldValue>,  // ← Child documents
    val sortByFields: List<Double>,
    val returnFields: MutableList<FieldValue>,
    ...
)
```

### Key Fields

| Field | Description |
|-------|-------------|
| `leafDocId` | Document ID within its specific Lucene segment (for efficient field access) |
| `leafOrd` | Which segment this document belongs to |
| `children` | List of child documents from related namespaces |

---

## 3. Configuration Example: Parent-Child Relationship

### Scenario: Stores and Items

**Store (Parent)**

```json
{
  "id": "store_42",
  "name": "Joe's Restaurant",
  "rating": 4.5
}
```

**Item (Child)**

```json
{
  "id": "item_123",
  "name": "Margherita Pizza",
  "store_id": "store_42"  // ← Foreign key
}
```

### Schema Configuration

#### Item Schema

**File:** `configurations/food-items/prod/item/schema.libsonnet`

```jsonnet
{
  schema: {
    primary_key: {
      name: "id",
      source: "id"
    },

    foreign_keys: [
      {
        name: "store_id",           // Foreign key field
        source: {
          name: "store_id",
          container_data_type: "scalar"  // Single value
        },
        children: ["store"]         // References "store" namespace
      }
    ],

    fields: [
      { name: "name", field_type: "text" },
      { name: "price", field_type: "double" }
    ]
  }
}
```

#### Store Schema

**File:** `configurations/stores/prod/store/schema.libsonnet`

```jsonnet
{
  schema: {
    primary_key: {
      name: "id",
      source: "id"
    },

    fields: [
      { name: "name", field_type: "text" },
      { name: "rating", field_type: "double" }
    ]
  }
}
```

---

## 4. How Documents are Indexed

### Indexing an Item Document

**File:** [core/src/main/kotlin/com/doordash/argo/core/index/ForeignKeyFieldModel.kt:43-70](core/src/main/kotlin/com/doordash/argo/core/index/ForeignKeyFieldModel.kt#L43-L70)

When indexing `item_123` with `store_id="store_42"`:

```java
// Foreign key field generates THREE Lucene fields:

1. StringField("item:store_id", "store_42", Field.Store.YES)
   // Stores the foreign key value for retrieval

2. SortedSetDocValuesField("item:store_id", BytesRef("store_42"))
   // Fast in-memory access without loading stored fields

3. StringField("ref:primary_key:store", "store_42", Field.Store.NO)
   // Reference term for reverse lookups
   // Format: ref:primary_key:{child_namespace}
```

### Why Three Fields?

1. **StringField**: For exact matching in queries
2. **DocValues**: For efficient iteration during collection
3. **Reference term**: For linking to child documents

---

## 5. Query Example: Find Items from High-Rated Stores

**Use Case:** "Find all pizza items from stores with rating ≥ 4.0"

### Query Structure

```json
{
  "namespace": "item",
  "keywords": {
    "query": "pizza"
  },
  "join": {
    "inner_search_queries": [
      {
        "namespace": "store",
        "filter": {
          "range_query": {
            "field": "rating",
            "lower_bound": 4.0
          }
        },
        "return_fields": ["id", "name", "rating"],
        "limit": 1000
      }
    ]
  },
  "return_fields": ["id", "name", "price", "store_id"],
  "limit": 20
}
```

---

## 6. Execution Flow: Step-by-Step

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt](searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt)

### Step 1: Execute Inner (Child) Query

```kotlin
// Execute store query first
val innerQueryResults = matchAndRankJoinQueries(searchQuery)

// Results: Stores with rating >= 4.0
// ArgoResults(
//   documents = [
//     ArgoDocument(primaryKey = "store_42", fields = {name: "Joe's", rating: 4.5}),
//     ArgoDocument(primaryKey = "store_99", fields = {name: "Best Pizza", rating: 4.8}),
//     ArgoDocument(primaryKey = "store_117", fields = {name: "Pizza Place", rating: 4.2})
//   ]
// )
```

### Step 2: Create Join Clause

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt:218-231](searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt#L218-L231)

```kotlin
private fun makeJoinClause(
    outerSearchQuery: SearchQuery,
    innerPrimaryKeys: List<StringFieldValue>,  // ["store_42", "store_99", "store_117"]
    innerSearchQuery: SearchQuery               // namespace: "store"
): BooleanClause {

    // Find foreign key field in item namespace that references store
    val foreignKeyField = indexModel.getForeignKeyField(
        parentNamespace = "item",
        childNamespace = "store"
    )
    // Returns: ForeignKeyFieldModel(name = "store_id", children = ["store"])

    // Create TermInSetQuery: store_id IN ["store_42", "store_99", "store_117"]
    val joinQuery = TermInSetQuery(
        field = "item:store_id",
        terms = [BytesRef("store_42"), BytesRef("store_99"), BytesRef("store_117")]
    )

    return BooleanClause(joinQuery, BooleanClause.Occur.FILTER)
}
```

### Step 3: Execute Outer (Parent) Query

```kotlin
// Build final Lucene query
BooleanQuery(
  SHOULD: KeywordQuery("pizza"),                    // Score by relevance
  FILTER: TermInSetQuery(                           // Only from these stores
    field = "item:store_id",
    terms = ["store_42", "store_99", "store_117"]
  )
)

// Execute search on item namespace
// Results:
//   item_123 (name: "Margherita Pizza", store_id: "store_42", score: 8.5)
//   item_456 (name: "Pepperoni Pizza", store_id: "store_99", score: 7.2)
//   item_789 (name: "Hawaiian Pizza", store_id: "store_42", score: 6.8)
```

### Step 4: Attach Child Documents

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/search/MatchingAndRankingCollector.kt:397-422](searcher/src/main/kotlin/com/doordash/argo/searcher/search/MatchingAndRankingCollector.kt#L397-L422)

During collection, for each matched item document:

```kotlin
private fun getChildDocuments(docId: Int): List<ListOfDocumentsFieldValue> {

    return foreignKeysBound.map { fkBound ->
        // fkBound contains:
        // - foreignKeyModel: ForeignKeyFieldModel(name="store_id", children=["store"])
        // - innerDocsByPk: Map of store ArgoDocuments keyed by primary key
        // - docValues: SortedSetDocValues for "item:store_id"

        // Step 1: Read foreign key value from this document
        val docValues = fkBound.docValues
        if (!docValues.advanceExact(docId)) {
            return@map ListOfDocumentsFieldValue("store", emptyList())
        }

        // Step 2: Get foreign key values (ordinals in the sorted set)
        val foreignKeyOrd = docValues.nextOrd()  // Example: ordinal 42
        val foreignKeyValue = docValues.lookupOrd(foreignKeyOrd)  // "store_42"

        // Step 3: Lookup child document in pre-fetched results
        val childDoc = fkBound.innerDocsByPk[StringFieldValue("store_42")]
        // Returns: ArgoDocument(
        //   primaryKey = "store_42",
        //   returnFields = [
        //     StringFieldValue("name", "Joe's Restaurant"),
        //     DoubleFieldValue("rating", 4.5)
        //   ]
        // )

        // Step 4: Return child documents wrapped
        ListOfDocumentsFieldValue(
            namespace = "store",
            documents = listOf(childDoc)
        )
    }
}
```

### Final ArgoDocument Structure

```kotlin
// Final ArgoDocument for item_123:
ArgoDocument(
    docId = 12345,
    leafDocId = 123,       // Position in segment
    leafOrd = 0,           // Segment ID
    primaryKey = StringFieldValue("item_123"),
    children = [
        ListOfDocumentsFieldValue(
            namespace = "store",
            documents = [
                ArgoDocument(
                    primaryKey = StringFieldValue("store_42"),
                    returnFields = [
                        StringFieldValue("name", "Joe's Restaurant"),
                        DoubleFieldValue("rating", 4.5)
                    ]
                )
            ]
        )
    ],
    returnFields = [
        StringFieldValue("name", "Margherita Pizza"),
        DoubleFieldValue("price", 12.99),
        StringFieldValue("store_id", "store_42")
    ]
)
```

---

## 7. ForeignKeyBound: The Linking Mechanism

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/search/MatchingAndRankingCollector.kt:890-896](searcher/src/main/kotlin/com/doordash/argo/searcher/search/MatchingAndRankingCollector.kt#L890-L896)

```kotlin
class ForeignKeyBound(
    val foreignKeyModel: ForeignKeyFieldModel,
    val foreignPrimaryKeyModel: PrimaryKeyFieldModel,
    val docValues: SortedSetDocValues?,           // Foreign key doc values
    val innerDocsByPk: Map<StringFieldValue, ArgoDocument>,  // Pre-fetched children
    val ordToDocCache: MutableMap<Long, ArgoDocument?> = mutableMapOf()
)
```

### Purpose

- **Binds** parent documents to their children during collection
- **docValues**: Provides fast access to foreign key values without loading stored fields
- **innerDocsByPk**: All child documents from the join query, keyed by primary key
- **ordToDocCache**: Caches lookups for performance

### Example

```kotlin
ForeignKeyBound(
    foreignKeyModel = ForeignKeyFieldModel(name="store_id", children=["store"]),
    foreignPrimaryKeyModel = PrimaryKeyFieldModel(namespace="store", name="id"),
    docValues = SortedSetDocValues("item:store_id"),
    innerDocsByPk = {
        "store_42" -> ArgoDocument(primaryKey="store_42", ...),
        "store_99" -> ArgoDocument(primaryKey="store_99", ...),
        "store_117" -> ArgoDocument(primaryKey="store_117", ...)
    }
)
```

---

## 8. Document Hydration Using leafDocId and leafOrd

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt:303-359](searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt#L303-L359)

After matching documents, hydration loads their stored fields:

```kotlin
private suspend fun hydrate(
    results: ArgoResults
) {
    val resultDocs = results.documents.filter { it.requiresHydration }

    // Group by segment (leafOrd) for efficient access
    val perLeafDocs = resultDocs.groupBy { it.leafOrd }
    // Example: {
    //   0 -> [doc1, doc3, doc5],  // Segment 0
    //   1 -> [doc2, doc7],         // Segment 1
    //   2 -> [doc4, doc6, doc8]    // Segment 2
    // }

    // For each segment
    perLeafDocs.forEach { (leafOrd, docs) ->
        val leafReader = indexReader.leaves()[leafOrd].reader()

        // Sort by leafDocId for sequential disk access
        docs.sortBy { it.leafDocId }

        docs.forEach { doc ->
            // Use leafDocId to fetch stored fields from this segment
            val storedFields = leafReader.storedFields()
                .document(doc.leafDocId, requestedFields)

            // Use leafDocId for doc values access
            val nameDocValues = leafReader.getBinaryDocValues("name")
            if (nameDocValues.advanceExact(doc.leafDocId)) {
                doc.returnFields.add(
                    StringFieldValue("name", nameDocValues.binaryValue().utf8ToString())
                )
            }

            val priceDocValues = leafReader.getNumericDocValues("price")
            if (priceDocValues.advanceExact(doc.leafDocId)) {
                doc.returnFields.add(
                    DoubleFieldValue("price", Double.fromBits(priceDocValues.longValue()))
                )
            }
        }
    }
}
```

### Why leafDocId and leafOrd?

- Lucene indexes are split into **segments** (leaves)
- Each segment has its own document IDs starting from 0
- **leafOrd** identifies which segment
- **leafDocId** is the position within that segment
- Together they allow efficient field retrieval from the correct segment reader

---

## 9. Final Response Structure

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/grpc/SearcherResponseBuilder.kt:146-150](searcher/src/main/kotlin/com/doordash/argo/searcher/grpc/SearcherResponseBuilder.kt#L146-L150)

```kotlin
// Build response with child document offsets
collection.addPrimaryKeys(doc.primaryKey.value)
doc.children.forEach { childDocs ->
    val builder = ChildDocumentOffsets.newBuilder()
    childDocs.value.forEach { childDoc ->
        builder.addOffset(childDoc.resultOffset)  // Position in child results array
    }
    collection.addChildDocumentOffsets(builder)
}
```

### Response Example

```json
{
  "documents": [
    {
      "primary_key": "item_123",
      "fields": {
        "name": "Margherita Pizza",
        "price": 12.99,
        "store_id": "store_42"
      },
      "child_document_offsets": [
        {"offsets": [0]}  // Points to store at index 0 in child results
      ]
    }
  ],
  "child_documents": [
    {
      "namespace": "store",
      "documents": [
        {
          "primary_key": "store_42",
          "fields": {
            "name": "Joe's Restaurant",
            "rating": 4.5
          }
        }
      ]
    }
  ]
}
```

---

## 10. Multi-Level Foreign Keys

You can have multiple foreign keys creating complex hierarchies:

```jsonnet
// item schema
foreign_keys: [
  { name: "store_id", children: ["store"] },
  { name: "menu_id", children: ["menu"] },
  { name: "category_id", children: ["category"] }
]
```

### Query with Multiple Joins

```json
{
  "namespace": "item",
  "keywords": {"query": "pizza"},
  "join": {
    "inner_search_queries": [
      {
        "namespace": "store",
        "filter": {"range_query": {"field": "rating", "lower_bound": 4.0}}
      },
      {
        "namespace": "menu",
        "filter": {"term_query": {"field": "type", "value": "dinner"}}
      },
      {
        "namespace": "category",
        "filter": {"term_query": {"field": "cuisine", "value": "italian"}}
      }
    ]
  }
}
```

### This Creates

```
item_123
├─ store_42 (Joe's Restaurant, rating: 4.5)
├─ menu_7 (Dinner Menu)
└─ category_salads (Italian Cuisine)
```

---

## Summary: Argo's Block Join Implementation

| Aspect | Implementation |
|--------|----------------|
| **Relationship Definition** | Foreign keys in schema configuration |
| **Index Structure** | Flat index with linked foreign key fields |
| **Query Strategy** | Execute child queries first, filter parents by results |
| **Parent-Child Linking** | Via ForeignKeyBound using doc values lookup |
| **Document Structure** | ArgoDocument with children list |
| **Efficiency** | DocValues for fast foreign key access, leafDocId/leafOrd for segment-aware hydration |
| **Flexibility** | Multiple foreign keys, list-valued foreign keys, multi-level joins |

---

## Key Insight

> **Argo implements parent-child relationships without Lucene's BlockJoin, using foreign keys and doc values for more flexible querying!**

### Advantages over Traditional BlockJoin

1. **Flexibility**: Can query parent or child namespace independently
2. **Multiple Relationships**: Support multiple foreign keys per document
3. **Dynamic Filtering**: Join queries can have complex filters
4. **Simpler Indexing**: No need for block structure in index
5. **List Foreign Keys**: Support many-to-many relationships

### Trade-offs

- **Join Query Cost**: Inner queries must execute first
- **Memory Usage**: Child documents held in memory during collection
- **Join Size Limits**: Large join results can impact performance
