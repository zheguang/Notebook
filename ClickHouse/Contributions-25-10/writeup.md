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

Many text search involves matching certain prefix or suffix, for example
```
SELECT ... FROM products WHERE description LIKE 'ClickHouse%';
```
will match products ClickHouse Client, ClickHouse Backup, and ClickHouse Server, and so on.

During query execution, these `LIKE` patterns will be compiled into regular expression, and computed via automata.  This strategy works for general patterns, however is too slow for simple prefix and suffix matching.  
These affix patterns can be checked more efficiently by comparing substrings.  This PR is for creating this fast path.  The query performance improvement can be around 5 times.

Like many relational databases and compilers for programming languages, ClickHouse parses SQL queries into trees of expresssions, and performs query optimization to transform the trees into forms to execute with higher efficiency.

Query optimization in ClickHouse is done via passes.  This PR therefore creates a new pass to transform any `like` expressions present in query trees, while preserving the semantics. 

Example of query tree:
```
```

There are several options to rewrite `like` expression into.  Take an example of `description LIKE 'ClickHouse%'` --> `'ClickHouse':
- Option 1: range comparison: `'ClickHouse' <= description AND description < 'ClickHousf'`
- Option 2: as SIMD-optimized functions: `startswith(description, 'ClickHouse')`

In a typical programming language, you wouldn't worry about the difference. But for analytical databases like ClickHouse, the performance difference between this two options can be about 5 times.  

The option 2 is more efficient due to requiring less computation and less pipeline complexity.  Option 1 requires two string comparisons, one for comparing lowerbound and one for comparing upperbound.  But ClickHouse does not work on one row at a time.  Instead, ClickHouse processes a bunch of rows, called the granule together for each operator, and passes the results as a new granule to the next operator in the pipeline.  This means that Option 1 also needs to store the intermediate result after the comparison with the lowerbound, and then pass to the comparison with the upperbound.  This puts memory pressure on the execution.

Option 2 howver, requires less computation. Internally, the function `startswith` is hand-written with intrinsic instructions for SIMD optimization, such that the string prefix is only compared once to multiple strings.  This also means there is no need to store and pass around intermediate data.  

### Performance

#### Benchmark of LIKE rewrite PR#85920

Median time, s	Relative time variance	Test	#	Query
0.572	0.003	like_perfect_affix_rewrite	0	SELECT count() FROM tab WHERE str LIKE 'prefix%' SETTINGS optimize_rewrite_like_perfect_affix=0
0.135	0.003	like_perfect_affix_rewrite	1	SELECT count() FROM tab WHERE str LIKE 'prefix%' SETTINGS optimize_rewrite_like_perfect_affix=1
0.682	0.003	like_perfect_affix_rewrite	2	SELECT count() FROM tab WHERE str LIKE '%suffix' SETTINGS optimize_rewrite_like_perfect_affix=0
0.135	0.004	like_perfect_affix_rewrite	3	SELECT count() FROM tab WHERE str LIKE '%suffix' SETTINGS optimize_rewrite_like_perfect_affix=1


#### Bechmark of SIMD case insensitive search PR#87374
StartsWith + lower
:) select sum(startsWith(lower(l_comment), 'te')) from lineitem;

SELECT sum(startsWith(lower(l_comment), 'te'))
FROM lineitem

      1 row in set. Elapsed: 0.957 sec. Processed 240.01 million rows, 8.22 GB (250.86 million rows/s., 8.59 GB/s.)
Substring + lower
 :) select sum(lower(left(l_comment, 2)) == 'te') from lineitem;

SELECT sum(lower(left(l_comment, 2)) = 'te')
FROM lineitem

      1 row in set. Elapsed: 0.821 sec. Processed 240.01 million rows, 8.22 GB (292.40 million rows/s., 10.01 GB/s.)
StartsWithCaseInsensitive nonSIMD
My spike implementation: https://github.com/zheguang/ClickHouse/tree/starts-endswith-case

:) select sum(startsWithCaseInsensitive(l_comment, 'te')) from lineitem;

SELECT sum(startsWithCaseInsensitive(l_comment, 'te'))
FROM lineitem

1 row in set. Elapsed: 0.605 sec. Processed 240.01 million rows, 8.22 GB (396.56 million rows/s., 13.58 GB/s.)
Given this observation, I think I can separate out this work of starts/endsWithCaseInsensitive with SIMD to a separate PR. After that I will add ILIKE rewrite... Is that reasonable?

# Links
- [Performance benchmark for PR#85920](https://s3.amazonaws.com/clickhouse-test-reports/PRs/85920/b28218b80e7042a42a6d8144292a6e857e0871a1//performance_comparison_arm_release_master_head_3_3/report.html)
