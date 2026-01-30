# Search Query Structure in Argo Search

---

## 1. Top-Level Structure

SearchQuery is the core model defined in [core/src/main/kotlin/com/doordash/argo/core/model/SearchQuery.kt:12](core/src/main/kotlin/com/doordash/argo/core/model/SearchQuery.kt#L12):

```kotlin
data class SearchQuery(
    val namespace: String,                    // Target index (e.g., "store", "item")
    val keywordsQuery: KeywordsQuery,         // Text/vector search
    val filter: Filter,                       // Boolean filters
    val join: Join,                           // Joins with other namespaces
    val groupBy: GroupBy,                     // Grouping/deduplication
    val facetCount: FacetCount,              // Aggregations
    val returnFields: List<String>,           // Fields to return
    val contextFeatures: Map<String, Value>,  // Context for ranking
    val phasedSortBy: PhasedSortBy,          // Sorting strategy
    val dedup: Dedup,                        // Deduplication
    val queryPlanner: QueryPlanner?,          // Query pipeline
    val reorderings: List<Reordering>        // Post-processing
)
```

---

## 2. Filter Structure

### FilterQuery Types

From [core/src/main/kotlin/com/doordash/argo/core/model/Filter.kt](core/src/main/kotlin/com/doordash/argo/core/model/Filter.kt)

```kotlin
sealed interface FilterQuery

// Composition
data class BooleanQuery(
    val clauses: List<BooleanClause>,
    val minimumShouldMatch: Int = 0
) : FilterQuery

data class BooleanClause(
    val occur: Occur,        // MUST, MUST_NOT, SHOULD, FILTER
    val query: FilterQuery
)
```

### Occurrence Types

| Occur | Description | Logic |
|-------|-------------|-------|
| **MUST** | Required (affects scoring) | AND logic |
| **MUST_NOT** | Exclusion | NOT logic |
| **SHOULD** | Optional (needs minimumShouldMatch) | OR logic |
| **FILTER** | Required but doesn't affect scoring | Post-filtering |

### Typical Filter Example

```json
{
  "filter": {
    "filter_query": {
      "boolean_query": {
        "clauses": [
          {
            "occur": "OCCUR_MUST",
            "query": {
              "term_query": {
                "field": "is_active",
                "value": "true"
              }
            }
          },
          {
            "occur": "OCCUR_MUST",
            "query": {
              "point_in_set_query": {
                "field": "price_range",
                "query_value": {
                  "int_values": {"elements": [1, 2]}
                }
              }
            }
          },
          {
            "occur": "OCCUR_MUST_NOT",
            "query": {
              "term_query": {
                "field": "is_closed",
                "value": "true"
              }
            }
          }
        ],
        "minimum_should_match": 0
      }
    }
  }
}
```

**Translation:** `is_active = true AND price_range IN (1,2) AND is_closed != true`

---

## 3. Join Structure

### Join Model

From [core/src/main/kotlin/com/doordash/argo/core/model/Join.kt:10](core/src/main/kotlin/com/doordash/argo/core/model/Join.kt#L10)

```kotlin
data class Join(val innerSearchQueries: List<SearchQuery>)
```

### How Joins Work

1. **Inner query executes first** (the "follower" namespace)
2. Returns a set of matching documents
3. Outer query filters based on inner query results
4. Use `ReferenceField` to reference fields from inner results

---

### Example 1: Simple Join - "Items from nearby stores"

```json
{
  "search_query": {
    "namespace": "item",
    "keywords_query": {
      "groups": [{
        "keywords": [{
          "keyword": "chicken",
          "field": "name"
        }]
      }]
    },
    "join": {
      "search_query": [{
        "namespace": "store",
        "filter": {
          "filter_query": {
            "geo_distance_query": {
              "field": "location",
              "distance_meters": 5000.0,
              "point": {"lat": 37.4455, "lon": -122.1602}
            }
          }
        },
        "return_fields": ["store_id", "name"],
        "size": 1000
      }]
    },
    "return_fields": ["item_id", "name", "store_id"],
    "size": 100
  }
}
```

**Translation:** "Find chicken items, but only from stores within 5km of the location"

---

### Example 2: Join with ReferenceField - "Items with matching menu_id"

```json
{
  "search_query": {
    "namespace": "item",
    "keywords_query": {
      "groups": [{
        "keywords": [{"keyword": "pizza", "field": "name"}]
      }]
    },
    "filter": {
      "filter_query": {
        "boolean_query": {
          "clauses": [{
            "occur": "OCCUR_MUST",
            "query": {
              "point_in_set_query": {
                "field": "menu_id",
                "query_value": {
                  "reference_field": {
                    "namespace": "store",
                    "field": "menu_id"
                  }
                }
              }
            }
          }]
        }
      }
    },
    "join": {
      "search_query": [{
        "namespace": "store",
        "filter": {
          "filter_query": {
            "term_query": {
              "field": "is_open",
              "value": "true"
            }
          }
        },
        "return_fields": ["store_id", "menu_id"],
        "size": 1000
      }]
    }
  }
}
```

**Translation:** "Find pizza items where menu_id matches the menu_id values from open stores"

---

### Example 3: Three-Level Nested Join

```json
{
  "search_query": {
    "namespace": "item",
    "keywords_query": {
      "groups": [{
        "keywords": [{"keyword": "burger", "field": "name"}]
      }]
    },
    "join": {
      "search_query": [{
        "namespace": "menu",
        "filter": {
          "filter_query": {
            "term_query": {"field": "category", "value": "dinner"}
          }
        },
        "join": {
          "search_query": [{
            "namespace": "store",
            "filter": {
              "filter_query": {
                "term_query": {"field": "rating_gt_4", "value": "true"}
              }
            },
            "return_fields": ["store_id"],
            "size": 100
          }]
        },
        "return_fields": ["menu_id", "store_id"],
        "size": 500
      }]
    },
    "size": 50
  }
}
```

**Translation:** "Find burgers from dinner menus from stores with rating > 4"

**Execution order:** `store → menu → item` (innermost to outermost)

---

## 4. Keywords Query Structure

### KeywordsQuery Model

From [core/src/main/kotlin/com/doordash/argo/core/model/Keywords.kt:59](core/src/main/kotlin/com/doordash/argo/core/model/Keywords.kt#L59)

```kotlin
data class KeywordsQuery(
    val groups: List<KeywordGroup>,      // Text keyword groups
    val vectorQueries: List<VectorQuery>, // Vector similarity queries
    val clientKeywords: String?,          // Raw user input
    val fuzzyQueries: List<FuzzyQuery>   // Fuzzy matching
)

data class KeywordGroup(
    val keywords: List<Keyword>,
    val occur: KeywordGroupOccur,     // SHOULD, MUST, COMPUTED_FIELDS_ONLY
    val minShouldMatch: Int?          // Minimum keywords to match
)

data class Keyword(
    val keyword: String,
    val field: String,
    val ordinal: Int,                 // Position in query
    val occur: KeywordOccur          // SHOULD, MUST
)
```

---

### Example 1: Keywords with minShouldMatch

```json
{
  "keywords_query": {
    "groups": [{
      "keywords": [
        {"keyword": "chicken", "field": "name", "ordinal": 0},
        {"keyword": "spicy", "field": "name", "ordinal": 1},
        {"keyword": "curry", "field": "name", "ordinal": 2}
      ],
      "min_should_match": 2,
      "occur": "SHOULD"
    }]
  }
}
```

**Translation:** "Match documents where at least 2 out of 3 keywords appear"

---

### Example 2: Multiple Keyword Groups

```json
{
  "keywords_query": {
    "groups": [
      {
        "keywords": [
          {"keyword": "pizza", "field": "name", "occur": "MUST"}
        ],
        "occur": "MUST"
      },
      {
        "keywords": [
          {"keyword": "vegetarian", "field": "tags", "occur": "SHOULD"},
          {"keyword": "vegan", "field": "tags", "occur": "SHOULD"}
        ],
        "occur": "SHOULD",
        "min_should_match": 1
      }
    ]
  }
}
```

**Translation:** "Must contain 'pizza' in name AND at least one of 'vegetarian' or 'vegan' in tags"

---

### Example 3: Vector + Keywords Hybrid Search

```json
{
  "keywords_query": {
    "groups": [{
      "keywords": [
        {"keyword": "italian", "field": "cuisine", "occur": "SHOULD"}
      ],
      "occur": "SHOULD"
    }],
    "vector_queries": [{
      "field": "store_embedding",
      "target": {"elements": [0.1, 0.2, ..., 0.9]},
      "k": 10,
      "filter": {
        "term_query": {"field": "has_delivery", "value": "true"}
      },
      "occur": "SHOULD"
    }]
  }
}
```

**Translation:** "Find stores matching 'italian' keyword OR similar to the embedding vector (among stores with delivery)"

---

## 5. Complete Real-World Example

**Use case:** "Find affordable pizza items from highly-rated stores within 3km that are currently open"

```json
{
  "search_query": {
    "namespace": "item",

    "keywords_query": {
      "groups": [{
        "keywords": [
          {"keyword": "pizza", "field": "name", "ordinal": 0}
        ],
        "occur": "MUST"
      }],
      "client_keywords": "pizza"
    },

    "filter": {
      "filter_query": {
        "boolean_query": {
          "clauses": [
            {
              "occur": "OCCUR_MUST",
              "query": {
                "term_query": {
                  "field": "is_available",
                  "value": "true"
                }
              }
            },
            {
              "occur": "OCCUR_MUST",
              "query": {
                "point_range_query": {
                  "field": "price_cents",
                  "lower_value": {"int_values": {"elements": [0]}},
                  "upper_value": {"int_values": {"elements": [1500]}}
                }
              }
            },
            {
              "occur": "OCCUR_MUST",
              "query": {
                "point_in_set_query": {
                  "field": "store_id",
                  "query_value": {
                    "reference_field": {
                      "namespace": "store",
                      "field": "store_id"
                    }
                  }
                }
              }
            }
          ]
        }
      }
    },

    "join": {
      "search_query": [{
        "namespace": "store",
        "filter": {
          "filter_query": {
            "boolean_query": {
              "clauses": [
                {
                  "occur": "OCCUR_MUST",
                  "query": {
                    "geo_distance_query": {
                      "field": "location",
                      "distance_meters": 3000.0,
                      "point": {"lat": 37.7749, "lon": -122.4194}
                    }
                  }
                },
                {
                  "occur": "OCCUR_MUST",
                  "query": {
                    "term_query": {
                      "field": "is_open_now",
                      "value": "true"
                    }
                  }
                },
                {
                  "occur": "OCCUR_MUST",
                  "query": {
                    "point_range_query": {
                      "field": "rating",
                      "lower_value": {"float_values": {"elements": [4.0]}},
                      "upper_value": {"float_values": {"elements": [5.0]}}
                    }
                  }
                }
              ]
            }
          }
        },
        "return_fields": ["store_id", "name", "rating"],
        "size": 500
      }]
    },

    "return_fields": ["item_id", "name", "price_cents", "store_id"],
    "size": 50
  }
}
```

### Execution Flow

1. **Inner query (store):** Find stores within 3km, currently open, rating 4.0-5.0
2. **Outer query (item):** Find pizza items that are available, price ≤ $15, and belong to stores from step 1

---

## Query Architecture Summary

This architecture provides powerful composition of:

| Component | Purpose | Example Use Case |
|-----------|---------|------------------|
| **Filters** | Boolean conditions | Price ranges, availability, categories |
| **Joins** | Cross-namespace relationships | Items from nearby stores |
| **Keywords** | Text/vector search | Product name matching, semantic search |
| **Reference Fields** | Dynamic filter values from joins | Menu IDs from high-rated stores |
| **Nested Joins** | Multi-level relationships | Items → Menus → Stores → Categories |

---

## Key Design Patterns

### 1. Filter-First Pattern

Use filters to narrow the search space before applying expensive operations:

```json
{
  "filter": {
    "term_query": {"field": "is_active", "value": "true"}
  },
  "keywords_query": {
    "groups": [{"keywords": [...]}]
  }
}
```

### 2. Join with Reference Field

Extract values from join results to use in outer query filters:

```json
{
  "filter": {
    "point_in_set_query": {
      "field": "menu_id",
      "query_value": {
        "reference_field": {
          "namespace": "store",
          "field": "menu_id"
        }
      }
    }
  },
  "join": {
    "search_query": [{"namespace": "store", ...}]
  }
}
```

### 3. Hybrid Search Pattern

Combine keyword relevance with vector similarity:

```json
{
  "keywords_query": {
    "groups": [{"keywords": [...], "occur": "SHOULD"}],
    "vector_queries": [{"field": "embedding", "occur": "SHOULD"}]
  }
}
```

---

## Best Practices

1. **Limit Join Result Sizes**: Use the `size` parameter to prevent memory issues
2. **Use FILTER for Non-Scoring Conditions**: Saves computational overhead
3. **Apply Filters Before Joins**: Reduces the number of documents to process
4. **Set Reasonable `minShouldMatch`**: Balances precision and recall
5. **Use Reference Fields Judiciously**: They add complexity but enable powerful queries

---

## Performance Considerations

- **Join Size Impact**: Large join results can slow down queries
- **Filter Selectivity**: More selective filters early in the query reduce work
- **Vector Query Cost**: Vector searches are more expensive than keyword searches
- **Nested Join Depth**: Each level adds latency; keep depth to 2-3 levels max
- **Return Fields**: Only request fields you need to reduce payload size
