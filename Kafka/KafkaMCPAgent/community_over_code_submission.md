# Agentic Analytics and Performance Tuning with Apache Kafka

## Short Description

How can AI transform low-level data infrastructure into intuitive solutions for complex, high-level domains? In this session, we explore the design of a Kafka MCP Agent that enables agentic workflows for real-time analytics and performance tuning—tasks traditionally requiring deep human expertise.

## Abstract

Apache Kafka excels as streaming infrastructure, but solving high-level problems on top of it still demands deep human expertise. A chef on a cruise ship wants to know "What are my most popular entrees tonight?" A storage engineer needs to find which connector configuration maximizes throughput. These are analytical and performance questions — yet today's Kafka tooling, including emerging MCP (Model Context Protocol) integrations, only exposes low-level operations like producing records or managing consumer groups. There is a fundamental gap between the high-level questions real users ask and the primitives that current tools provide.

This session presents a Kafka MCP Agent that closes this gap through agentic workflows. Rather than wrapping Kafka's client API in a chatbot interface, the agent composes multiple domain-specific tool layers — streaming, relational, and visual — to tackle problems no single layer can solve alone. The architecture connects an LLM to Kafka via the Model Context Protocol, then augments it with a relational engine (DuckDB/chDB) for analytics over streaming data and a visualization layer (Matplotlib) for rendering insights. This deliberate tool composition is what transforms low-level data infrastructure into an intuitive problem-solving assistant.

We demonstrate two agentic workflows. First, real-time analytics: a non-technical user asks natural language questions about business data in Kafka topics, and the agent autonomously consumes events, materializes them into queryable tables, runs SQL, and returns visual answers — no Flink, no KSQL, no data pipeline required. Second, performance tuning: the agent orchestrates benchmarking workflows for a Kafka storage connector, systematically exploring configuration parameters to find optimal throughput, replacing the tedious guess-benchmark-repeat cycle.

We then examine what makes this work and where it breaks. A key design insight: while the LLM remains outside your control, the tools and their composition are decisions entirely within it. Attendees will leave with practical architectural patterns for building agentic systems over streaming platforms, and an honest assessment of the tradeoffs involved.
