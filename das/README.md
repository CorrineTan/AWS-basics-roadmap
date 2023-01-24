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
2. KCL: checkpointing (resume progress), shard discovery, DynanoDB - checkpointing (one row per shard).<br>
ExpiredIteratorException - increase WCU
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

highly-secure, portable devices to collect and process data at the edge




# Domain 2: Storage

Main Services: S3+ Glacier, DynamoDB, ElasticCache

# Domain 3: Processing

Main Services: Lambda, ML, Glue, EMR, SageMaker, Data Pipeline

# Domain 4: Analysis

Main Services: Elasticsearch, Athena, Redshift

# Domain 5: Visualization

Main Services:QuickSight, KMS, CloudHSM

# Domain 6: Security

# Everything Else