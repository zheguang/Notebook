# My Agent Kafka
by Model Context Protocol

[Guang Zhao](https://zheguang.github.io)

©️ 2025 NetApp, Inc. All rights reserved.

--

## Agentic `___`?

E.g. Housing agents, car mechanics, doctors

- Interact with high-level language
- Master domain-specific tools
- Hide away domain complexity


--

## Kafka Agentic `___`?

Problem: tools are too low-level

--

### Too low-level

```ascii
           🧑‍🍳
           | high-level language
           ▼
      Kafka Agent for ___?
          |      ▼
       ❌ |     ___? ✅   (Gap)
          |      |
          ▼      ▼
        Kafka Operations  (Current MCP tools)
           |
           ▼
        Kafka Server
```

* ❌ "Can you produce X to the topic Y hashed by Z"

* ✅ "How are my sales doing today?"

---

## High-level problems

My Kafka Agentic `___`

- Real-time analytics
- Performance tuning

---

## Real-time analytics

![cruise-food.jgg](./cruise-food.jpg)

--

### Chef asks...

```text
    What are my most popular entrees tonight?

    How many do I need to prepare tomorrow?
```

😩

Write SQL? 

Hire a Kafka programmer?

😐: Use my Kafka agent?

--

### Kafka up for analytics?

```text
    "Need to bridge from the operational to the analytical"

        -- Confluent engineer at Current'25
```

--

### Data model gap
```ascii
    Less programmable                            SQL
           ❌                                     ▼
Log:   | Event 1 |     <==>    Table:       | col_1 | col_2 ..
       | Event 2 |                          +-------+------ 
       | ..      |                    row_1 | ..    | ..
            |                         row_2 | ..    | ..
            ▼                          ..
```
* Apache Flink
* Kafka Streams
* KSQL (retiring)

- TableFlow (Confluent)
- Iceberg topic (Aiven)
- Kafka table engine (ClickHouse)

😩: still not easy for chef

--

### ClickHouse = data warehouse + LLM

- Collects 2PB business data
- Analyst: 
    - "Which onboarding steps correlate with high 90-day retention"

```text
    No longe writing SQLs by hand ... Huge productivity gain 

         - Mihir Gokhale, PM, Open House'25
```

😐: Kafka = Streams + LLM?

---

##  Performance tuning

![f1-tuning.png](f1-tuning.png)

--

### My Kakfa-ONTAP connector
```text
Which ONTAP block size maximizes throughput?
a. 4KB    b. 4MB    c. 4GB
```

😐: Guess -> benchmark -> repeat

🤖: Orchestrate for me?

---

## Demo

- Analyze business data
- Tune cluster performance

👻 Non-deterministic "bugs"?

---

## How does it work?

--

### How complicated can it be?

- Input: question
- Compute: ...
    - Programs: models, machine code
    - CPU, storage, network
- Ouput: answer

How different from "trad coding"?

--

### System view

```ascii
😐: input

🤖:
    +-------+     +-------+     +---------+
    |  LLM  |<--->|  MCP  |<--->|  Tools  |
    +-------+     +-------+     +----+----+
                                     |
             +-----------------------+---------------+
             |                       |               |
   +---------+--------+  +-----------+----+  +-------+-------+
   |        Streaming |  | Relational     |  |   Visual      |
   +------------------+  +----------------+  +---------------+
   | Kafka Client API |  | DuckDB / chDB  |  | Matplotlib    |
   +------------------+  +----------------+  +---------------+
🤖: answer

```

---

## Takeaway

- Agent to solve high-level problems with Kafka
- High-level problems needs more complex tools
- Complexity of tools comes from chosen combination
- LLM is outside of your control<sup>`*`</sup>, but tools are within

---

## Thoughts?

- [https://github.com/instaclustr/agent-kafka](https://github.com/instaclustr/agent-kafka)
- guang@netapp.com
- [#opensauce](), [#open-ai]()
