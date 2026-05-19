# MySQL Indexes: How They Work

Date: 2026-05-19

## Question

What is an index in MySQL and how does it work? (Both conceptual understanding and implementation details)

## Answer Summary

### What Problem Indexes Solve

Indexes transform query performance from O(n) full table scans to O(log n) lookups. Like a book's index that points to specific pages rather than reading every page, database indexes provide fast lookup structures.

### B+ Tree Implementation

MySQL InnoDB uses **B+ Trees** - balanced tree structures optimized for disk storage:

**Structure:**
- Internal nodes: store keys and pointers (directory structure)
- Leaf nodes: store actual data or pointers to data
- Leaf nodes are linked in a chain for range queries
- Tree stays balanced: all root-to-leaf paths equal length

**Lookup Process:**
For 1 million rows, typically only 3-4 comparisons needed:
1. Start at root node
2. Navigate through internal nodes
3. Reach leaf node with data/pointer

**Why B+ Trees (not binary trees):**
- Wide nodes (hundreds of keys) = fewer tree levels
- Fewer disk I/O operations (each level = one disk read)
- Cache-friendly: nodes fit in disk pages (16KB in InnoDB)

### Index Types in InnoDB

**1. Primary Key (Clustered Index)**
- Table data IS stored in B+ tree leaf nodes
- Table physically ordered by primary key
- Fastest lookup (no secondary lookup needed)

**2. Secondary Indexes**
- Leaf nodes contain: `[indexed_value | primary_key]`
- Requires two lookups: index → primary key → actual row
- Called "bookmark lookup"

### Range Queries

Linked leaf nodes enable efficient range scans:
```sql
SELECT * FROM users WHERE id BETWEEN 100 AND 200;
```
Once MySQL finds id=100, it walks the linked list of leaf nodes to 200 without re-traversing the tree.

### Tradeoffs

**Benefits:**
- Fast lookups: O(log n) vs O(n)
- Efficient range queries
- Supports ORDER BY, GROUP BY optimizations

**Costs:**
- Storage: each index is a separate B+ tree structure
- Write overhead: INSERT/UPDATE/DELETE must update all relevant index trees
- Tree rebalancing operations when nodes are full

### When to Use Indexes

**Index these columns:**
- WHERE clause filters
- JOIN conditions
- ORDER BY columns
- GROUP BY columns

**Avoid over-indexing:**
- Small tables (< 1,000 rows)
- Frequently updated columns
- Low cardinality columns (e.g., boolean fields)
- Tables with heavy write workload

---

## Creating Indexes: Different Methods

MySQL provides multiple ways to create indexes:

### 1. Standalone CREATE INDEX
```sql
CREATE INDEX idx_email ON users(email);
```

### 2. Inside CREATE TABLE
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    email VARCHAR(255),
    INDEX idx_email (email)
);
```

### 3. ALTER TABLE (for existing tables)
```sql
ALTER TABLE users ADD INDEX idx_email (email);
```

### 4. UNIQUE INDEX (enforces uniqueness)
```sql
CREATE UNIQUE INDEX idx_email ON users(email);
```

### 5. Specialized Indexes
- **PRIMARY KEY**: Automatic clustered index
- **FULLTEXT**: For text search
- **SPATIAL**: For geographic data

All methods create the same B+ tree structure - different syntax for different scenarios.

---

## Multi-Column (Composite) Indexes

Indexes can span multiple columns, enabling efficient queries on multiple conditions.

### Syntax
```sql
CREATE INDEX idx_country_city_age ON users(country, city, age);
```

### The "Phonebook Principle"

Like a phonebook sorted by (last_name, first_name):
- Can look up by last name ✓
- Can look up by last name + first name ✓
- Cannot efficiently look up by first name only ✗

### Leftmost Prefix Rule

**Index on (A, B, C) works for queries filtering on:**
- A only ✓
- A, B ✓
- A, B, C ✓

**Does NOT work for:**
- B only ✗
- C only ✗
- B, C ✗

### Storage in B+ Tree

For index `(country, city, age)`, leaf nodes sorted:
```
['Canada', 'Montreal', 25] → pointer
['Canada', 'Montreal', 30] → pointer
['Canada', 'Toronto', 28] → pointer
['US', 'LA', 35] → pointer
['US', 'NYC', 22] → pointer
```

Can efficiently find:
- All 'US' entries (first column)
- Within 'US', all 'NYC' entries (second column)
- Within 'US'+'NYC', specific age (third column)

Cannot efficiently find all 'NYC' entries across countries (not grouped together).

### Column Order Matters

```sql
-- Index A: good for queries starting with country
CREATE INDEX idx_country_age ON users(country, age);

-- Index B: good for queries starting with age
CREATE INDEX idx_age_country ON users(age, country);
```

These are completely different indexes!

**Choose column order based on:**
1. **Selectivity**: Most selective (fewest matches) column first
2. **Query patterns**: Which columns filtered most often
3. **Equality before range**: Equality conditions before range conditions

### Example: Equality Before Range
```sql
-- GOOD: equality first, range second
CREATE INDEX idx_country_age ON users(country, age);
SELECT * FROM users WHERE country = 'US' AND age > 25;
```

Once you do a range scan, subsequent columns in the index are no longer sorted for filtering.

### Covering Indexes

When a query only needs columns IN the index, MySQL never looks up the actual row:

```sql
CREATE INDEX idx_country_city_age ON users(country, city, age);

-- "Covered" by index - no table lookup needed
SELECT country, city, age FROM users WHERE country = 'US';
```

This is called an **index-only scan** - extremely fast!

### Composite vs Multiple Single-Column Indexes

**Multiple single indexes:**
```sql
CREATE INDEX idx_country ON users(country);
CREATE INDEX idx_age ON users(age);
```

**Query: `WHERE country = 'US' AND age = 30`**
- MySQL picks ONE index, then filters remaining condition in memory

**One composite index:**
```sql
CREATE INDEX idx_country_age ON users(country, age);
```

**Query: `WHERE country = 'US' AND age = 30`**
- MySQL uses BOTH conditions efficiently in tree traversal

**Composite is usually better for multi-condition queries.**

---

## Key Insights

1. **Index = sorted lookup structure** optimized for O(log n) search
2. **B+ Tree = disk-optimized balanced tree** with wide nodes and linked leaves
3. **Primary key = clustered index** where data IS the tree
4. **Secondary index = points to primary key** requiring bookmark lookup
5. **Composite indexes follow leftmost prefix rule** - column order critical
6. **Covering indexes eliminate table lookups** when all needed columns are indexed
7. **Tradeoff: read speed vs write cost** - every index adds write overhead

---

## Full Conversation

# Question Log
Started: 2026-05-19 12:05

## Phase 1: Log Start
✓ Logging initiated

## Phase 2: Question
User: "i would like to understand what an index is and how it works in a mysql database"

## Phase 3: Clarification

Claude: "Are you interested in understanding:
1. The general concept and purpose of indexes (what problem they solve)?
2. The specific data structure MySQL uses internally (B-trees)?
3. Both - how they work conceptually AND the implementation details?"

User: "3"

## Phase 4: Answer

Claude provided comprehensive explanation covering:

**Conceptual Understanding:**
- Problem indexes solve: O(n) full table scan → O(log n) lookup
- Book analogy: index at back of book vs reading every page

**Implementation Details:**
- B+ Tree data structure used by MySQL InnoDB
- Structure: internal nodes (keys/pointers) + leaf nodes (data) + linked leaves
- Lookup process: root → internal nodes → leaf (typically 3-4 steps for millions of rows)
- Why B+ Trees not binary trees: wider nodes = fewer levels = fewer disk I/O operations

**Storage Mechanics:**
- Primary Key (Clustered Index): table data IS stored in B+ tree leaves
- Secondary Indexes: leaves contain [indexed_value | primary_key], requires bookmark lookup
- Leaf node linking enables efficient range queries

**Tradeoffs:**
- Storage cost: each index is separate B+ tree structure
- Write cost: every INSERT/UPDATE/DELETE must update all index trees
- May require tree rebalancing

**When to use indexes:**
- WHERE clauses, JOINs, ORDER BY, GROUP BY columns
- Avoid over-indexing: small tables, frequently updated columns, low cardinality, heavy writes

Full technical explanation with visual examples provided.

## Phase 5: Confirmation

Claude: "Does this explanation make sense? Do you understand both how indexes solve the performance problem conceptually and how B+ trees work internally?"

User: "yes, i also wonder if there is another way to create particular indexes on a table and if you can create index on multiple fields."

Claude: Explained additional concepts:

**Different ways to create indexes:**
- CREATE INDEX (standalone)
- Inside CREATE TABLE
- ALTER TABLE ADD INDEX
- UNIQUE INDEX (enforces uniqueness)
- PRIMARY KEY (automatic clustered index)
- FULLTEXT INDEX (text search)
- SPATIAL INDEX (geographic data)

**Multi-column (Composite) Indexes:**
- Syntax: CREATE INDEX idx_name ON table(col1, col2, col3)
- Phonebook principle: sorted by first column, then second, then third
- Leftmost prefix rule: index on (A,B,C) works for queries on A, A+B, A+B+C, but NOT B, C, or B+C
- Storage: B+ tree leaves sorted by all columns in order
- Column order matters: selectivity, query patterns, equality before range
- Covering indexes: when query only needs indexed columns, no table lookup needed
- Composite index more efficient than multiple single-column indexes for multi-condition queries

Full examples and use cases provided.

User: "yes, everything is clear now"

---

Completed: 2026-05-19 12:15
