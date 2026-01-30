# How Indexer Indexes Data from Data Sources

## Architecture Overview

The Argo Search indexer operates in two phases:

1. **Full Build** - Complete table scan on initialization
2. **Incremental Updates** - Continuous updates from high-priority queues

---

## Complete Indexing Flow with Example

This example demonstrates how a **Store** index with child **Products** is built.

---

## Phase 1: Initialization

### Environment Variables

```bash
ARGO_INDEX_METADATA → /config/store_index.yaml
ARGO_STACK_CONFIG → /config/stack.yaml
ARGO_INDEXER_DIR → /data/indexer
IS_FIRST_STAGE → true
```

### Step 1: Parse Index Metadata

The system reads the index configuration:

```yaml
namespaces:
  store:
    data_source:
      type: crdb
      table: stores
    fields:
      - store_id (primary)
      - name
      - location
    foreign_keys:
      - products (→ product)

  product:
    data_source:
      type: crdb
      table: products
    fields:
      - product_id (primary)
      - store_id
      - name
      - price
```

### Step 2: Create IndexerIndexModel

- **Root**: "store" namespace
- **Child**: "product" namespace

For each namespace:
- **InputDocumentReader** (CRDBInputDocumentReader)
- **UpdateReader** (Kafka consumer)

### Step 3: Initialize IndexerRepository

Creates the repository structure:

```
/data/indexer/repository/
├── index/           (Lucene directory)
├── commits/
└── repository_manifest.yaml
```

---

## Phase 2: Full Build (indexAll)

### Step 1: Save Update Queue Position

The indexer saves the current Kafka positions to catch up later:

```
Kafka Topic: "store_updates"

Partition 0: offset 1000  ← Save these positions
Partition 1: offset 1050     (for catch-up later)
Partition 2: offset 980
```

### Step 2: Index Root Namespace (Store)

**SQL Query:**
```sql
SELECT store_id, name, location, products
FROM stores
WHERE micro_shard_id IN (0,1,2,3,4)  -- For this shard
```

**Documents Retrieved:**

```json
{
  "store_id": "S001",
  "name": "Downtown Store",
  "location": "123 Main St",
  "products": ["P001", "P002", "P003"]
}

{
  "store_id": "S002",
  "name": "Uptown Store",
  "location": "456 Oak Ave",
  "products": ["P004", "P005"]
}
```

**For Each Document:**

1. **Parse with schema** - Create Lucene Document:
   ```
   Field: store_id = "S001" (stored, indexed)
   Field: name = "Downtown Store" (stored, indexed)
   Field: location = "123 Main St" (stored)
   Field: _namespace = "store" (metadata)
   Field: _update_type = "0" (full build)
   ```

2. **Add to index:**
   ```
   repository.addDocument(luceneDoc)
   └─> IndexWriter.addDocument()
   ```

3. **Collect child IDs:**
   ```
   NamespaceIdCollector.collect()
   └─> foreign_key "products" → ["P001","P002","P003"]
   ```

### Step 3: Index Child Namespace (Product)

**Collect Foreign Keys:**

```
NamespaceIdCollector.apply()
For foreign key "products":
  Collected IDs: ["P001","P002","P003","P004","P005"]
```

**SQL Query:**
```sql
SELECT product_id, store_id, name, price
FROM products
WHERE product_id IN ('P001','P002','P003','P004','P005')
```

**Product Documents Retrieved:**

```json
{
  "product_id": "P001",
  "store_id": "S001",
  "name": "Pizza",
  "price": 12.99
}

{
  "product_id": "P002",
  "store_id": "S001",
  "name": "Burger",
  "price": 8.99
}
```

**For Each Product:**

Create Lucene Document and add to index:

```
Field: product_id = "P001"
Field: store_id = "S001" (join field)
Field: name = "Pizza"
Field: price = 12.99 (NumericDocValuesField)
Field: _namespace = "product"
```

### Step 4: Commit and Merge

```
repository.commit()
└─> IndexWriter.commit()
    └─> Flush all buffered docs to disk

repository.waitAndPersistMerge()
└─> Trigger Lucene merge policy
└─> Wait for background segment merges

Result: Optimized index segments on disk
```

### Step 5: Push to S3

**Upload Structure:**

```
s3://bucket/store-index/generation-5/
├── index/
│   ├── segments_1
│   ├── _0.cfs (segment 0)
│   └── _1.cfs (segment 1)
├── commits/
│   └── 000000000001.commit_manifest.yaml
└── repository_manifest.yaml
```

**Commit Manifest:**

```yaml
commitNumber: 1
fullBuildTime: 2026-01-29T10:00:00Z
namespaces: [store, product]
documentsIndexed:
  store: 1500
  product: 5200
```

---

## Phase 3: Incremental Updates (Update Loop)

**Frequency:** Every 60 seconds (commitPeriod)

### Step 1: Read from Update Queue

```
UpdateReader.read()
Kafka Topic: "store_updates"
```

**Messages Read (since position saved at full build):**

```
Partition 0, Offset 1001:
  {type: UPDATE, key: "S001"}

Partition 0, Offset 1002:
  {type: UPDATE, key: "S003"}

Partition 1, Offset 1051:
  {type: DELETE, key: "S010"}

Partition 0, Offset 1003:
  {type: UPDATE, key: "S001"}  ← Duplicate!
```

### Step 2: Dedupe Updates

Keep only the latest update per document:

```
After deduplication:
- S001: UPDATE (from offset 1003, latest)
- S003: UPDATE
- S010: DELETE
```

### Step 3: Fetch Updated Documents

**SQL Query:**
```sql
SELECT store_id, name, location, products
FROM stores
WHERE store_id IN ('S001', 'S003')
```

**Result:**

```json
{
  "store_id": "S001",
  "name": "Downtown Store UPDATED",
  "location": "123 Main St",
  "products": ["P001", "P002", "P003", "P006"]  // New product P006 added
}

{
  "store_id": "S003",
  "name": "New Store",
  ...
}
```

### Step 4: Execute Update Commands

**For S001 UPDATE:**

```
ReIndexUpdateCommand.execute()
├─ Create Lucene Document with updated fields
│  └─ Field: _update_type = "1" (incremental)
│  └─ Field: update_queue:partition = 0
│  └─ Field: update_queue:offset = 1003
│
├─ repository.updateDocument(
│    key = Term("store_id", "S001"),
│    doc = updatedLuceneDoc
│  )
│  └─> IndexWriter deletes old doc, adds new
│
└─ NamespaceIdCollector.collect()
   └─> products: ["P001","P002","P003","P006"]
       └─> New child: P006 (needs indexing)
```

**For S010 DELETE:**

```
repository.deleteDocuments(
  Term("store_id", "S010")
)
```

### Step 5: Update Affected Children

```
New/changed child IDs detected: ["P006"]

InputDocumentReader.readByKeys(["P006"])
└─> Fetch new product from database

Add P006 to index:
└─> repository.addDocument(productP006)
```

### Step 6: Commit and Push

```
repository.commit()
└─> Persist changes to disk

repository.push(
  nextIndexState,
  nextIncrementalUpdateTime = 2026-01-29T10:01:00Z
)
└─> Upload to S3
    └─> New commit manifest:
        commitNumber: 2
        updateQueueTimeOffset: 2026-01-29T10:01:00Z
```

### Step 7: Check Readiness

```
Calculate lag:
  updateLag = 50 messages behind
  indexSize = 6700 documents
  lagRatio = 50 / 6700 = 0.007 = 0.7%

if lagRatio < 0.5%:  ← Threshold
  markIndexReady()
  └─> Searchers can now query this index
```

**This process repeats every 60 seconds indefinitely.**

---

## Index Storage Structure

After indexing completes, the repository structure looks like:

```
/data/indexer/repository/
│
├─ index/                          # Lucene index directory
│  ├─ segments_2                   # Current segment info file
│  ├─ _0.cfs                       # Compound file segment 0
│  ├─ _1.cfs                       # Compound file segment 1
│  ├─ _2.cfe, _2.cfs              # Newer segments from updates
│  └─ write.lock                   # Write lock
│
├─ commits/                        # Commit history
│  ├─ 000000000000.commit_manifest.yaml  # Initial empty
│  ├─ 000000000001.commit_manifest.yaml  # After full build
│  └─ 000000000002.commit_manifest.yaml  # After incremental
│
└─ repository_manifest.yaml        # Overall repo metadata
```

### Lucene Document Fields

Each Lucene document contains:

- **Indexed fields** (for search): name, location
- **Stored fields** (for retrieval): all fields
- **DocValues** (for sorting/faceting): price
- **Metadata**: `_namespace`, `_update_type`, update_queue offsets
