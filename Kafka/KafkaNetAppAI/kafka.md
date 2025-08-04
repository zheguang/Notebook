# Kafka + NetApp <sup>^AI</sup>
Building a ️Connector Better than Market

🍺🐿️   🆚   🦀

[Guang Zhao](https://zheguang.github.io)

©️ 2025 NetApp, Inc. All rights reserved.

---

## Data

🍺 NetApp Ontap/StorageGrid

🐿️ Kafka pipeline

❓ How to connect the two

--

## Interface

🍺 <ins>S</ins>imple (object) <ins>s</ins>torage <ins>s</ins>ervice (S3)

🐿️ Kafka Connect

🦀 Aiven/Confluent/... Connectors

--

## End of story?
Just a simple config away?

---

## Problem

✅ Parquet, CSV, JSON

❓ 1GB ~ 1TB per object

--

## Limits 🍺 🦀

--

### 🍺 Ontap/StorageGrid

Read: fail `GetObject*` >10GB

Write: both StorageGrid and AWS recommend
* 5GB for single `PutObject`
* 5TB max

--

### 🦀 Market Connectors

--------------------------------------------------------
| Confluent        | Aiven                  | Lenses   |
| ---------        | ------                 | ------   |
| <2GB<sup>a</sup> | <min(Instance,10GB)<sup>a,c,d</sup>  | <10GB<sup>d</sup>    |
| Closed           | -                      | -        |
| -                | Utf-8 only<sup>b</sup> | -        |
| -                | Inefficient conversion | -        |
--------------------------------------------------------
* a. Parquet
* b. Text
- c. "Download first" (Anti-Stream)
- d. "Single stream" (S3 limit)

---

## Problem = scale 🏋️

---

## Let's Solve it

Just ask Customer to split their data

End of story?

--

## No, let's Really Solve it
Simple I/O abstraction ➡️ data formats

Scalable I/O implementation ➡️️ external storage

---

## What affects Scalability

--

### Data Formats ➡️ Access Patterns

| CSV, (ND)JSON | Parquet     |
| ------------- | -------     |
| Line          | Columnar    | 
| Sequential    | Semi Random |

<!---
--

### Text: Sequential Access

```csv
1,first,row     *
2,second,row    |
...             |
N,nth,row       v
```
```json
{ object: 1 }   *
{ object: 2 }   |
...             |
{ object: N }   v
```
Scan the file from the first to the last line.
-->

--

### Parquet: Semi Random Access


```text
+----------------+       
| Chunk 1, Col 1 <-------+-------+
| Chunk 1, Col 2 |       | Row 1 |
| C...           |       | Row 2 |
| Chunk 1, Col N |       | ...   |
+----------------+       | Row M |
| Chunk 2, Col 1 |       +-------+
| C...           |
| Chunk 2, Col N |
+----------------+
...
| Chunk N        |
+----------------+
| File Metadata  |  Imagine cutting a tofu
+ ---------------+
```
1. jump to the end to read Metadata
2. jump back to chunk, read row in multiple offsets

--

### Kafka Connect(or)

- Poll external records by batches
- Convert types
- Publishes records to Kafka topic

External systems and data formats, are what?

--

### \#0: All are ... Byte Streams

```java
abstract class java.io.InputStream {
    ...
    // Only forward
    long skip(long n) throws IOException;

    // Let's not get into mark() and reset()...
}
```

Can't support Parquet

Can't handle large S3 object


---

## Break the Abstraction

--

### \#1: With Random Access
```java
class RandomAccessInputStream extends j.i.FilterInputStream {
    ...
    // Forward and backward
    void seek(long offset) throws IOException;
}
```

~~Can't support Parquet~~

Can't handle large S3 object

--

### \#2: Broken into Extents

```java
class ExtentInputStream extends RandomAccessInputStream {
    long extentSize;
    long extentOffset;
    ...
}
```

--

#### Diagram

```text
+--------------------------------------------------------+
| byte[0], byte[1], ...                        byte[N]   |
+================+================+=====+================+
| ext[0]         | ext[1]         | ... | ext[M]         |
+----------------+----------------+-----+----------------+
| S3.GetRange[0] | S3.GetRange[1] | ... | S3.GetRange[M] |
+----------------+----------------+-----+----------------+
```

* $$ byte[i] = ext[i / size][i \mod size ] $$
* multiple smaller reads on a large object

--

### Power of Abstraction

```text
                +--------------+
                |  Connector   |
                +-------+------+
                        |
     +------------------+-----------------+-----+
     | Parquet Decoder  | Unicode Decoder | ... |
     +------------------+-----------------+-----+
                        |
+-----------------------+--------------------------------+
| byte[0], byte[1], ...                        byte[N]   |
+================+================+=====+================+
| ext[0]         | ext[1]         | ... | ext[M]         |
+----------------+------+---------+-----+----------------+
                        |
        +---------------+-------------+--------+
        | Object Store  | File System | ...    |
        +---+-----------+-------------+--------+
            |
        +---+------+
        |  S3      |
        +----------+-------------+--------+-----+
        | 🍺 Ontap | StorageGrid | 😄 Aws | ... |
        +----------+-------------+--------+-----+
                  /
+----------------+----------------+----------------------+
| S3.GetRange[0] | S3.GetRange[1] | ... | S3.GetRange[M] |
+----------------+----------------+----------------------+
```
* Bytes 🔄 data formats
* Extents 🔄 external systems

--

### Extensible

```text
        Data Formats
            ^
            |
    Parquet |
            |
    Text    |
            |   🍺 Ontap  StorageGrid  😄 Aws
            +----------------------------------> Systems
    Extent-Stream
```

--

### Optimization

Extent 🔈 access pattern ➡️ storage

Storage may optimize, such as caching or prefetching

--

### Optimal Extent Size ❓

Too small: consume resources

Too big: waste unread bytes; S3 limit

🤖 🆚 🤷

How to find out? For each system, each format...

---

## (Auto-)Tune Our Connector

<!---
--

### External I/O characteristics

- Changing external systems and workload
- Documentation inaccuracies
- Human intuition unreliable

❓ Find the best Connector performance
-->

<!---
--

### Unknowns
- External system designs & changes
- Documentation inaccuracies
- Human intuition unreliable

-->

--

### Machine Learning Problem

$$ \arg\max_{param} P(connector \| param, sys, work) $$

- <ins>param</ins>eters: extent size
- <ins>Sys</ins>tem: S3, Ontap, StorageGrid, Local
- Representative <ins>work</ins>load?
- P, model: throughput, or latency

---

## 🤖 AI Attempt \#1

--

### Code-only AI

👦: Hey 🤖, given my code, what's the optimal parameter?

$$ \arg\max_{params} P(connector \| param) $$

--

### AI correct?
🤖:
"Disk block sizes commonly are 512B or 4KB. Set 1KB."

🤖:
"S3 recommends PutObject limit 5GB, max 5TB, Set 5GB."

---

## Benchmark
Let's do the hard work

--

### TPC-H Dataset
-------------------------------------------------------
|         | Parquet Small | Parquet Large | CSV Small |
| --      | --------      | -------       | --------- |
|Table    | Customer      | Lineitem      | Customer  |
|Scale    | 10            | 30            | 10        |
|Rows     | 1.5M          | 37.23B        | 1.5M      |
|Compress | 1:2           | 1:2           | 1:1       |
|Size(B)  | 118M          | 6.3G<sup>a,b</sup>          | 237M      |
-------------------------------------------------------------------

- a. Beyond Confluent limit
- b. Aiven min instance disk space

--

### Workload
```python
num_polls = 10
batch_size = 128
for i in range(num_polls):
    poll(batch_size)
```

--

### 🍺 StorageGrid, time (ms)
------------
| Extent(B) | Csv Small | Parquet Small | Parquet Large   |
| ------    | -------:  | ------------: | ------------:   |
| 1K        | 480,501   | >1min         | >1min           |
| 4K        | 470,438   | >1min         | >1min           |
| 1M        | 4,695     | 17,134        | 12,292          |
| 4M        | 2,967     | 8,630         | 9,304           |
| 1G        | 2,937     | 6,782         | 6,377           |
| 4G        | 2,669     | 6,242         | 6,196           |
------------
* std ~ 5% mean

--

### 🍺 StorageGrid vs 😃 AWS, Time (ms)

------------------
| Extent(B) | StorageGrid<sup>a</sup> | Aws<sup>a</sup>     |
| --------- | ----------: | --:     |
| 1K        | 480,501     | 573,496 |
| 1M        | 4,695       | 4,813   |
| 1G        | 2,937       | 2,895   |
------------------
* a. CSV Small
- std ~ 5% mean


--

### Takeaway

* Our Connector scales better than Confluent
* StorageGrid vs Aws comparable
* Optimal extent size != AI suggests
    - depends on formats, maybe systems
    - requires internal knowledge

---

## 🤖 AI Attempt \#2

--

### Stochastic gradient descent

```python
num_polls = 10
batch_size = 128
x = extent_size
for i in range(num_polls):
    cost = model(poll(batch_size), x)
    grad = gradient(model, cost, x)
    x = step(grad, x)
```
Convergence: found an extent size with fastest poll

--

### Problems

Model is non-differentiable.

It assumes each poll time is stable given extent.

But there is no bound.

--

![poll.png](./poll.png)

--

### 🤷 No guarantee 
Based on local info, cannot converge to optimal extent

---

## 🤖 AI Attempt \#3

Combine LLM (\#1) and ML (\#2)

--

### Agent for Benchmark + Tune

```python
while True:
    extent_sizes, num_polls = Llm.generate_response(
        'Propose candidate extent sizes, observation window', 
        context)

    costs = []
    for x in extent_sizes:
        costs += repeat(
            num_polls, 
            model(poll(batch_size), x))

    better_extent_size = find_min(costs, extent_sizes)

    context.add(better_extent_size)
```
AI gets both general and specific context

--

### Result

Experimental... ⌛

---

## Features

* Unlimited size
* Extensible formats
* Extensible endpoints
* (Autotune agent ⌛)

- Unicode
- Aync object discovery
- Nested prefixes
- 1-1 Type conversion

👍 Differentiate from Market {🦀,...}

--

## Heads up

- 👁️ Cassandra Parquet/Avro Transformer by [Stefan](stefan.miklosovic@netapp.com)

- Operational to analytical:
```
👁️ Cassandra -- Parquet -- 🐿️ Kafka
                  |        
                  |
                  X
```

---

## Thanks

* Anup: Testing
* Amanda: Organizing
* Carlos: Organizing
* Justin: Organizing

- Nilkua: NetApp configs
- Tharindu: Native format
- Varun: Organizing
- Win: Ontap

* Team Kafka: Review PR
* Team Open Source: Discussion
* Team PoC: Customer & Product

🤷 All technical errors are mine.

---

## Thoughts?

💻 https://github.com/instaclustr/kafka-connect-connectors

▶ Contribution welcome!

📧 Guang.Zhao@netapp.com

Slack #opensauce

🏢 Melbourne
