> Preparing for the DAS exam and summary of the important knowledge I learn from Udemy Course https://www.udemy.com/course/aws-data-analytics/

# Domain 1: Collection

Main Services: Kinesis, IoT Core, Snowball, SQS, DMS, Direct Connect

### Three types of collection:

1. Real Time - Immediate actions

- KDS, SQS, IoT

2. Near-real time - Reactive actions

- KDF, DMS

3. Batch - Historical Analysis

- Snowball, Data Pipeline

### KDS - Kinesis Data Stream

Provision the shards

#### Producer Details:

Producers - produce Record to KDS. Can be an application, client, SDK/KPL(Kinesis Produce Library), Kinesis Agnet(only on Linux, send the log file to KDS) and 3rd party libraries (Spark, Kakfa, Flume).

 - API for SDK: PutRecord and PutRocord(s) <br>
 SDK use case: low throughput, higher latency, simple API, AWS Lambda.

 - Handling Kinessi API Exceptions <br>
 ProvisionedThroughputExceeded Exception: 1. retries with backoff   2. Increase shards(scaling)  3. Ensure the partition key is a good one.

 - KPL: Use case: high performance and long-running producers. It can handle the above exception (configurable retry mechanism). <br>
 2 APIs: Synchronous and Asynchronous API (better performance for async) <br>

 - Batching: increasing throughput and decrease cost, and higher packing efficiencies and better performance. - Collect (2 records)    - Aggregate (1 record, increase latency). Adjust the delay using RecordMaxBufferedTime (default 100ms).

 Note: If you don't want any delay in your apps, use AWS SDK (dont use KPL).

 - Kinesis Agent
 Monitor log files and send to KDS. Java-based agent built on KPL. Install only on Linux.
 CLI:
 ``` 
 sudo yum install -y aws-kinesis-agent
 ```


This time Record consists of: Partition key(which shard it will go to), Data Blob(value) - up to 1 MB. 

Speed: Record can be sent by 1MB/sec or 1k msg/sec per shard.

#### Consumer Details:

Consumers - KCL/SDK, Lambda, KDF(firehose) and KDA(analytics).

This time Record consists of: Partition key, Sequence no. (where the data was at the shard), Data Blob(value). 

Speed: - Shared: 2MB/sec per shard all consumers; - Enhanced: 2MB/sec per shard per consumer. 

2 characters: <br>
 - Immutability: once data is inserted in Kinesis, it can't be deleted. <br>
 - Ordering: data shares the same partition goes to same shard.

2 Modes: <br>
- Provisioned node           <br>  
- On-demand mode

Security of KDS: <br>
 - Control access/authorizaiton using IAM <br>
 - Encryption in flight - HTTPS endpoint <br>
 - Encryption at rest - KMS

Important for Consumers:
1. Kinesis SDK - GetRecords: if 5 consumers application comsume from the same shard, means every consumer can poll once a second and receive less than 400KB/s. 

If 10 consumers application concurrently from 1 shard, getrecords() has average latency of 2 seconds. Because - upto 5 gerecords API calls per second.  

2. KCL: checkpointing (resume progress), shard discovery, DynanoDB - checkpointing (one row per shard).<br>
ExpiredIteratorException - increase WCU

For example, you have 8mb/s data from KPL and 2mb/s data from KCL. That's because DDB is under provisioned, checkpointing does not happen fast enough and results in a lower throughput for your KCL based application. Make sure to increase the RCU / WCU.

3. Kinesis Connector Library (S3, DDB, Redshift, ElasticSearch)


#### Kinesis Data Streams CLI commands:
Commands:

```
aws kinesis put-record --stream-name <Demo> --partition-key <user1> --data "user signup" --cli-binary-format raw-in-base64-out
```

```
aws kinesis describe-stream --stream-name <Demo>
```

```
aws kinesis get-shard-iterator --stream-name <Demo> --shard-id <shardId-00000000000> --shard-iterator-type <TRIM_HORIZON>
```

```
aws kinesis get-records --shard-iterator <share_iterator_id>
```

#### Other KDS Knowledge:

1. Kinesis Enhanced Fan Out: each consumer get 2MB/s of provisioned throughput per shard (per consumer). because it pushes data to consumers over HTTP/2. (default limit of 20)

- no matter how many consumers you have, in enhanced fan out mode, each consumer will receive 2MB per second of throughput and have an average latency of 70ms. 

2. Scale Kinesis: <br>
- adding shards: shard splitting (increase stream capacity). Opposite is to merge shards. 
=> out-of-order records because of: resharding. Cannot reshard in parallel.
- Auto Scaling

3. Duplicates in Data Streams:<br>
- In Producers: due to network timeouts. Fix: enbed unique record ID in the data to de-duplicate on consumer side
- In Consumers: 1) worker terminates unexpectedly 2) worker instances are added or removed  3)shards are merged or split 4) application is deployed.  Fix: 1) make your consumer application idempotent 2) if the final destination can handle duplicates, it's recommend to do it there.

4. Security:<br>
- access/control: IAM
- encryption in flight: HTTPS request
- encrption at rest: KMS
- client side encryption
- VPC endpoints 

### KDF - Kinesis Data Firehose

KDF: store data into target destinations.

#### KDF producers and consumers
Producer: (as KDS): applications, client, SDK/KPL, Kinesis Agent. (KFS Specific) KDS, CloudWatch(logs, events), AWS IoT.

KFS "Batch writes" to a destionation Consumers (near real time 0 60 seconds latency.).

Consumers: S3, Redshift, ElasticSearch. (thirdparty) datadog, splunk, new relic, mongoDB. Customer destinations like HTTP Endpoint.

#### Why KDF
1. load data into Redshift, s3, splunk, elasticsearch. 
2. Good automatc scaling. 
3. Data transformation via AWS LAMBDA. 
4. You can collect source records, transformation failures, delivery failures to another S3 bucket. 

Important: SPARK and KCL only read from KDS, not KDF!!

Firehose Buffer Sizing: flushed based on time and size rules. High throughput: buffer size will be hit. Low throughput: buffer time will be hit. 

Mininum buffer time interval is 60 seconds.

#### KDF vs KDS

1. KDS: real time. KDF: near real time
2. KDS: Custom code (producer, consumer). KDF: fully managed, send to S3, Splunk, ElasticSearch, Redshift
3. KDS: must manage scalling (shard splitting/merging). KDF: Auto scaling
4. KDS: data storage for up to 365 days. KDF: no data storage
5. KDS: use Lambda to insert data in real-time to elasticsearch. KDF: serverless data transformations using Lambda

#### CloudWatch logs subscription filters - with kinesis
- near real time: CW - Subscription filter - KDF (use Lambda to do transformation) - Elastic Search 
- real time load into ES: CW - subscription filter - Lambda function (real time) - Elastic Search
- real time analystics: CW - subscription filter - KDS - KDA - Lambda

Need to go to aws-kinesis and chagne the agent.json, to add a flow with KDF or KDS. 

#### SQS (a little similar to kinesis)

- Producing messages: define body, add message attributes (metadata), provide delay delivery, get back (message identifier, md5 hash of the body)
- Consumeing messages: poll SQS for messages, process msg within visibility timeout, delete msg using msg ID and receipt handle
- FIFO queue: 5 min interval de-duplication using Duplication ID.
- SQS extended client: send large message exceed message size limit.

- SQS Use Case: 1) decouple applications  2) buffer writes to a database   3) handle large loads of messages comming in 


#### SQS vs Kinesis
- Kinesis: data streamin. Use case: fast log/event data collection and process; realtime metrics and reports; mobile data capture; realtime data analytics, gaming data; stream processingp; IOT data feed

- SQS: decoupling applications. Use case: order/image processing; request offloading; buffer/batch messages for future processing

### IOT ("Thing")
IOT Thing - Thing registry - Device gateway - (send message) - IoT message broker - IoT Rules Engine (target: Kinesis, SQS, Lambda) - Device Shadow

IoT device Gateway protocol - FTP

### DMS - database migration service

- Source: on-premise and EC2 (Oracle, MSSQL, MySQL, MariaDB, PostgresSQL, MongoDB, SAP, DB2); Azure (Azure SQL Database); Amazon RDS (all including Aurora); S3

- Targets: on-premise and EC2 (Oracle, MSSQL, MySQL, MariaDB, PostgresSQL, MongoDB, SAP, DB2); Amazon RDS (all including Aurora); S3; Redshift; DDB; ElasticSearch; KDS; DocumentDB

- AWS Schema Coversion Tool (SCT): converts your db's schema from one engine to another.

OLTP: SQL Server or Oracle to MySQL, PostgresSQL, Aurora
OLAP: Teradata or Oracle to Redshift

### Direct Connect (DX)

- provide a dedicated private connection from a remote network to your VPC.

- Dedicated conneciton: setup between your DC and AWS Direct Connect locations

- Virtual Private Gateway: setup on VPC

- Same connection you can access public resources (S3) and private (EC2)

- Use cases: 1) large datasets: increase bandwidth throughput, lower cost.   2) more consistent network experience - applications using real-time data feeds.  3) hybrid environment (on prem + cloud)

- Direct Connect Gateway: setup DX to one or more VPC in many different regions

### Snow Family

highly-secure, portable off-line physical devices to collect and process data at the edge and migrate data into and out of AWS.

edge locations: limited/no internet access

Data migration: snowcone, snowball edge, snowmobile.

Edge computing: snowcone, snowball edge.

### MSK

Fully managed - MSK creates and manage Kafka brokers nodes and Zookeeper nodes for you. Data stored on EBS volumes.

1 difference MSK vs Kinesis: MSK can custom configuration to send large messages (10MB), but kinesis hard limit of 1MB. 

#### MSK flow

Kinesis, IOT, RDS as Producers - (write topic to) - MSK Cluster (3 brokers) - (poll from topic) to Consumers (EMR, S3, SageMaker, Kinesis, RDS)

#### MSK security 

1. Encrpytion in MSK:
- Encryption in flight: TLS between brokers
- Encryption at rest: KMS

2. Network security: security groups

3. Authentication: who can read/write to kafka topics:
- MutualTLS(AuthN) + Kafka ACLs(AuthZ)
- SASL/SCRAM(AuthN) + Kafka ACLs(AuthZ)
- IAM (AuthN + AuthZ)

#### MSK Monitoring

- CloudWatch metrics
- Broker Log Delivery 
- Prometheus

#### MSK Others

- MSK Connect
- MSK Serverless (auto provision resources and scales compute & storage)

#### Kinesis vs MSK 

1. KDS: 1MB hard limit size.   MSK: configurable for higher

2. KDS: Data Stream with Shards.    MSK: Kafka Topics with Partitions

3. KDS: Shard spliting and merging. MSK: can only add partitions to a topic (cannot remove)

4. KDS: TLS inflight encryption.    MSK: PLAINTEXT or TLS inflight encryption.


# Domain 2: Storage

Main Services: S3+ Glacier, DynamoDB, ElasticCache

### S3

#### S3 Use Cases
- Backup and Storage
- Disaster Recovery
- Archive
- Hybrid Cloud storage
- Application hosting
- Media hosting
- Data lakes & big data analytics
- Software delivery
- Static website

#### S3 Basics

- S3 objects(files) have a key - full path (composed of a prefix + object name)

- S3 max object size: 5TB. If upload more than 5GB, must use "multi-part upload". 

- versioning - bucket level, delete marker

- Replication: must enable versioning => only new objects are replicated. (the exisitng ones need to be s3 batch replication)
1. CRR (cross-region): compliance, lower latency access, replication across accouts
2. SRR(same-region): log aggregation, live replication between production and test accoutns
3. No chain of replication.

#### S3 Storage types

- S3 standard
- S3 Infrequent Access (IA): backup copies of on-premise data, or data you can recreate. - 30 days
- S3 one zone-infrequent access. Use case: s3 thumbnails with lifecycle to expire them after 60 days.
- s3 glacier instant: milliseconds retrieval - 90 days
- s3 glacier flexible: expedited(1-5min), standard(3-5hours), bulk(5-12hours)
- s3 glacier deep archieve: standard(12hours), bulk(48hours) - 180days to 700days 
- s3 intelligent tiering

#### S3 Event Notifications

use case: generate thumbnails of images uploaded to s3

- Advanced filtering
- multiple destinations
- eventbridge capabilities: archive, replay events, reliable delivery.

Ue Lifecycle rules to transition actions, expiration actions, 

#### S3 Performance

- Multi-parts upload: must use for file > 5GB (recommended for file > 100MB)
- Transfer acceleration: use a edge location
- S3 Byte-range fetches: speed up downloads

#### S3 Select 

retrieve less data using SQL by performing server-side filtering

#### S3 Encryption

- SSE(server-side)-S3: must set header "x-amz-server-side-encryption": "AES256"
- SSE-KMS: must set header "x-amz-server-side-encryption": "aws:kms".  Limit: GenerateDataKey KMS API service quota. 
- SSE-C(customer-provided)
- CSE(client-side)

#### S3 Access Points & Object Lambda

 Access Points: one policy per access point (per appartment): easier to manage than complex bucket policies

 Object Lambda: change object before it is retrived by the caller application

### DynamoDB

NOSQL serverless database: distributed, doesn't support query joins, aggregation (sum, avg). But can scale horizontally ().

Each DDB tables has a primary key. Infinite number of items(rows, max_size = 400KB). Each item has attributes (can be added over time - can be null - columns).

#### DDB Primary Key

- Partition Key (HASH): must be unique and diverse

- Partition Key + Sort Key (HASH + RANGE)

#### DDB RCU and WCU

read/write capacity modes

- Default: Provisioned mode

- On-demand mode (more expensive)

- Throttling reasons: (only happen with provision, not on-demand)
ProvisionedTrhoughputExceededException - burst capacity has been consumed. Retry: exponential backoff.

1. Hot Keys: one partition key is being read too many times (popular item)
2. hot partitions
3. very large items

solution:
1. exponential backoff
2. distribute partition keys
3. if RCU issue, use DAX (ddb accelerator)


- 1 WCU =  1 write per sec for an item up to 1KB.

- RCU: strongly consistent read vs eventually consistent read

So 1 RCU = 1 Strongly consistent read per sec, or 2 eventually consistent reads per sec, for an item up to 4KB.

WCUs and RCUs are spread evenly across parititions.

Partitions internal: copies of your data that live on specific servers. 

#### DDB APIs

1. Writing Data: PutItem, UpdateItem, Conditional Writes
2. Reading Data: GetItem, Query (KeyConditionExpression, FilterExpression), Scan (read an entire table)
3. Deleting Data: DeleteItem, DeleteTable
4. Batch Operations: BatchWriteItem (25 putitem or deleteitm, but cannot update). BatchGetItem.

#### DDB Index

1. LSI: Local secondary index 

It's an alternative sort key. (same partition key as of base table)

2. GSI: global second index

It's an alternative primary key. (must Provision WCU and RCUs)

Important: if writes are thrttled on GSI, then the main table will be throttled.

#### DDB PartiQL

sql-like query

#### DDB DAX

It solves the hot key problem (too many reads).

5 min TTL for cache (default)

#### DDB Streams

Ordered stream of item level modifications (create, update, delete).
Destinations: KDS, Lambda, KCL.

It also has shards.

#### DDB TTL

auto delete items after an expiry timstamp - unix epoch timestamp - called "expire_on" (TTL settings you can set)

#### DDB others:

1. DDB and S3: 1) Store large objects pattern: use a S3 bucket to store the object URL, then URL put into DDB tables 2) DDB index S3 objects Metadata

2. DDB Security: 1) access DDB without internet: VPC endpoints   2) encryption in transit: SSL/TLS

### ElasticCache

In-memory database: high performance, low latency. fully managed Redis(key-value, more popular) or Memcached(object store). 

Main purpose: reduce the load off of database for read intensive workloads.

# Domain 3: Processing

Main Services: Lambda, ML, Glue, EMR, SageMaker, Data Pipeline

### AWS Lambda - serverless data processing

a way to run code snippets in the cloud, continuous scaling.

Use case: Real-time file/stream processin, ETL, cron replacement, process AWS Events.

Limits: 1000 concurrent executions per region. 15min timeout.

Lambda is Stateless. You need to use S3 or DDB to keep track of the state. 

### AWS Glue - Table definitions and ETL

#### Glue Basics

Serverless descrovery and definition of table definitions and schema (in S3 data lakes, RDS, Redshift, EMR, Athena), and custom ETL jobs(trigger driven, on a schedule or on demand)

Glue crawler: scans data in S3, creates schema for unstracture data to be used in Athena, EMR, Reshift, then use QuickSight to visualize. 

Pay attention to S3 partition. Need to put the thing you want to query against at top partition like device/yy/mm/dd

Glue and Hive: Hive running on EMR run SQL-like queries from EMR. Glue data catalog can serve as Hive metastore.

Glue ETL: Spark under the hood (scala or python). DPU (data processing units) to Spark job performance. 

Glue ETL has Glue scheduler, or Glue triggers.

Glue DynamicFrame is a collection of DynamicRecords(self-dscribing, have a schema)

When modifying the DataCatalog - either rerun the crawler, or have script use "enableUpdateCatalog" and "partitionKeys" options

#### GLUE ETL - Transformations

- Bundled transformation: DropFields, DropNullFields, Filter, Join, Map.

- ML transformation: FindMatches ML

- Format conversions: CSV, JSON, Avro, Parquet, ORC, XML

- All Spark transformations (K-Means)

- ResolveChoice: deals with ambiguities in DynamicFrame and returns a new one  

#### Running Glue jobs:

- cron style time-based schedules

- job bookmarks: persist state from the job run ( so don't end up with reprocessing the same data) (important: only handle new rows, not updated rows)

- CloudWatch events: when ETL succeeds or fails, fire off a labmda function.  

#### Glue others

1. Glue studio: visualize DAG
2. Glue DataBrew: receipe (really like Dataiku!!), pre-process data
3. Glue Elastic Views: build materialized views, SQL interface

### Lake Formation 

It's on top of Glue. Loading data and monitoring data flows. Help build a data lake.

Workflow of build data lake using lake formation: 

1. glue connection on your data source 2. create a s3 bucket 3. register s3 to lake formation, grant permissions 4. create database in lake formation for data catalog, grant permissions  5. use a blueprint for a workflow  6. run the workflow  7. grant select permission to whoever needs to read it.

Cross account access to lake formation: RAM - resouce access manager.

Lake Formation can implement column level security on your data lake.

### EMR

- Master, Core (store HDFS data on HDFS or EMRFS), Task node(can be spot to save money)
- Long-running cluster vs transient cluster

#### EMR Storage

1. HDFS: hadoop distributed file system

- multiple copies store across cluster instances for redundancy.
- file store as blocks (128 MB default size)
- But! ephemral - lost if cluster is terminated.

2. EMRFS

- access S3 as if it were HDFS
- emrfs consistent view: no longer a pain anymore! because of S3 strong consistency.
It used to, as multiple nodes might write to single s3 file at a time.

3. Local file system  - temp data (buffers, caches)

4. EBS for EBS

#### EMR Hadoop and services

HDFS -> YARN (resource negotiator) -> MapReducer (mapper: map data to key/value pairs, reducers: reduce intermediate results to final output)

-  Scaling: auto vs managed

scale-up: first add core nodes, then add task nodes, up to max units

scale-down: first remove task nodes, then core nodes, no further than minimum. Always start with spot

- Pig writing mapper/reducer not using Java but use Pig Latin(SQL like syntax)

- Presto: interactive queires at petabyte scale  (but stil for OLAP not OLTP) , in memory

- Zeppelin: like spark shell - can run spark code interactively. speeds up the development cycle. Makes spark feel more like a data science tool.

- EMR notebooks: backed up to s3, hosted inside VPC, accessed only via aws console.

- Hue: hadoop user experience

- Splunk: makes machine data accessible, usable, valuable to everyone. Operational tool. 

- Flume: streams data into cluster (orginally made to handle log aggregation)

- MXNet: framework of Deep Learning (like Tensorflow)

- S3DistCP: copy large amounts of data from s3 into hdfs, or from hdfs into s3.

- Ganglia (monitor), Mahout (ML), Accumulo (NoSQL db), Sqoop(relational db connector), HCatalog(table and storage management for hive metastore), Kinesis Connector (access KDS in your scripts), Ranger (data security manager for Hadoop)

- Kerberos: secure user authentication


#### Data Pipeline 

Like Hadoop Oozie - flow of jobs. 

Destination: S3, RDS, DDB, Redshift, EMR

Activities: EMR, HIVE, COPY, SQL, scripts

# Domain 4: Analysis

Main Services: Elasticsearch, Athena, Redshift

# Domain 5: Visualization

Main Services:QuickSight, KMS, CloudHSM

# Domain 6: Security

# Everything Else