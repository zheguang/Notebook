# Apache Kafka Diskless in the Open
Running Aiven's Inkless for KIP-1150 with only Open Source Tools

[Guang Zhao](https://zheguang.github.io)

©️ 2025 NetApp, Inc. All rights reserved.

--

## Once upon a time

Let there be many cheap disks, and much big data.
Then came MapReduce Hadoop, and streaming Kafka.

Preserving state is all there about.
Partition state to scale.
Crossing AZs if you want to sleep well.

## What if storage never fails

Amazon simple storage service (S3)

"Cloud native": separation of compute and storage

## Towards cloud-native Kafka

```
+--------------+     +-------------+
| Broker 1     |     | Broker 2    |
|              |     |             |     +/- Broker N?
| Log on disk  |     | Log on disk |
+--------------+     +-------------+
```

```
+--------------------+     +--------------------+
| Broker 1           |     | Broker 2           |
|                    |     |                    |   +/- Broker N?
| Recent Log on disk |     | Recent Log on disk |
+------+-------------+     +-+------------------+
       |                     |
+------+---------------------+----------+
|      Old Log in object storage        |
+---------------------------------------+
```

```
+--------------+     +--------------+
| Broker 1     |     | Broker 2     |
|              |     |              |   +/- Broker N?
| Log in Cache |     | Log in Cache |
+------+-------+     +-------+------+
       |                     |
+------+---------------------+------+
|      Log in object storage        |
+-----------------------------------+
```


