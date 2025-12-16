# Making the Fastest Database Even Faster: My ClickHouse 25.10 Contributions

Speed matters in data analytics. When you are processing billions of rows, even small performance improvements compound into massive time and cost savings. For ClickHouse 25.10, I contributed three code patches that push the boundaries of query performance—particularly for text search operations that are ubiquitous in modern data workloads.

## Merged Contributions

Three pull requests merged into the ClickHouse 25.10 changelog:

1. **New SIMD-optimized text search functions** ([PR#87374](https://github.com/ClickHouse/ClickHouse/pull/87374))
   - Added `startsWithCaseInsensitive` and `endsWithCaseInsensitive` functions with hand-tuned SIMD optimizations

2. **Query optimizer enhancement for pattern matching** ([Issue#71421](https://github.com/ClickHouse/ClickHouse/issues/71421), [PR#85920](https://github.com/ClickHouse/ClickHouse/pull/85920))
   - Automatically rewrites `LIKE` expressions with prefix/suffix patterns into faster SIMD-optimized function calls
   - **~5x performance improvement** on affix pattern queries

3. **Critical bug fix for parallel INSERT SELECT with CTEs** ([Issue#85368](https://github.com/ClickHouse/ClickHouse/issues/85368), [PR#87789](https://github.com/ClickHouse/ClickHouse/pull/87789))
   - Resolved metadata catalog resolution failure in parallel insertion to replicated tables

# Overview: Why These Contributions Matter

## Performance: Making Text Search Blazing Fast

ClickHouse has earned its reputation as the fastest analytics database through relentless optimization. My contributions in 25.10 focus on a critical operation that touches nearly every data workload: **text search**.

Text data is everywhere. Whether you are:
- Searching through application logs and distributed traces in observability platforms
- Retrieving information from document stores and knowledge bases
- Transforming textual data in data warehouses
- Building retrieval-augmented generation (RAG) pipelines for AI applications

...you need efficient text search operations. When these operations run on billions of rows, even small improvements multiply into dramatic performance gains.

### The Problem: Bridging Pattern Complexity and Performance

ClickHouse's query optimizer is decades of database research distilled into production code. And like research, it is constantly evolving for better. One area of improvement identified by the community is pattern matching with regular expressions.  Regular expression can be computationally expensive to evaluate. But many common patterns—like checking if a log message starts with "ERROR" or ends with ".json"—don't need the full power of regex engines.

**The idea**: For simple prefix and suffix patterns (what we call "affix" matching), SIMD-optimized substring comparison can be **5x faster** than regex evaluation.

**The solution**: Rather than requiring users to manually optimize their queries, we built logic into the query optimizer itself. A new optimization pass automatically detects `LIKE` expressions with affix patterns and rewrites them into SIMD-optimized function calls—transparent to the user, massive performance gain.

Now with PR#85920 (the query rewriting optimization) and PR#87374 (the additional SIMD-optimized functions), you can speed up your queries with:
- New database setting [optimize_rewrite_like_perfect_affix](https://clickhouse.com/docs/operations/settings/settings#optimize_rewrite_like_perfect_affix) to enable this optimization
- New SIMD-optimized functions [startsWithCaseInsensitive](https://clickhouse.com/docs/sql-reference/functions/string-functions#startsWithCaseInsensitive), [startsWithCaseInsensitiveUTF8](https://clickhouse.com/docs/sql-reference/functions/string-functions#startsWithCaseInsensitiveUTF8), [endsWithCaseInsensitive](https://clickhouse.com/docs/sql-reference/functions/string-functions#endsWithCaseInsensitive), [endsWithCaseInsensitiveUTF8](https://clickhouse.com/docs/sql-reference/functions/string-functions#endsWithCaseInsensitiveUTF8).

## Reliability: Fixing Data Ingestion at Scale

ClickHouse excels at ingesting massive data volumes from sources like Apache Kafka and Apache Iceberg. In replicated, shared-nothing deployments, ClickHouse parallelizes data insertion across replicas for maximum throughput.

However, a subtle bug (Issue#85368) caused failures when `INSERT SELECT` queries used common table expressions (CTEs). The query interpreter on each replica failed to resolve the CTE in its metadata catalog, breaking parallel ingestion workflows. 

PR#87789 fixes this catalog resolution issue, ensuring reliable data ingestion in production environments where replication is critical for both performance and fault tolerance.  

The community reported the bug back in August and since then there had been multiple follow-ups. Given its impact, the fix is synced to the ClickHouse Cloud, and backported to 25.8.  

Now with PR$87789, whether you are the user of ClickHouse Cloud or self-hosting cluster, you will be able to use the faster, parallel data ingestion reliably.

# Deep Dive: Technical Implementation Details

Let's explore how these optimizations work under the hood—from query tree transformations to SIMD intrinsics to distributed system debugging.

## PR#85920: Query Optimizer Rewrite for LIKE Expressions

### The Problem: Regex Overhead for Simple Patterns

Consider a common query pattern in data warehouse ([try it yourself](https://fiddle.clickhouse.com/4a0ba187-a260-49f9-afe5-af6c29f1831e)):

```sql
SELECT count(*) FROM products WHERE description LIKE 'ClickHouse%';
```

This matches "ClickHouse Server", "ClickHouse Local", "ClickHouse MCP", etc., but not "chDB".

In standard ClickHouse execution, `LIKE` patterns are compiled into regular expressions and evaluated using finite automata. This general-purpose approach handles complex patterns beautifully—but it's overkill for simple prefix and suffix matching. For these "affix" patterns, direct substring comparison is **~5x faster**.

### The Solution: Automatic Query Rewriting

Rather than asking users to manually rewrite their queries, we teach the optimizer to do it automatically. This PR adds a new optimization pass to ClickHouse's query analyzer.

#### Understanding Query Trees

Like most modern databases (and programming language compilers), ClickHouse parses SQL into abstract syntax trees (ASTs) and performs multiple optimization passes to transform these trees into more efficient execution plans. 

Here's what the query tree looks like for our example:

```
QUERY id: 0
  PROJECTION
    ...
  JOIN TREE
    ...
  WHERE
    FUNCTION id: 4, function_name: like, function_type: ordinary
      ARGUMENTS
        LIST id: 5, nodes: 2
          IDENTIFIER id: 6, identifier: name
          CONSTANT id: 7, constant_value: 'ClickHouse%', constant_value_type: String
```

The `WHERE` clause contains a `like` function with two arguments: the column `name` and the pattern `'ClickHouse%'`.

#### Choosing the Right Rewrite Strategy

When we detect a `LIKE` expression with an affix pattern (e.g., `description LIKE 'ClickHouse%'`), we have two main rewrite options:

**Option 1: Range Comparison**
```sql
'ClickHouse' <= description AND description < 'ClickHousf'
```

**Option 2: SIMD-Optimized Function**
```sql
startsWith(description, 'ClickHouse')
```

In a typical programming language, these might seem equivalent. But in an analytical database processing millions of rows, **Option 2 is ~5x faster**. Why?

### Why SIMD Functions Win: Less Computation, Simpler Pipeline

The performance difference comes down to how ClickHouse processes data in batches (called "granules") through execution pipelines.

**Option 1 challenges:**
- Requires **two** string comparisons per row (lower bound and upper bound)
- Creates intermediate results after the first comparison
- Must materialize and pass these intermediate results to the second comparison
- Doubles the number of filter transforms in the execution pipeline
- Increases memory pressure from storing intermediate data

**Option 2 advantages:**
- Requires only **one** comparison per row
- The `startsWith` function is hand-optimized with SIMD intrinsics to compare prefixes against multiple strings in parallel
- No intermediate results to store or pass
- Simpler, more efficient execution pipeline  

#### Visualizing the Pipeline Difference

We can see this directly in the execution pipelines. Option 2 reduces filter transforms by **50%**:

**Option 1: Range Comparison (8 Filter Transforms)**
```
EXPLAIN pipeline SELECT * FROM products
WHERE name >= 'ClickHouse' AND name < 'ClickHousf';

(Expression)
ExpressionTransform × 4
  (Filter)
  FilterTransform × 8          ← More transforms
    (ReadFromMemoryStorage)
```

**Option 2: SIMD Function (4 Filter Transforms)**
```
EXPLAIN pipeline SELECT * FROM products
WHERE startsWith(name, 'ClickHouse');

(Expression)
ExpressionTransform × 4
  (Filter)
  FilterTransform × 4          ← Fewer transforms
    (ReadFromMemoryStorage)
```

#### The Optimized Query Tree

Based on this analysis, our optimization pass automatically transforms the query tree. The original `like` function node is replaced with a `startsWith` function:

**Before optimization:**
```
WHERE
  FUNCTION function_name: like
    ARGUMENTS
      IDENTIFIER: name
      CONSTANT: 'ClickHouse%'
```

**After optimization:**
```
WHERE
  FUNCTION function_name: startsWith, result_type: UInt8
    ARGUMENTS
      COLUMN: name
      CONSTANT: 'ClickHouse'
```

This transformation is completely transparent to users. They write idiomatic SQL with `LIKE` patterns, and ClickHouse automatically optimizes it to take the fastest execution path. The performance results speak for themselves (see the Performance section below).

## PR#87374: SIMD-Optimized Case-Insensitive Text Search

### The Performance-Critical Inner Loop

Text search functions in ClickHouse execute in what we call the "inner loop"—they're evaluated against millions of row granules in rapid succession. Even tiny CPU cycle savings per comparison multiply dramatically across massive datasets.

This PR introduces new case-insensitive text search functions (`startsWithCaseInsensitive` and `endsWithCaseInsensitive`) with hand-tuned SIMD optimizations. These functions are the foundation that makes the query rewriting in PR#85920 possible for case-insensitive searches.

### Understanding SIMD: Data Parallelism at the Instruction Level

**SIMD** (Single Instruction, Multiple Data) allows a single CPU instruction to operate on multiple data elements simultaneously. Instead of comparing one character at a time, SIMD instructions can compare 16 bytes in a single operation on modern x86 CPUs with SSE2.

For example,
- **Scalar processing**: Compare "C" with "c", then "L" with "l", then "I" with "i"... (one at a time)
- **SIMD processing**: Compare "ClickHouse1234567" against target pattern in just a few instructions (16 bytes at once)

### Optimization Techniques

Implementing high-performance SIMD functions requires careful engineering. We employ several techniques:

#### 1. Separate Fast Paths for ASCII and UTF-8

ASCII characters are always 1 byte, while UTF-8 uses 1-4 bytes per character. By separating these code paths, we avoid unnecessary width checks in the ASCII fast path:

```c++
using CaseInsensitiveComparator = std::variant<
    std::unique_ptr<ASCIICaseInsensitiveStringSearcher>,
    std::unique_ptr<UTF8CaseInsensitiveStringSearcher>>;
```

#### 2. Hoist Constant Comparator Construction

When comparing against a constant pattern like `'ClickHouse'`, we construct the comparator once before the loop, not millions of times inside it:

```c++
// Build comparator ONCE before the loop
const CaseInsensitiveComparator const_comparator =
    constCaseInsensitiveComparatorOf<NeedleSource>(needle_source);

size_t row_num = 0;
while (!haystack_source.isEnd())
{
    // Compare each row with pre-built comparator
    res_data[row_num] = comparator->compare(...);
    row_num++;
}
```

#### 3. Substring-Only Comparison

For affix patterns, we only compare substrings of matching length. No need to scan entire strings when checking prefixes or suffixes:

```c++
// For prefix matching, only compare first N characters
res_data[row_num] = comparator->compare(
    haystack.data,
    haystack.data + pattern_length,  // Only compare prefix
    haystack.data
);
```

#### 4. SIMD Intrinsics for Parallel Comparison

The core case-insensitive comparison uses SSE2 intrinsics to compare 16 bytes at once:

```c++
// Load 16 bytes from input string
const auto v_haystack = _mm_loadu_si128(reinterpret_cast<const __m128i *>(pos));

// Compare against lowercase pattern (16 bytes in parallel)
const auto v_against_l = _mm_cmpeq_epi8(v_haystack, cachel);

// Compare against uppercase pattern (16 bytes in parallel)
const auto v_against_u = _mm_cmpeq_epi8(v_haystack, cacheu);

// Combine results: match if either lowercase OR uppercase matched
const auto v_against_l_or_u = _mm_or_si128(v_against_l, v_against_u);

// Extract comparison results as bitmask
const auto mask = _mm_movemask_epi8(v_against_l_or_u);
```

This processes **16 character comparisons in just a few CPU cycles** instead of iterating through them one by one.

*Reference: [Intel Intrisic Reference](https://www.intel.com/content/dam/develop/external/us/en/documents/18072-347603.pdf)*

### The Result

These optimizations deliver **~38% speedup** on the TPC-H benchmark for case-insensitive pattern matching queries (see Performance section below).

## PR#87789: Fixing INSERT SELECT with Constant Common Table Expressions

### The Bug: Metadata Catalog Resolution in Replicated Environments

Common table expressions (CTEs) are a powerful SQL feature that lets you define temporary named result sets within a query. They're especially useful for complex data transformations:

```sql
WITH transformed_data AS ( 
	-- Some constant expressions here
)
INSERT INTO user_totals
SELECT * FROM transformed_data;
```

ClickHouse's replicated architecture provides both high availability and parallel ingestion performance. When you insert data into a replicated table, ClickHouse distributes the work across replicas to maximize throughput.

However, Issue#85368 revealed a subtle bug: when `INSERT SELECT` queries used CTEs, the query interpreter on each replica failed to resolve the CTE definition in its local metadata catalog. The CTE was defined in the coordinator's context but wasn't properly propagated to replica query interpreters.

### The Root Cause: Missing Context Propagation

In ClickHouse's distributed query execution:
1. The coordinator parses the query and builds the execution plan
2. For replicated inserts, sub-queries are distributed to replicas
3. Each replica interprets and executes its portion of the query

The bug occurred at step 3. CTEs are stored in the query context's metadata catalog. When the coordinator distributed the `INSERT SELECT` to replicas, it sent the query text but didn't ensure the CTE definitions were available in each replica's query context.

Result: Each replica's query interpreter tried to resolve `transformed_data` (from the example above) and failed—because it wasn't in their local catalog.

### The Fix: Ensuring Context Consistency

PR#87789 fixes this by ensuring CTE definitions are properly propagated to replica query contexts before execution. The coordinator now:

1. Identifies all CTEs in the query
2. Serializes their definitions from its metadata catalog
3. Includes them in the execution context sent to each replica
4. Each replica registers these CTEs in its local context before interpretation

This ensures replicas can resolve CTE references during query interpretation, restoring parallel ingestion functionality for queries with CTEs.

### Impact

This fix is critical for production deployments where:
- Data ingestion pipelines use CTEs for complex transformations
- Replicated tables require high-throughput parallel inserts
- Reliability and consistency across replicas is non-negotiable

# Performance: Benchmark Results

Now let's look at the numbers. Here's how these optimizations perform on benchmarks.

## PR#85920: LIKE Rewrite Optimization (~5x Speedup)

As part of PR#85920, we added affix pattern queries to ClickHouse's continuous benchmarking suite. This ensures the CI/CD pipeline monitors performance for any improvements or regressions.

### Results Summary

The optimization delivers approximately **5x speedup** for both prefix and suffix pattern matching:

| Pattern Type | Without Optimization | With Optimization | **Speedup** |
|--------------|---------------------|-------------------|-------------|
| Prefix (`'prefix%'`) | 0.572s | **0.135s** | **4.2x faster** |
| Suffix (`'%suffix'`) | 0.682s | **0.135s** | **5.1x faster** |

*Note: Relative time variance < 0.004 for all measurements, indicating stable, reliable results*

### Benchmark Query Details

```sql
-- Prefix matching: 572ms → 135ms (4.2x improvement)
SELECT count() FROM tab WHERE str LIKE 'prefix%'
SETTINGS optimize_rewrite_like_perfect_affix=1

-- Suffix matching: 682ms → 135ms (5.1x improvement)
SELECT count() FROM tab WHERE str LIKE '%suffix'
SETTINGS optimize_rewrite_like_perfect_affix=1
```

The optimization is enabled by default in ClickHouse 25.10. Users can disable it with `SETTINGS optimize_rewrite_like_perfect_affix=0` if needed.

*Full benchmark report: [PR#85920 Performance Comparison](https://s3.amazonaws.com/clickhouse-test-reports/PRs/85920/b28218b80e7042a42a6d8144292a6e857e0871a1//performance_comparison_arm_release_master_head_3_3/report.html)*


## PR#87374: SIMD Case-Insensitive Search (~38% Speedup)

We benchmarked PR#87374 using the TPC-H benchmark at scale factor 30, which generates a ~30GB `lineitem` table with 240 million rows. This represents a realistic analytical workload.

### Test Query: Case-Insensitive Prefix Matching

We compared three approaches for finding comments starting with "te" (case-insensitive):

| Approach | Execution Time | Throughput | **Speedup vs. Baseline** |
|----------|---------------|------------|-------------------------|
| **Baseline**: `startsWith(lower(...), 'te')` | 0.957s | 251M rows/s | — |
| **Alternative**: `lower(left(..., 2)) = 'te'` | 0.821s | 292M rows/s | 1.17x |
| **SIMD Optimized**: `startsWithCaseInsensitive` | **0.605s** | **397M rows/s** | **1.58x (38% faster)** |

### Query Details

**Baseline: Combine startsWith with lower()**
```sql
SELECT sum(startsWith(lower(l_comment), 'te'))
FROM lineitem;

-- Result: 0.957 sec, 240M rows, 8.22 GB
-- Throughput: 250.86M rows/s, 8.59 GB/s
```

**Alternative: Substring extraction with case normalization**
```sql
SELECT sum(lower(left(l_comment, 2)) = 'te')
FROM lineitem;

-- Result: 0.821 sec, 240M rows, 8.22 GB
-- Throughput: 292.40M rows/s, 10.01 GB/s
```

**SIMD Optimized: Direct case-insensitive comparison**
```sql
SELECT sum(startsWithCaseInsensitive(l_comment, 'te'))
FROM lineitem;

-- Result: 0.605 sec, 240M rows, 8.22 GB
-- Throughput: 396.56M rows/s, 13.58 GB/s  ← Faster

```

### Analysis

The SIMD-optimized `startsWithCaseInsensitive` function delivers:
- **38% faster execution** than the baseline `startsWith(lower(...))`
- **58% higher throughput** (397M vs 251M rows/second)
- Processes 8.22 GB of string data at **13.58 GB/s** vs **8.59 GB/s**

This performance gain comes from avoiding the intermediate lowercase conversion and using hand-tuned SIMD instructions to perform case-insensitive comparison directly.

---

## Real-World Impact

These improvements matter because:

- **Text search is everywhere**: Logs, traces, documents, user-generated content—text data pervades modern analytics
- **Scale multiplies gains**: A 5x speedup on queries processing billions of rows translates to massive time and cost savings
- **Transparency**: Users get automatic optimizations without changing their SQL
- **Production-ready**: Fixes ensure reliable operation at scale

## Looking Forward

Contributing to ClickHouse has been an incredible learning experience for me in:
- Database query optimization and compiler techniques
- Low-level performance engineering with SIMD intrinsics
- Distributed systems and metadata consistency
- Collaborative open-source development with a world-class engineering team

I'm excited to see these optimizations help ClickHouse users extract insights faster from their data. The journey of making the fastest database even faster continues!

---

*For questions or discussions about these contributions, feel free to reach out or join the conversation in the linked GitHub issues and pull requests.*
