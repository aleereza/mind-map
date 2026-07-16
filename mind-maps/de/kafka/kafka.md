# Apache Kafka {#apache-kafka}

> type: category

## Basics {#basics}

> type: category

### What Is Kafka? {#what-is-kafka}

> type: concept

Apache Kafka is a distributed event-streaming platform for publishing, storing, and processing streams of records. Producers append events to durable, partitioned topics, while consumers read those events independently, making Kafka useful for real-time data pipelines, event-driven services, and streaming analytics.

### How It Works {#how-kafka-works}

> type: concept

Kafka provides a durable, shared event log between systems, so producers and consumers do not need to be online at the same moment and multiple consumers can independently use or replay the same data. In an IoT setup, multiple sensors send measurements over local device protocols to an edge device. A gateway application running on that edge device collects, filters, normalizes, batches, or aggregates the readings, then uses a Kafka client to publish records to a topic such as `sensor-readings`. Kafka brokers append each record to a partition, retain it, and replicate it as configured. An alerting service, time-series database sink, or data-lake loader consumes the topic when ready. Kafka does not automatically choose a destination: that work belongs to a consumer application or Kafka Connect sink. The brokers run centrally as a self-managed or managed cluster, while the edge device and destination only need client or connector software.

#### Core Concepts {#core-concepts}

> type: category

##### Sensor {#sensor}

> type: concept

A measurement component that observes one physical property, such as temperature, pressure, or vibration. A sensor may be too simple to communicate with Kafka, and multiple sensors can send independent data streams to the same edge device.

##### Edge Device {#edge-device}

> type: concept

A local computing device near the sensors, such as an industrial PC, PLC, embedded controller, or gateway appliance. It receives streams from multiple sensors and provides compute, temporary buffering, local processing, and network connectivity for sending selected data upstream.

##### Gateway Application {#gateway-application}

> type: concept

Software running on the edge device that collects sensor streams, translates local device protocols, and can validate, normalize, filter, batch, or aggregate readings before publishing them. It is often the producer in an IoT setup.

##### Producer {#producer}

> type: concept

An application role that publishes records to Kafka topics. In the IoT example, the gateway is usually the producer and uses a Kafka client library to send sensor readings.

##### Kafka Client {#kafka-client}

> type: concept

A language library or compatible interface used by an application to authenticate with brokers and produce or consume records. It handles Kafka's network protocol and serializes outgoing keys and values or deserializes incoming ones.

##### Kafka Broker {#kafka-broker}

> type: concept

A Kafka server process that receives client requests and stores topic partitions in append-only log-segment files on its storage. Kafka does not require an external database; brokers replicate partition data with other brokers for fault tolerance.

##### Topic {#topic}

> type: concept

A named, durable stream of related records, such as `sensor-readings`. Producers write to the topic and consumers read from it independently.

###### Partition {#partition}

> type: concept

A partition is one ordered, append-only shard of a topic. Every record is written to one partition and receives a sequential offset there; multiple partitions distribute storage and work across brokers, but ordering is guaranteed only within a partition. When a key is present, the producer normally hashes its serialized bytes so the same key goes to the same partition.

###### Retention {#retention}

> type: concept

Retention controls how long or how much of a topic's log Kafka keeps, independently of whether consumers have read it. Time- or size-based retention removes old log segments, while compaction can retain the latest record for each key. Retention defines the replay window; it is not a backup.

##### Cluster {#cluster}

> type: concept

A group of Kafka brokers working together as one service. A cluster spreads topic partitions and their replicas across brokers for capacity and fault tolerance.

###### Replication {#replication}

> type: concept

Replication keeps copies of each partition on different brokers. With replication factor `3`, one leader serves the partition while two followers copy it and an in-sync follower can take over after failure. Replication improves availability and durability, but it multiplies storage and does not replace backup or extend retention.

##### Event {#event}

> type: concept

An event is the meaningful fact that happened, such as sensor `42` measuring `21.4°C`; a record is Kafka's stored representation of that event, and message is an informal term often used for either. A record carries a possibly null key, a value payload, a timestamp, and optional application-defined headers; its topic is selected when produced, and Kafka assigns its partition and offset. The producer serializes the key and value into bytes using JSON, Avro, Protocol Buffers, text, or another agreed encoding, and the consumer deserializes those bytes. The key is application data such as `sensor-42`, not inherently a hash, though the producer commonly hashes its serialized bytes to choose a partition.

##### Consumer {#consumer}

> type: concept

An application role that reads records from topic partitions and processes them. Reading does not delete records; a consumer tracks offsets so it can continue from its last position or intentionally replay earlier data.

##### Sink {#sink}

> type: concept

A consumer application or Kafka Connect sink connector that reads a Kafka topic and writes records to an external destination, such as a time-series database, data lake, or alerting system.

##### Managed Kafka Service {#managed-kafka-service}

> type: concept

A hosted offering that exposes Kafka-compatible cluster endpoints while the provider operates the underlying service. Providers commonly handle compute, storage, networking, broker and controller provisioning, security, monitoring, failed-node replacement, patches, upgrades, scaling, partition rebalancing, and high availability. You still design topics, keys, partitions, retention, schemas, producers, consumers, and business processing; depending on the offering, the data plane may be open-source Apache Kafka or a compatible managed engine.

###### Aiven for Apache Kafka {#aiven-kafka}

> type: concept

A fully managed Apache Kafka service that runs in Aiven's cloud or a bring-your-own-cloud environment. Aiven operates the cluster and offers managed scaling, Kafka Connect, monitoring, and local, tiered, or object-storage-backed topic options.

###### Davra {#davra}

> type: concept

A full industrial IoT platform that bundles and manages Kafka alongside device integration, databases, analytics, digital twins, and application services. Kafka acts as Davra's internal enterprise message bus and IoT data stream, so Davra is broader than a standalone managed Kafka service.

###### Confluent Cloud {#confluent-cloud}

> type: concept

A fully managed data-streaming platform with a cloud-native Apache Kafka engine. Confluent operates cluster provisioning, scaling, reliability, and security and also provides connectors, schema governance, stream processing, and monitoring tools.

###### Amazon MSK {#amazon-msk}

> type: concept

AWS's managed Apache Kafka service. AWS provisions and operates the compute, storage, networking, brokers, maintenance, high availability, and failover while applications use standard Kafka clients.

###### Google Cloud Managed Kafka {#google-managed-kafka}

> type: concept

Google Cloud's managed service for open-source Apache Kafka. It automates broker provisioning, sizing, storage, partition rebalancing, infrastructure maintenance, security integration, monitoring, and logging.

## Runtime Model {#runtime-model}

> type: category

Kafka is not one infinite loop searching external systems for messages. It is a collection of long-running broker server processes and client applications communicating over persistent TCP connections: producers push records to brokers, consumers pull records from brokers, and brokers store the records between them.

### Producer Application {#producer-application}

> type: concept

In the IoT example, the gateway application typically runs a loop that waits for sensor readings. When data arrives, it builds a record and asks its Kafka client to serialize, batch, and push the record to a broker; Kafka itself does not inspect the sensors.

#### Bootstrap & Metadata {#producer-bootstrap-metadata}

> type: concept

The client starts with one or more bootstrap addresses such as `kafka-1.example.com:9092`. It resolves a hostname to an IP address, opens a TCP connection, and requests cluster metadata to learn which brokers exist and which broker leads each topic partition; the bootstrap broker is an entry point, not a permanent proxy.

#### Persistent Connections {#producer-persistent-connections}

> type: concept

The client maintains and reuses TCP connections to the brokers it needs instead of opening a new connection for every record. Connections may cross a private network, VPN, or the internet and commonly use TLS encryption plus authentication; the client handles reconnecting and retrying when connections fail.

#### Produce Request {#producer-produce-request}

> type: concept

Calling `producer.send(record)` makes the client serialize and batch records into Kafka's binary produce-request format, then send the bytes through a TCP connection to the partition's leader broker. TCP divides the request into IP packets for network transport, and the broker returns an acknowledgement according to the producer's configuration.

### Broker Process {#broker-process}

> type: concept

A broker is a long-running Kafka server process, commonly run as an operating-system service, container, or Kubernetes workload. At startup it loads its partition logs, joins the cluster, opens configured TCP listeners—commonly on port `9092`—and waits efficiently for client and broker requests. A managed Kafka provider runs these broker processes for you.

#### Request Handling {#broker-request-handling}

> type: concept

Network I/O and request-handler loops wait on sockets rather than repeatedly scanning for data. A produce request is validated and appended to a partition log; a fetch request asks for records after an offset and may be held briefly until data arrives, avoiding a wasteful busy loop.

#### Background Work {#broker-background-work}

> type: concept

Kafka continuously performs internal maintenance alongside client requests. Followers fetch partition updates from leaders, retention removes expired log segments, compaction and log maintenance manage storage, and cluster coordination tracks metadata and broker leadership.

### Consumer Application {#consumer-application}

> type: concept

A consumer application typically runs a poll loop that asks Kafka for records after its current partition offsets. Kafka may hold an empty fetch request open briefly, then returns new records or a timeout; the consumer deserializes and processes the results. Kafka does not push records to the destination, and reading them does not delete them.

## Data Flow {#data-flow}

> type: procedure

### Sensor Measures {#sensor-measures}

> type: step
> edge.next: edge-device-collects

One or more sensors measure properties in the physical world, such as sensor `42` recording a temperature of `21.4°C`, and emit their own streams of readings.

### Edge Device Collects {#edge-device-collects}

> type: step
> edge.next: gateway-builds-event

The edge device is a local computing device—such as an industrial PC, PLC, embedded controller, or gateway appliance—that receives streams from multiple sensors. It may filter, normalize, batch, aggregate, or temporarily buffer readings before they move upstream.

### Gateway Builds Event {#gateway-builds-event}

> type: step
> edge.next: client-creates-record

The gateway application is software running on the edge device. It converts selected sensor data into application events containing information such as the sensor ID, measured value, unit, and measurement time, and decides what should be published to Kafka.

### Client Creates Record {#client-creates-record}

> type: step
> edge.next: client-serializes-sends

The Kafka client is a library used by the gateway application to communicate with Kafka. Acting as a producer, the gateway uses it to create a record with the destination topic, key, value, timestamp, and any optional headers.

### Client Serializes & Sends {#client-serializes-sends}

> type: step
> edge.next: broker-appends-record

The client serializes the key and value into bytes, chooses a partition—normally using the key—and finds its leader through cluster metadata. It reuses a persistent TCP connection, or opens one when needed, and sends a binary Kafka produce request; DNS, IP routing, optional TLS and authentication, packet transport, retries, and acknowledgements are handled underneath.

### Broker Appends Record {#broker-appends-record}

> type: step
> edge.next: kafka-replicates-retains

The leader broker appends the record to the partition log on disk and assigns its sequential offset within that partition.

### Kafka Replicates & Retains {#kafka-replicates-retains}

> type: step
> edge.next: consumer-reads-record

Follower brokers copy the partition data according to its replication factor, and Kafka keeps the record according to the topic's retention policy.

### Consumer Reads Record {#consumer-reads-record}

> type: step
> edge.next: application-uses-event

A consumer fetches records from its current partition offset and uses its Kafka client to deserialize the key and value back into application data.

### Application Uses Event {#application-uses-event}

> type: step

The consumer can trigger an alert, perform a calculation, or use a sink to write the data to another system. Reading advances the consumer's position but does not delete the Kafka record; retention still controls when Kafka removes it.

## Architecture {#architecture}

> type: category

## Reliability & Operations {#reliability-and-operations}

> type: category

## Use Cases {#use-cases}

> type: category

## Ecosystem & Interfaces {#ecosystem-and-interfaces}

> type: category
