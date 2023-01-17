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
 Monitor log files and send to KDS. Java-based agent built on KPL. Install only on Linux


This time Record consists of: Partition key(which shard it will go to), Data Blob(value) - up to 1 MB. 

Speed: Record can be sent by 1MB/sec or 1k msg/sec per shard.

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