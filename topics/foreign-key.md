# Foreign Keys in Argo Search: Explained with Examples

---

## 1. Basic Concept: Document Relationships

Think of foreign keys like database relationships. You have **parent documents** that reference **child documents**.

### Example Scenario: Restaurant Menu System

#### Namespace: item (Food Items)

```json
{
  "id": "item_123",
  "name": "Margherita Pizza",
  "price": 12.99,
  "store_id": "store_42"  // ← FOREIGN KEY pointing to Store
}
```

#### Namespace: store (Restaurants)

```json
{
  "id": "store_42",  // ← PRIMARY KEY
  "name": "Joe's Italian Restaurant",
  "address": "123 Main St",
  "rating": 4.5
}
```

**Relationship:** `item.store_id` → `store.id`

---

## 2. Configuration: Defining Foreign Keys

### Example 1: Simple Foreign Key (Scalar - Single Value)

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
        name: "store_id",              // Foreign key field name
        source: {
          name: "store_id",            // Source field in raw data
          required: true                // Must be present
        },
        children: ["store"]             // References "store" namespace
      }
    ],

    fields: [
      {
        name: "name",
        field_type: "text"
      },
      {
        name: "price",
        field_type: "double"
      }
    ]
  }
}
```

**What this means:**
- Each item document has a `store_id` field
- This field contains one store ID (scalar value)
- It references documents in the `store` namespace
- During search, you can join items with their stores

---

### Example 2: List Foreign Key (Multiple Values)

**Real Example:** An item can be available at multiple stores

```jsonnet
{
  schema: {
    primary_key: {
      name: "id",
      source: "id"
    },

    foreign_keys: [
      {
        name: "store_ids",                    // Note: plural
        source: {
          name: "available_at_stores",        // Array field in source data
          container_data_type: "list"         // Multiple values
        },
        children: ["store"]
      }
    ]
  }
}
```

**Sample Document:**

```json
{
  "id": "item_456",
  "name": "Coca Cola",
  "available_at_stores": ["store_42", "store_99", "store_117"]
}
```

**Indexed as:**
- `store_ids` field contains: `["store_42", "store_99", "store_117"]`
- Automatically deduplicated if source has duplicates
- Can match any of these three stores in joins

---

### Example 3: Multiple Foreign Keys

**Real Example from codebase:** `configurations/food-items/prod/item/schema.libsonnet`

```jsonnet
{
  schema: {
    primary_key: {
      name: "id",
      source: "id"
    },

    foreign_keys: [
      {
        name: "store_id",
        source: { name: "store_id", required: true },
        children: ["store"]                    // References store namespace
      },
      {
        name: "menu_id_store_id",
        source: { name: "menu_id_store_id" },
        children: ["menu"]                     // References menu namespace
      },
      {
        name: "category_id",
        source: { name: "category_id" },
        children: ["category"]                 // References category namespace
      }
    ]
  }
}
```

**Document Example:**

```json
{
  "id": "item_789",
  "name": "Caesar Salad",
  "store_id": "store_42",
  "menu_id_store_id": "menu_7_store_42",
  "category_id": "cat_salads"
}
```

**Relationships:**

```
item_789
├─ store_id → store_42 (can join with store namespace)
├─ menu_id_store_id → menu_7_store_42 (can join with menu namespace)
└─ category_id → cat_salads (can join with category namespace)
```

---

## 3. Join Queries: Using Foreign Keys

### Example 1: Basic Join Query

**Use Case:** "Find all pizza items from stores with rating > 4.0"

```json
{
  "namespace": "item",
  "keywords": {
    "query": "pizza"
  },
  "join": {
    "search_query": [
      {
        "namespace": "store",
        "filter": {
          "range_query": {
            "field": "rating",
            "lower_bound": 4.0
          }
        },
        "return_fields": ["id", "name", "rating"],
        "size": 1000
      }
    ]
  },
  "return_fields": ["id", "name", "price", "store_id"],
  "limit": 20
}
```

#### Execution Flow

**Step 1: Execute inner query on store namespace**

```
Search stores WHERE rating > 4.0
Results:
  - store_42 (rating: 4.5)
  - store_99 (rating: 4.8)
  - store_117 (rating: 4.2)

Extracted Primary Keys: ["store_42", "store_99", "store_117"]
```

**Step 2: Create join clause for item namespace**

```kotlin
// Internally, this becomes:
TermInSetQuery(
  field = "store_id",
  values = ["store_42", "store_99", "store_117"]
)
```

**Step 3: Execute outer query with join clause**

```
Search items WHERE
  keywords MATCH "pizza"
  AND store_id IN ["store_42", "store_99", "store_117"]

Results:
  - item_123 (name: "Margherita Pizza", store_id: "store_42")
  - item_456 (name: "Pepperoni Pizza", store_id: "store_99")
  - item_789 (name: "Hawaiian Pizza", store_id: "store_42")
```

**Step 4: Attach child documents to results**

```json
{
  "documents": [
    {
      "id": "item_123",
      "name": "Margherita Pizza",
      "price": 12.99,
      "store_id": "store_42",
      "store": {                          // ← Child document attached
        "id": "store_42",
        "name": "Joe's Italian Restaurant",
        "rating": 4.5
      }
    },
    {
      "id": "item_456",
      "name": "Pepperoni Pizza",
      "price": 14.99,
      "store_id": "store_99",
      "store": {
        "id": "store_99",
        "name": "Best Pizza Place",
        "rating": 4.8
      }
    }
  ]
}
```

---

### Example 2: Join with Reference Field

**Use Case:** "Find items from the same menu as high-rated stores"

```json
{
  "namespace": "item",
  "keywords": {
    "query": "pasta"
  },
  "filter": {
    "point_in_set_query": {
      "field": "menu_id",
      "query_value": {
        "reference_field": {           // ← Reference to join results
          "namespace": "store",
          "field": "menu_id"
        }
      }
    }
  },
  "join": {
    "search_query": [
      {
        "namespace": "store",
        "filter": {
          "range_query": {
            "field": "rating",
            "lower_bound": 4.5
          }
        },
        "return_fields": ["id", "menu_id"],
        "size": 1000
      }
    ]
  }
}
```

#### Execution Flow

**Step 1: Execute store query**

```
Search stores WHERE rating >= 4.5
Results:
  - store_42 (menu_id: 7)
  - store_99 (menu_id: 12)
  - store_150 (menu_id: 7)
```

**Step 2: Resolve reference field**

```kotlin
// Extract menu_id values from store results
reference_field "store.menu_id" → [7, 12, 7]

// Convert to concrete query (with dedup)
PointInSetQuery(
  field = "menu_id",
  values = [7, 12]
)
```

**Step 3: Execute item query**

```
Search items WHERE
  keywords MATCH "pasta"
  AND menu_id IN [7, 12]
  AND store_id IN ["store_42", "store_99", "store_150"]  // From join

Results: Items matching pasta from menus 7 or 12,
         AND belonging to high-rated stores
```

---

## 4. How Foreign Keys are Indexed

### Lucene Index Structure

When you index a document with foreign keys:

**Document:**

```json
{
  "id": "item_123",
  "name": "Margherita Pizza",
  "store_id": "store_42"
}
```

**Lucene Fields Created:**

```java
// From ForeignKeyFieldModel.parse():

1. StringField("store_id", "store_42", Field.Store.YES)
   // Allows exact retrieval of the foreign key value

2. SortedSetDocValuesField("store_id", BytesRef("store_42"))
   // Allows fast iteration over foreign key values without doc loading

3. StringField("store_id.store:term", "store_42", Field.Store.NO)
   // Creates reference term for reverse lookups
   // Format: {foreign_key_field}.{child_namespace}:term
```

**Why three fields?**

1. **StringField**: Fast exact matching for joins
2. **DocValues**: Memory-efficient access during scoring/sorting
3. **Reference term**: Enables reverse lookups (finding parents by child)

---

## 5. Runtime Example: Join Execution

Let's trace a real query execution:

**Query:** "Find pizza items from stores near Times Square"

```json
{
  "namespace": "item",
  "keywords": { "query": "pizza" },
  "join": {
    "search_query": [{
      "namespace": "store",
      "filter": {
        "geo_distance": {
          "field": "location",
          "lat": 40.758,
          "lon": -73.985,
          "distance_km": 1.0
        }
      },
      "return_fields": ["id", "name", "address"],
      "size": 100
    }]
  }
}
```

### Runtime Trace

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt:136-165](searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt#L136-L165)

#### Step 1: Execute inner join queries recursively

```kotlin
val innerQueryResults: Map<String, List<ArgoResults>> =
    outerSearchQuery.join.searchQuery.associate { innerSearchQuery ->
        val results = matchAndRank(innerSearchQuery)
        innerSearchQuery.namespace to listOf(results)
    }

// Results for "store" namespace:
// innerQueryResults["store"] = [
//   ArgoResults(
//     documents = [
//       ArgoDocument(primaryKey = "store_42", ...),
//       ArgoDocument(primaryKey = "store_99", ...),
//       ArgoDocument(primaryKey = "store_117", ...)
//     ]
//   )
// ]
```

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt:218-231](searcher/src/main/kotlin/com/doordash/argo/searcher/search/ArgoSearcher.kt#L218-L231)

#### Step 2: Create join clause

```kotlin
private fun makeJoinClause(
    indexModel: SearcherIndexModel,
    outerSearchQuery: SearchQuery,
    innerPrimaryKeys: List<StringFieldValue>,  // ["store_42", "store_99", "store_117"]
    innerSearchQuery: SearchQuery               // namespace: "store"
): BooleanClause {

    // Find the foreign key field in item namespace that references store
    val foreignKeyField = indexModel.getForeignKeyField(
        parentNamespace = "item",
        childNamespace = "store"
    )
    // foreignKeyField.name = "store_id"

    // Create IN query: store_id IN ["store_42", "store_99", "store_117"]
    val joinQuery = foreignKeyField.makeQuery(innerPrimaryKeys)

    return BooleanClause(joinQuery, BooleanClause.Occur.FILTER)
}
```

**File:** [core/src/main/kotlin/com/doordash/argo/core/index/ForeignKeyFieldModel.kt:71-81](core/src/main/kotlin/com/doordash/argo/core/index/ForeignKeyFieldModel.kt#L71-L81)

```kotlin
override fun makeQuery(values: List<FieldValue>): Query {
    val terms = values.map { value ->
        BytesRef((value as StringFieldValue).value)
    }.toSet()

    return TermInSetQuery(
        fieldName,  // "store_id"
        terms       // [BytesRef("store_42"), BytesRef("store_99"), BytesRef("store_117")]
    )
}
```

#### Step 3: Execute main query

```kotlin
// Final Lucene query for items:
BooleanQuery(
  SHOULD: KeywordQuery("pizza"),           // Score by relevance
  FILTER: TermInSetQuery(                   // Only items from these stores
    field = "store_id",
    terms = ["store_42", "store_99", "store_117"]
  )
)
```

#### Step 4: Retrieve and attach child documents

**File:** [searcher/src/main/kotlin/com/doordash/argo/searcher/search/MatchingAndRankingCollector.kt:397-422](searcher/src/main/kotlin/com/doordash/argo/searcher/search/MatchingAndRankingCollector.kt#L397-L422)

```kotlin
// For each matched item document:
private fun getChildDocuments(docId: Int): List<ListOfDocumentsFieldValue> {

    // Read foreign key value from DocValues
    val foreignKeyDocValues = foreignKeysBound[0].docValues
    foreignKeyDocValues.advanceExact(docId)

    // Example: For item_123, reads "store_42"
    val foreignKeyValue = foreignKeyDocValues.nextOrd()

    // Lookup in innerDocsByPk map
    val childDoc = innerDocsByPk["store_42"]
    // Returns: ArgoDocument(
    //   primaryKey = "store_42",
    //   fields = {
    //     "name": "Joe's Italian Restaurant",
    //     "address": "123 W 42nd St"
    //   }
    // )

    return listOf(
        ListOfDocumentsFieldValue(
            namespace = "store",
            documents = listOf(childDoc)
        )
    )
}
```

### Final Result

```json
{
  "documents": [
    {
      "id": "item_123",
      "name": "Margherita Pizza",
      "price": 12.99,
      "store_id": "store_42",
      "store": [                              // ← Child documents
        {
          "id": "store_42",
          "name": "Joe's Italian Restaurant",
          "address": "123 W 42nd St"
        }
      ]
    }
  ]
}
```

---

## 6. Advanced Example: Multi-Level Joins

You can have multiple foreign keys creating complex relationships:

```
Category (e.g., "Italian Food")
   ↑
   | (category_id)
   |
Menu (e.g., "Dinner Menu")
   ↑
   | (menu_id)
   |
Item (e.g., "Margherita Pizza")
   ↑
   | (store_id)
   |
Store (e.g., "Joe's Restaurant")
```

### Configuration

```jsonnet
// item/schema.libsonnet
foreign_keys: [
  { name: "store_id", children: ["store"] },
  { name: "menu_id", children: ["menu"] },
  { name: "category_id", children: ["category"] }
]

// menu/schema.libsonnet
foreign_keys: [
  { name: "category_id", children: ["category"] }
]
```

### Query Example

```json
{
  "namespace": "item",
  "keywords": { "query": "pizza" },
  "join": {
    "search_query": [
      {
        "namespace": "store",
        "filter": { "range_query": { "field": "rating", "lower_bound": 4.0 } }
      },
      {
        "namespace": "menu",
        "filter": { "term_query": { "field": "meal_type", "value": "dinner" } }
      },
      {
        "namespace": "category",
        "filter": { "term_query": { "field": "cuisine", "value": "italian" } }
      }
    ]
  }
}
```

This finds pizza items that:
- Belong to stores with rating ≥ 4.0
- Are on dinner menus
- Are in the Italian cuisine category

---

## Summary: Foreign Key Workflow

| Step | Description |
|------|-------------|
| **1. Configuration** | Define foreign keys in schema with name, source, and children |
| **2. Indexing** | Foreign key values indexed as StringField + DocValues + reference terms |
| **3. Join Query** | Inner query executes first, returns primary keys |
| **4. Join Clause** | Creates TermInSetQuery on foreign key field with inner results |
| **5. Outer Query** | Executes with join clause as FILTER |
| **6. Hydration** | Child documents attached using DocValues lookup |

---

## Key Insights

### Performance Benefits

- **DocValues**: Fast foreign key lookups without loading full documents
- **TermInSetQuery**: Efficient filtering on foreign key values
- **Reference Terms**: Enable bidirectional relationship queries

### Design Patterns

- **Scalar Foreign Keys**: One-to-one or many-to-one relationships
- **List Foreign Keys**: Many-to-many relationships
- **Multi-Level Joins**: Complex hierarchical data structures

### Best Practices

1. **Use `required: true`** for mandatory relationships to catch data issues early
2. **Keep join result sizes bounded** with the `size` parameter to avoid memory issues
3. **Index only necessary foreign keys** - each adds storage overhead
4. **Consider denormalization** for frequently accessed fields instead of always joining
