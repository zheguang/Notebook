# Apache Kafka Diskless in the Open
Contributing to KIP-1150

[Guang Zhao](https://zheguang.github.io)

©️ 2026 NetApp, Inc. All rights reserved.

April 24, 2026

--

## Diskless Topic

```
Classic            Diskless
+----------+       +----------+
| Broker   |       | Broker   |
|   Disk   |       +----------+
| +-----+  |           |
| | Log |  |          Network
| +-----+  |           |
+----------+       +--------------+
                  | Cloud Storage |
                  |               |
                  | +-----+       |
                  | | Log |       |
                  | +-----+       |
                  +---------------+
```

--

Why is this a good idea?

Does the idea solve everything?

--

## Contributing to KIP-1150

Beyond Aiven's Inkless fork

This talk:
- Open source deployment + benchmark
- AWS-native, FSx ONTAP
- Reviews, understanding

More: Operations, Business, ...

--

## Also in this talk

- FSx ONTAP
- Kubernetes
- Grafana/Prometheus
- Postgres
- (Even) ClickHouse

Everyone can contribute!

---

## Once upon a time

Let there be cheap disks, and big data.
Then came Hadoop, and Kafka.

Preserving state is all about.
With partition we scale.
Crossing AZs if you want to sleep well.

Without AI,
A human

--

## What if storage never fails?

Amazon Simple Storage Service (S3)
- "11 nines" durability
- 1 in 10B objects lost over 10K years

Elastic block storage: "3 nines", 1 in 1000 fail yearly
Disk: 98%, 1 in 50 fail yearly

--

## ... and infinite bandwidth?

S3: unlimited

EBS: 1-4GB/s, 0.3M iops
SSD: 1-7GB/s, 2M iops

## ... and cheaper?

S3: 1x

EBS: 4x
SSD: 6x

--

## But higher latency?

S3: 100ms

EBS: 1ms
SSD: <1ms

--

## What if FSxN combines both extremes...

FSxN: 72GB/s, >2M iops, <1ms, 5x cost

--

## "Cloud native"

Solve durability problem once cheaply
Forucs on performance by scaling

Separation of compute and storage

---

## Towards cloud-native Kafka

Classic topic

```
+--------------+     +-------------+
| Broker 1     |     | Broker 2    |
|              |     |             |     +/- Broker N?
| Log on disk  |     | Log on disk |
+--------------+     +-------------+
```

Log = topic partition

--

Tiered storage topic

```
+--------------------+     +--------------------+
| Broker 1           |     | Broker 2           |
|                    |     |                    |   +/- Broker N?
| Recent Log on disk |     | Recent Log on disk |
+------+-------------+     +-+------------------+
       |                     |
+------+---------------------+----------+
|      Old Log in cloud storage         |
+---------------------------------------+
```
--

Diskless topic

```
+--------------+     +--------------+
| Broker 1     |     | Broker 2     |
|              |     |              |   +/- Broker N?
| Log in Cache |     | Log in Cache |
+------+-------+     +-------+------+
       |                     |
+------+---------------------+------+
|      Log in cloud storage         |
+-----------------------------------+
```

Scale for performance/cost easily

Broker failure is no longer scary

---

## Concept drift from Classic Kafka?

--

### Leader

No more. Brokers don't own logs:
- No more partition leader
- Any broker can write to any partition

--

### Ordering

Same. 

To the same partition:
- Messages have total order (Quiz: determined by who?)
- Messages from an "idempotent" producer's appear in send order

No ordering across partitions

--

Answer: Coordination. Postgres

--

### Durability

Handled by S3. Similar multi-zone durability with Classic 3-AZ 3-RF setup.

### Replication

Changed. Only for routing.

Replication factor: "Only N brokers handle partition X"
Acknowledgement: 0 = no durability; >0 = durable
In sync replica: no effect

--

### File

No longer maps 1-1 to a partition's segment.

Broker buffer: messages across partitions

S3 file: interleave messages across partitions

Coordinator: log order to upload order

Merger: reorganizes upload order (Quiz: reminds you of who?)

--

Answer: ClickHouse's MergeTree.

---

## Example

```
To the same partition:

Producer P1 sends: [a, b, c]     Producer P2 sends: [x, y]

Broker B1 buffers: [a, b, c]     Borker B2 buffers: [x, y]

B1 upload file: F1=[a, b, c]     B2 upload file: F2=[x, y]
B1 commit: C1=[a, b, c]          B2 commit: C2=[x, y]
B1 cache: [a, b, c]              B2 cache: [x, y]

      PG commits log orders between C1 and C2

Consumer C1 poll the partition with offset X for x

B1 look up commit cache: X? -> F2[0]
  If miss: PG look up X, populate
B1 look up cache: F2[0]? -> x
  If miss: S3 download F2[0]
```
---

## Deployment

--

### Kubernetes cluster

```
+---------------------+      +---------------------+
| KRaft pool          |   ___| Postgres            |
|   Persistent volume |  /   |   persistent volume |
+------+--------------+ /    +---------------------+
       |               /    
+------+--------------+        +--------------------+
| Broker pool         |        | Strimzi Operator   |
|   ephemeral storage +--------+   Entity           |
+------+--------------+        |   Cruise control   |
       |                       |   Metrics exporter |
       |                       +--------+-----------+
       |                                | 
+------+--------+            +----------+-----------+
| AWS S3 / FSxN |            | Prometheus / Grafana |
+---------------+            +----------------------+
```

--

## Infra (both AWS and FSxN modes)
```
| Resource             | Details                               |
|----------------------|---------------------------------------|
| EKS Cluster          | Managed K8s cluster                   |
| EKS Node Group       | EC2 worker nodes (e.g. 3 to 3 AZs)    |
| EBS CSI Driver       | Persistent volumes (PG option, Kraft) |
| Cluster Autoscaler   | scales node group                     |
```

--

## AWS mode only
```
|----------------------|--------------------|
| S3 Bucket            | For Kafka storage  |
```

## FSxN mode only

```
|----------------------|----------------------------------|
| FSxN File System     | ONTAP file system                |
| FSxN SVM             | Storage Virtual Machine          |
| Security Groups      | EKS to FSxN rules                |
| Trident              | Only with PG on FSxN iSCSI mode  |
```
--

## Example for 3 AZ
- 3 nodes on 3 AZs (32 vCPU, 128GB RAM, 80GB EBS, no local NVMe)
- Strimzi manages Inkless, broker pool with ephemeral storage, Kraft pool on EBS
- Bitnami manages Postgres, 1 primary 2 replica with EBS or standalone with FSxN via iSCSI
- AWS S3: in-built multi-AZ durability
- For FSxN: 2 AZ with SnapMirror to 3rd AZ

