# Building Data Pipelines

When people discuss building data pipelines using Apache Kafka, they are usuallly referring to a couple of use cases. 

1. The first is building a data pipeline where Apache Kafka is one of the two end points. For example, getting data from Kafka to S3 or getting data from MongoDB into Kafka. 
2. The second use case involves building a pipeline between two different systems but using Kafka as an intermediary. An example of this is getting data from Twitter to Elasticsearch by sending the data first from Twitter to Kafka and then from Kafka to Elasticsearch.

The main value Kafka provides to data pipelines is its ability to serve as a very large, reliable buffer between various stages in the pipeline, effectively decoupling producers and consumers of data within the pipeline.

## Considerations When Building Data Pipelines

While we can’t get into all the details on building data pipelines here, we would like to highlight some of the most important things to take into account when designing software architectures with the intent of integrating multiple systems.

### Timeliness

Most data pipelines fit somewhere in between these two extremes:

* Expect their data to arrive in large bulks once a day;
* Expect the data to arrive a few milliseconds after it is generated.

### Reliability

First of all, it's necessary understood what means exactly-once delivery for kafka pipelines. What is, every event from the source system will reach the destination with no possibility for loss or duplication.

Since many of the end points are data stores that provide the right semantics for exactly-once delivery, a Kafka-based pipeline can often be implemented as exactly-once. Indeed, many of the available open source connectors support exactly-once delivery.

### High and Varying Throughput

The data pipelines we are building should be able to scale to very high throughputs as is often required in modern data systems. Even more importantly, they should be able to adapt if throughput suddenly increases.

Kafka is a high-throughput distributed system—capable of processing hundreds of megabytes per second on even modest clusters—so there is no concern that our pipeline will not scale as demand grows.

### Data Formats

The data types supported vary among different databases and other storage systems. You may be loading XMLs and relational data into Kafka, using Avro within Kafka, and then need to convert data to JSON when writing it to Elasticsearch, to Parquet when writing to HDFS, and to CSV when writing to S3.

* Kafka itself and the Connect APIs are completely agnostic when it comes to data formats.

### Transformations

ELT stands for Extract-Load-Transform and means the data pipeline does only minimal transformation (mostly around data type conversion), with the goal of making sure the data that arrives at the target is as similar as possible to the source data. These are also called high-fidelity pipelines or data-lake architecture.

### Security

Security is always a concern. In terms of data pipelines, the main security concerns are:

• Can we make sure the data going through the pipe is encrypted? This is mainly a concern for data pipelines that cross datacenter boundaries.
• Who is allowed to make modifications to the pipelines?
• If the data pipeline needs to read or write from access-controlled locations, can it authenticate properly?

### Coupling and Agility

One of the most important goals of data pipelines is to decouple the data sources and data targets. There are multiple ways accidental coupling can happen:

#### Ad-hoc pipelines

Some companies end up building a custom pipeline for each pair of applications they want to connect. For example, they use Logstash to dump logs to Elasticsearch, and so on. It also means that every new system the company adopts will require building additional pipelines, increasing the cost of adopting new technology, and inhibiting innovation.

#### Loss of metadata

If the data pipeline doesn’t preserve schema metadata and does not allow for schema evolution, you end up tightly coupling the software producing the data at the source and the software that uses it at the destination. Without schema information, both software products need to include information on how to parse the data and interpret it.

#### Extreme processing

The more agile way is to preserve as much of the raw data as possible and allow downstream apps to make their own decisions regarding data processing and aggregation.

## When to Use Kafka Connect Versus Producer and Consumer

When writing to Kafka or reading from Kafka, you have the choice between using traditional producer and consumer clients, as described in Chapters 3 and 4, or using the Connect APIs and the connectors as we’ll describe below. Before we start diving into the details of Connect, it makes sense to stop and ask yourself: “When do I use which?”

* Use Kafka clients when you can modify the code of the application that you want to connect an application to and when you want to either push data into Kafka or pull data from Kafka.

* You will use Connect to connect Kafka to datastores that you did not write and whose code you cannot or will not modify.

Connect is recommended because it provides out-of-the-box features like configuration management, offset storage, parallelization, error handling, support for different data types, and standard management REST APIs.

## Kafka Connect

Kafka Connect is a part of Apache Kafka and provides a scalable and reliable way to move data between Kafka and other datastores.

Kafka Connect runs as a cluster of worker processes. You install the connector plugins on the workers and then use a REST API to configure and manage connectors, which run with a specific configuration.

### Running Connect

Kafka Connect ships with Apache Kafka, so there is no need to install it separately. For production use, especially if you are planning to use Connect to move large amounts of data or run many connectors, you should run Connect on separate servers.

* bootstrap.servers:: A list of Kafka brokers that Connect will work with. Connectors will pipe their data either to or from those brokers. You don’t need to specify every broker in the cluster, but it’s recommended to specify at least three.

* group.id:: All workers with the same group ID are part of the same Connect cluster. A connector started on the cluster will run on any worker and so will its tasks.

* key.converter and value.converter :: Connect can handle multiple data formats stored in Kafka. The two configurations set the converter for the key and value part of the message that will be stored in Kafka. The default is JSON format using the JSONConverter included in Apache Kafka. 

Once the workers are up and you have a cluster, make sure it is up and running by checking the REST API:

`curl http://localhost:8083/`

We can also check which connector plugins are available:

`curl http://localhost:8083/connector-plugins`

### Connector Example: File Source and File Sink

To start, let’s run a distributed Connect worker.

`bin/connect-distributed.sh config/connect-distributed.properties &`

Now it’s time to start a file source. As an example, we will configure it to read the Kafka configuration file—basically piping Kafka’s configuration into a Kafka topic:

`echo '{"name":"load-kafka-config", "config":{"connector.class":"FileStreamSource","file":"config/server.properties","topic":"kafka-config-topic"}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"`

To create a connector, we wrote a JSON that includes a connector name, load-kafka-config , and a connector configuration map, which includes the connector class, the file we want to load, and the topic we want to load the file into.

### Connector Example: MySQL to Elasticsearch

Now that we have a simple example working, let’s do something more useful. Let’s take a MySQL table, stream it to a Kafka topic and from there load it to Elasticsearch and index its contents.

The next step is to make sure you have the connectors. If you are running Confluent OpenSource, you should have the connectors already installed as part of the platform. Otherwise, you can just build the connectors from GitHub:

1. Go to the Elasticsearch connector https://github.com/confluentinc/kafka-connect-elasticsearch
2. Clone the repository
3. Run mvn install to build the project
4. Repeat with the JDBC connector https://github.com/confluentinc/kafka-connect-jdbc

Now take the jars that were created under the target directory where you built each connector and copy them into Kafka Connect’s class path:

```
mkdir libs
cp ../kafka-connect-jdbc/target/kafka-connect-jdbc-3.1.0-SNAPSHOT.jar libs/
cp ../kafka-connect-elasticsearch/target/kafka-connect-elasticsearch-3.2.0-SNAPSHOT-package/share/java/kafka-connect-elasticsearch/* libs/
```

1. The next step is to create a table in MySQL that we can stream into Kafka using our JDBC connector.
2. The next step is to configure our JDBC source connector. We can find out which configuration options are available by looking at the documentation.

With this information in mind, it’s time to create and configure our JDBC connector:

`echo '{"name":"mysql-login-connector", "config":{"connector.class":"JdbcSourceConnector","connection.url":"jdbc:mysql://127.0.0.1:3306/test?user=root","mode":"timestamp","table.whitelist":"login","validate.non.null":false,"timestamp.column.name":"login_time","topic.prefix":"mysql."}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"`

Let’s make sure it worked by reading data from the mysql.login topic:


`bin/kafka-console-consumer.sh --new --bootstrap-server=localhost:9092 -- topic mysql.login --from-beginning`


Note that while the connector is running, if you insert additional rows in the login table, you should immediately see them reflected in the mysql.login topic.

Now let’s start the elasticsearch connector:

`echo '{"name":"elastic-login-connector", "config":{"connector.class":"ElasticsearchSinkConnector","connection.url":"http://localhost:9200","type.name":"mysql-data","topics":"mysql.login","key.ignore":true}}' | curl -X POST -d @- http://localhost:8083/connectors --header "content-Type:application/json"`


If you add new records to the table in MySQL, they will automatically appear in the mysql.login topic in Kafka and in the corresponding Elasticsearch index.

### A Deeper Look at Connect

To understand how Connect works, you need to understand three basic concepts and how they interact. As we explained earlier and demonstrated with examples, to use Connect you need to run a cluster of workers and start/stop connectors.

#### Connectors and tasks

##### Connectors: The connector is responsible for three important things:

* Determining how many tasks will run for the connector
* Deciding how to split the data-copying work between the tasks
* Getting configurations for the tasks from the workers and passing it along

##### Tasks: Tasks are responsible for actually getting the data in and out of Kafka.

#### Workers

Kafka Connect’s worker processes are the “container” processes that execute the connectors and tasks. They are responsible for handling the HTTP requests that define connectors and their configuration, as well as for storing the connector configuration, starting the connectors and their tasks, and passing the appropriate configurations along.

Workers are also responsible for automatically committing offsets for both source and sink connectors and for handling retries when tasks throw errors.

#### Converters and Connect’s data model

The last piece of the Connect API puzzle is the connector data model and the converters. Kafka’s Connect APIs includes a data API, which includes both data objects and a schema that describes that data. For example, the JDBC source reads a column from a database and constructs a Connect Schema object based on the data types of the columns returned by the database.

#### Offset management

Offset management is one of the convenient services the workers perform for the connectors (in addition to deployment and configuration management via the REST API). The idea is that connectors need to know which data they have already processed, and they can use APIs provided by Kafka to maintain information on which events were already processed.


