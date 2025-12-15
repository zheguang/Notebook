# My contributions for ClickHouse 25.10:

Three PRs merged for changelog:
- New text search with SIMD-based functions: [PR#87374](https://github.com/ClickHouse/ClickHouse/pull/87374)
- New query optimization on pattern matching queries: [Issue#71421](https://github.com/ClickHouse/ClickHouse/issues/71421), [PR#85920](https://github.com/ClickHouse/ClickHouse/pull/85920)
- Fix common table expression bug for insert query: [Issue#85368](https://github.com/ClickHouse/ClickHouse/issues/85368), [PR#87789](https://github.com/ClickHouse/ClickHouse/pull/87789)

# Why

## PR#87374 and PR#85920
- ClickHouse is the fastest analytics database. My contributions make it even faster.
- Text search is common for many workloads many data have textual representations.  Workloads such as searching logs and traces in observability, retrieiving information from documents, data transformation in data warehouse, and generative AI applications.
- Faster text search will make all these applications run faster, save time and resources, unlocking more analytical insights, and leading to faster decisions making in both business and AI applications.

- ClickHouse is fast thanks to its query optimization, which has recently been reworked [link](). How do we make use of the new query optimizer to further push query efficiency?
- For text search, complex pattern matching requires more compute-intensive regular expression evaluation.  However, for simpler patterns such as affix (prefix and suffix) matching, simple substring comparison optimized for single-instruction-multiple-data (SIMD) can be several times faster.  

## PR#87789
- ClickHosue is great for ingesting large amount of data.  The data ingestion facility in ClickHouse is crucial for shoveling large amount of data from external sources such as Apache Kafka and Apache Iceberg into ClickHouse native format for most efficiency. 
- ClickHouse is often deployed as a replicated, share-nothing database. Data ingestion in this environment needs to ensure metadata consistency among all shards and replicas.  ClickHouse further optimizes ingestion in this environemnt by parallelizing data insertion among replicas.  However, this bug appears when the query interpreter on each replica fails to resolve the common table expression in its metadata catalog.  

# Deep dive

## PR#85920 Rewrite `like` expression for affix patterns

Many text search involves matching certain prefix or suffix, for [example](https://fiddle.clickhouse.com/4a0ba187-a260-49f9-afe5-af6c29f1831e):
```
SELECT count(*) FROM products WHERE description LIKE 'ClickHouse%';
```
will match products ClickHouse Server, ClickHouse Local, and ClickHouse MCP, and so on, but not chDB.

During query execution, these `LIKE` patterns will be compiled into regular expression, and computed via automata.  This strategy works for general patterns, however is too slow for simple prefix and suffix matching.  
These affix patterns can be checked more efficiently by comparing substrings.  This PR is for creating this fast path.  The query performance improvement can be around 5 times.

Like many relational databases and compilers for programming languages, ClickHouse parses SQL queries into trees of expresssions, and performs query optimization to transform the trees into forms to execute with higher efficiency.

Query optimization in ClickHouse is done via passes.  This PR therefore creates a new pass to transform any `like` expressions present in query trees, while preserving the semantics. 

Example of query tree:
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
          CONSTANT id: 7, constant_value: \'ClickHouse%\', constant_value_type: String
```

There are several options to rewrite `like` expression into.  Take an example of `description LIKE 'ClickHouse%'` --> `'ClickHouse':
- Option 1: range comparison: `'ClickHouse' <= description AND description < 'ClickHousf'`
- Option 2: as SIMD-optimized functions: `startswith(description, 'ClickHouse')`

In a typical programming language, you wouldn't worry about the difference. But for analytical databases like ClickHouse, the performance difference between this two options can be about 5 times.  

The option 2 is more efficient due to requiring less computation and less pipeline complexity.  

First consider option 1. Option 1 requires two string comparisons, one for comparing lowerbound and one for comparing upperbound.  But ClickHouse does not work on one row at a time.  Instead, ClickHouse processes a bunch of rows, called the granule together for each operator, and passes the results as a new granule to the next operator in the pipeline.  This means that Option 1 also needs to store the intermediate result after the comparison with the lowerbound, and then pass to the comparison with the upperbound.  This puts memory pressure on the execution.

Option 2 howver, requires less computation. Internally, the function `startswith` is hand-written with intrinsic instructions for SIMD optimization, such that the string prefix is only compared once to multiple strings.  This also means there is no need to store and pass around intermediate data.  

For our running example, we can compare the filter transforms and see that Option 2 cuts down the number of filter transforms by a half:
```
EXPLAIN pipeline SELECT * FROM products where name >= 'ClickHouse' and name < 'ClickHousf';

(Expression)
ExpressionTransform × 4
  (Filter)
  FilterTransform × 8
    (ReadFromMemoryStorage)

EXPLAIN pipeline SELECT * FROM products where startswith(products, 'ClickHouse');

(Expression)
ExpressionTransform × 4
  (Filter)
  FilterTransform × 4
    (ReadFromMemoryStorage)
```

We can also see the difference in query plan's actions, where Option 2 reduces execution actions
```
Expression ((Project names + Projection))
Actions: INPUT : 0 -> __table1.uid Int16 : 0
         INPUT : 1 -> __table1.name String : 1
         INPUT : 2 -> __table1.port Int16 : 2
         ALIAS __table1.uid :: 0 -> uid Int16 : 3
         ALIAS __table1.name :: 1 -> name String : 0
         ALIAS __table1.port :: 2 -> port Int16 : 1
Positions: 3 0 1
  Filter ((WHERE + Change column names to column identifiers))
  AND column: greaterOrEquals(__table1.name, \'ClickHouse\'_String)
  Actions: INPUT : 0 -> name String : 0
           COLUMN Const(String) -> \'ClickHouse\'_String String : 1
           FUNCTION greaterOrEquals(name : 0, \'ClickHouse\'_String :: 1) -> greaterOrEquals(__table1.name, \'ClickHouse\'_String) UInt8 : 2
  Positions: 2 0 2
  Filter column: and(greaterOrEquals(__table1.name, \'ClickHouse\'_String), less(__table1.name, \'ClickHousf\'_String)) (removed)
  Actions: INPUT : 1 -> uid Int16 : 0
           INPUT : 3 -> port Int16 : 1
           INPUT : 2 -> name String : 2
           COLUMN Const(String) -> \'ClickHousf\'_String String : 3
           INPUT : 0 -> greaterOrEquals(__table1.name, \'ClickHouse\'_String) UInt8 : 4
           ALIAS uid :: 0 -> __table1.uid Int16 : 5
           ALIAS port :: 1 -> __table1.port Int16 : 0
           ALIAS name : 2 -> __table1.name String : 1
           FUNCTION less(name :: 2, \'ClickHousf\'_String :: 3) -> less(__table1.name, \'ClickHousf\'_String) UInt8 : 6
           FUNCTION and(greaterOrEquals(__table1.name, \'ClickHouse\'_String) :: 4, less(__table1.name, \'ClickHousf\'_String) :: 6) -> and(greaterOrEquals(__table1.name, \'ClickHouse\'_String), less(__table1.name, \'ClickHousf\'_String)) UInt8 : 3
  Positions: 3 5 1 0
    ReadFromMemoryStorage

Expression ((Project names + Projection))
Actions: INPUT : 0 -> __table1.uid Int16 : 0
         INPUT : 1 -> __table1.name String : 1
         INPUT : 2 -> __table1.port Int16 : 2
         ALIAS __table1.uid :: 0 -> uid Int16 : 3
         ALIAS __table1.name :: 1 -> name String : 0
         ALIAS __table1.port :: 2 -> port Int16 : 1
Positions: 3 0 1
  Filter ((WHERE + Change column names to column identifiers))
  Filter column: startsWith(__table1.name, \'ClickHouse\'_String) (removed)
  Actions: INPUT : 0 -> uid Int16 : 0
           INPUT : 1 -> name String : 1
           INPUT : 2 -> port Int16 : 2
           COLUMN Const(String) -> \'ClickHouse\'_String String : 3
           ALIAS uid :: 0 -> __table1.uid Int16 : 4
           ALIAS name : 1 -> __table1.name String : 0
           ALIAS port :: 2 -> __table1.port Int16 : 5
           FUNCTION startsWith(name :: 1, \'ClickHouse\'_String :: 3) -> startsWith(__table1.name, \'ClickHouse\'_String) UInt8 : 2
  Positions: 2 4 0 5
    ReadFromMemoryStorage
```

Given the analysis above, we conclude that Option 2 is the optimal approach. 
Following Option 2, our example's query tree is optimized into:
```
QUERY id: 0
  PROJECTION COLUMNS
    ...
  PROJECTION
    ...
  JOIN TREE
    ...
  WHERE
    FUNCTION id: 6, function_name: startsWith, function_type: ordinary, result_type: UInt8
      ARGUMENTS
        LIST id: 7, nodes: 2
          COLUMN id: 8, column_name: name, result_type: String, source_id: 3
          CONSTANT id: 9, constant_value: \'ClickHouse\', constant_value_type: String
```

Next, we will see how this difference in query plans result in substantial perforamnce gain.

### Performance

#### Benchmark of LIKE rewrite PR#85920

| Median time, s	| Relative time variance	| Query |
| --------------------- | ----------------------------- | ----- |
| 0.572	| 0.003	| SELECT count() FROM tab WHERE str LIKE 'prefix%' SETTINGS optimize_rewrite_like_perfect_affix=0
| 0.135	| 0.003	| SELECT count() FROM tab WHERE str LIKE 'prefix%' SETTINGS optimize_rewrite_like_perfect_affix=1
| 0.682	| 0.003	| SELECT count() FROM tab WHERE str LIKE '%suffix' SETTINGS optimize_rewrite_like_perfect_affix=0
| 0.135	| 0.004	| SELECT count() FROM tab WHERE str LIKE '%suffix' SETTINGS optimize_rewrite_like_perfect_affix=1

- Source: [Performance benchmark for PR#85920](https://s3.amazonaws.com/clickhouse-test-reports/PRs/85920/b28218b80e7042a42a6d8144292a6e857e0871a1//performance_comparison_arm_release_master_head_3_3/report.html)


#### Bechmark of SIMD case insensitive search PR#87374

1. StartsWith + lower
```
SELECT sum(startsWith(lower(l_comment), 'te'))
FROM lineitem

      1 row in set. Elapsed: 0.957 sec. Processed 240.01 million rows, 8.22 GB (250.86 million rows/s., 8.59 GB/s.)
```

2. Substring + lower
```
SELECT sum(lower(left(l_comment, 2)) = 'te')
FROM lineitem

      1 row in set. Elapsed: 0.821 sec. Processed 240.01 million rows, 8.22 GB (292.40 million rows/s., 10.01 GB/s.)
```

3. StartsWithCaseInsensitive
```
SELECT sum(startsWithCaseInsensitive(l_comment, 'te'))
FROM lineitem

1 row in set. Elapsed: 0.605 sec. Processed 240.01 million rows, 8.22 GB (396.56 million rows/s., 13.58 GB/s.)
```
