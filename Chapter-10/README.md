# Monitoring Kafka

## Metrics basics

### Where are the metrics?
All Metrics can be accessed via JMX interface

### Internal x External Measurements
#### Internal
1. Provided by the application being measured
2. More detailed


#### External
1. Third-party app or a Kafka client provide the metrics
2. More informative metrics, e.g, broker availability, latency

### Application Health Check
Two approaches can be used
1. External process reporting if the broker is up or down
2. Alert the lack of metrics being reported by the broker

### Metric Coverage
Choose which alarm and metrics are relevant for your system, to avoid alarm fatigue
Prefer fewer metrics with high-level coverage, this would avoid noise, but requires additional information to indicate the exact nature of the problem

## Kafka Broker Metrics
### Who watches the watchers?
Make sure that the monitoring and alerting for Kafka does not depend on Kafka working

## Under-Replicated Partitions
As we saw in the chapter 6, Kafka has some constraints regarding data delivery.
For that reason, watching under replicated partitions is an excellent choice of metrics to watch in Kafka
Also, this metric provides a number of insights of problems with Kafka and, due to the wide variety of problems that this metric can indicate, it is worthy of a deep look when this indicator value is other than zero.

Metric name | Under-replicated partitions
------------| ---------------------------
JMX MBean   | `kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions`
Value range | Integer, zero or greater

### Some examples
1. If the number of under replicated partitions is greater than zero across the entire cluster and haven't changed for a while, it means that a broker is probably down (hardware, OS or Java failure)
2. If the number of under replicated partitions is fluctuating, or if the number are steady but there's an offline broker it can be performance related. If the problem is at only one broker it shows that the other brokers are having problems replicating messages to this broker. If there are more brokers with problem, it could be a custer problem or still only one broker with problem. One way of doing this is to get the list of under-replicated partitions and check if there's an specific broker common to these partitions.

You can use `kafka-topics.sh` to check this information

```sh
kafka-topics.sh --zookeeper zoo1.example.com:2181/kafka-cluster --describe
--under-replicated


Topic: topicOne  Partition: 5  Leader: 1  Replicas: 1,2 Isr: 1
Topic: topicOne  Partition: 6  Leader: 3  Replicas: 2,3 Isr: 3
Topic: topicTwo  Partition: 3  Leader: 4  Replicas: 2,4 Isr: 4
Topic: topicTwo  Partition: 7  Leader: 5  Replicas: 5,2 Isr: 5
Topic: topicSix  Partition: 1  Leader: 3  Replicas: 2,3 Isr: 3
Topic: topicSix  Partition: 2  Leader: 1  Replicas: 1,2 Isr: 1
Topic: topicSix  Partition: 5  Leader: 6  Replicas: 2,6 Isr: 6
Topic: topicSix  Partition: 7  Leader: 7  Replicas: 7,2 Isr: 7
Topic: topicNine Partition: 1  Leader: 1  Replicas: 1,2 Isr: 1
Topic: topicNine Partition: 3  Leader: 3  Replicas: 2,3 Isr: 3
Topic: topicNine Partition: 4  Leader: 3  Replicas: 3,2 Isr: 3
Topic: topicNine Partition: 7  Leader: 3  Replicas: 2,3 Isr: 3
Topic: topicNine Partition: 0  Leader: 3  Replicas: 2,3 Isr: 3
Topic: topicNine Partition: 5  Leader: 6  Replicas: 6,2 Isr: 6
```

### Cluster-level problems

Cluster problems usually fall into one of two categories:
* Unbalanced load
* Resource exhaustion

#### Unbalanced load
To diagnose this problem you will need several metrics from the brokers in the cluster:
* Partition count
* Leader partition count
* All topics byte rate
* All topics message rate

For perfectly balanced cluster the numbers will be very alike, like the example bellow:

Broker | Partitions | Leaders | Bytes in | Bytes out
:---:  | :---:      |   :---: | :---:    | :---:
1      | 100        | 50      | 3.56 MB/s| 9.45 MB/s
2      | 101        | 49      | 3.66 MB/s| 9.25 MB/s
3      | 100        | 50      | 3.23 MB/s| 9.82 MB/s

Kafka does not provide any tool to automate the automatic reassignment of partitions in a cluster, which means that balancing traffic can be troublesome. Nonetheless, some organizations developed tools to help with this task, like the `kafka-assigner` tool developed by LinkedIn and released in the Open Source project [kafka-tools][kafka-tools-url]

#### Resource exhaustion
Bottlenecks:
* CPU
* Disk I/O
* Network throughput

OS level metrics
* CPU utilization
* Inbound network throughput
* Outbound network throughput
* Disk average wait time
* Disk percent utilization

Common problems caused by this
* Under-replicated partitions

Under-replicated partitions lead to customers having problems to produce and consume message, thus, it makes sense to develop a baseline for these metrics when your cluster is operating correctly and set thresholds that indicate a problem.

### Host-level problems
If the performance problem with Kafka is not present in the entire cluster and can be isolated to one or two brokers, it’s time to examine that server and see what makes it different from the rest of the cluster.
These problems fall into several problems, separated in those general categories:
* Hardware failures
* Conflicts with another process
* Local configuration differences

#### Hardware failures
* Linux debugging tools, e.g., dmesg; can help on finding the problem
* Most common: disk failure (remember that Kafka doesn't wait for disk flush)
* A single disk failure on a single broker can destroy the performance in the entire cluster
* Hardware or configuration network related problems can also happen

#### Conflicts with another process
* Wrongly installed applications
* Monitoring agent running with problems

#### Local configuration differences
* System or broker configuration
* Configuration management tool (Chef, Puppet, Ansible) is crucial to maintain consistent configurations

## Broker Metrics
### Active controller count
Indicates whether the broker is the current controller of the cluster. The value can be `0` or `1`
Having more than one broker with the controller metric equals to `1` indicates a problem and the administrative tools can't be used properly.
Having no broker as the controller causes the cluster to fail when there are state changes

Metric name | Active controller count
:---        | :---      
JMX MBean   | `kafka.controller:type=KafkaController,name=ActiveControllerCount`
Value range | Zero or one

### Request handler idle ratio
Kafka uses two thread pools for handling all client requests. The request handler requests are the responsible for handling the most of the client data, as reading and writing messages to the disk.

Metric name | Request handler average idle percentage
:---        | :---      
JMX MBean   | `kafka.server:type=KafkaRequestHandlerPool name=RequestHandlerAvgIdlePercent`
Value range | Float, between zero and one inclusive

The request handler idle ratio metric indicates the percentage of time the request handlers are not in use. The lower this number, the more loaded the broker is. Experience tells us that idle ratios lower than 20% indicate a potential problem, and lower than 10% is usually an active performance problem.

#### Common problem originator
1. High thread utilization. In general, the number of threads should be equal to the number of CPUs
2. Unnecessary thread work

### All topics bytes in
Can be used as an indicator of how much message traffic your brokers are receiving from producer clients. Can also be used as a traffic balancer between brokers.

Metric name | Bytes in per second
:---        | :---      
JMX MBean   | `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec`
Value range | Rates as doubles, count as integer

These two descriptive attributes tell us that the rates, regardless of the period of time they average over, are presented as a value of bytes per second. There are four rate attributes provided with different granularities:

1. `OneMinuteRate`: An average over the previous 1 minute.
2. `FiveMinuteRate`: An average over the previous 5 minutes.
3. `FifteenMinuteRate`: An average over the previous 15 minutes.
4. `MeanRate`: An average since the broker was started.

### All topics bytes out
The all topics bytes out rate, similar to the bytes in rate, is another overall growth metric. In this case, the bytes out rate shows the rate at which consumers are reading messages out. The outbound bytes rate may scale differently than the inbound bytes rate, thanks to Kafka’s capacity to handle multiple consumers with ease.

Metric name | Bytes out per second
:---        | :---      
JMX MBean   | `kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec`
Value range | Rates as doubles, count as integer

### All topics message in
While the bytes rates described previously show the broker traffic in absolute terms of bytes, the messages in rate shows the number of individual messages, regardless of their size, produced per second. This is useful as a growth metric as a different measure of producer traffic. It can also be used in conjunction with the bytes in rate to determine an average message size. You may also see an imbalance in the brokers, just like with the bytes in rate, that will alert you to maintenance work that is needed.

Metric name | Messages in per second
:---        | :---      
JMX MBean   | `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec`
Value range | Rates as doubles, count as integer

### Partition count
The partition count for a broker generally doesn’t change that much, as it is the total number of partitions assigned to that broker

Metric name | Partition count
:---        | :---      
JMX MBean   | `kafka.server:type=ReplicaManager,name=PartitionCount`
Value range | Integer, zero or greater

### Leader count
The leader count metric shows the number of partitions that the broker is currently the leader for.

Metric name | Leader count
:---        | :---      
JMX MBean   | `kafka.server:type=ReplicaManager,name=LeaderCount`
Value range | Integer, zero or greater

### Offline partitions
Along with the under-replicated partitions count, the offline partitions count is a critical metric for monitoring. This measurement is only provided by
the broker that is the controller for the cluster (all other brokers will report 0), and shows the number of partitions in the cluster that currently have no leader. Partitions without leaders can happen for two main reasons:
1. All brokers hosting replicas for this partition are down
2. No in-sync replica can take leadership due to message-count mismatches (with
unclean leader election disabled)

Metric name | Offline partitions
:---        | :---      
JMX MBean   | `kafka.controller:type=KafkaController,name=OfflinePartitionsCount`
Value range | Integer, zero or greater

### Request metrics
The Kafka protocol, described in Chapter 5, has many different requests. Metrics are provided for how each of those requests performs. The following requests have metrics provided:
* `ApiVersions`
* `ControlledShutdown`
* `CreateTopics`
* `DeleteTopics`
* `DescribeGroups`
* `Fetch`
* `FetchConsumer`
* `FetchFollower`
* `GroupCoordinator`
* `Heartbeat`
* `JoinGroup`
* `LeaderAndIsr`
* `LeaveGroup`
* `ListGroups`
* `Metadata`
* `OffsetCommit`
* `OffsetFetch`
* `Offsets`
* `Produce`
* `SaslHandshake`
* `SyncGroup`
* `UpdateMetadata`
  
For each of these requests, there are eight metrics provided, providing insight into each of the phases of the request processing.

\#   |Name               | JMX MBean
:--- |:---               | :---      
1    |Total time         | `kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Fetch`
2    |Request queue time | `kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request=Fetch`
3    |Local time         | `kafka.network:type=RequestMetrics,name=LocalTimeMs,request=Fetch`
4    |Remote time        | `kafka.network:type=RequestMetrics,name=RemoteTimeMs,request=Fetch`
5    |Throttle time      | `kafka.network:type=RequestMetrics,name=ThrottleTimeMs,request=Fetch`
6    |Response queue time| `kafka.network:type=RequestMetrics,name=ResponseQueueTimeMs,request=Fetch`
7    |Response send time | `kafka.network:type=RequestMetrics,name=ResponseSendTimeMs,request=Fetch`
8    |Requests per second| `kafka.network:type=RequestMetrics,name=RequestsPerSec,request=Fetch`

The seven time metrics each provide a set of percentiles for requests, as well as a discrete Count attribute, similar to rate metrics. The metrics are all calculated since the broker was started, so keep that in mind when looking at metrics that do not change for long periods of time, the longer your broker has been running, the more stable
the numbers will be. The parts of request processing they represent are:

#### Total time
Measures the total amount of time the broker spends processing the request,
from receiving it to sending the response back to the requestor.
#### Request queue time
The amount of time the request spends in queue after it has been received but before processing starts.

#### Local time
The amount of time the partition leader spends processing a request, including sending it to disk (but not necessarily flushing it).

#### Remote time
The amount of time spent waiting for the followers before request processing can complete.

#### Throttle time
The amount of time the response must be held in order to slow the requestor down to satisfy client quota settings.

#### Response queue time
The amount of time the response to the request spends in the queue before it can be sent to the requestor.

#### Response send time
The amount of time spent actually sending the response.


The attributes provided for each metric are:
#### Percentiles
50thPercentile , 75thPercentile , 95thPercentile , 98thPercentile , 99thPercentile , 999thPercentile

#### Count
Absolute count of number of requests since process start

#### Min
Minimum value for all requests

#### Max
Maximum value for all requests

#### Mean
Average value for all requests

#### StdDev
The standard deviation of the request timing measurements as a whole

Out of all of these metrics and attributes for requests, which are the important ones to monitor? At a minimum, you should collect at least the average and one of the higher
percentiles (either 99% or 99.9%) for the total time metric, as well as the requests per second metric, for every request type.

## Topic and Partition Metrics
In addition to the many metrics available on the broker that describe the operation of the Kafka broker in general, there are topic- and partition-specific metrics.
* Useful for debugging specific issues with a client
* Important to be accessible by Kafka users (the producer and consumer clients)

### Per-topic metrics
For all the per-topic metrics, the measurements are very similar to the broker metrics described previously. In fact, the only difference is the provided topic name, and that the metrics will be specific to the named topic.

\#    |Name                 | JMX MBean
:--- |:---                  | :---      
1    | Bytes in rate        | `kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec,topic=TOPICNAME`
2    | Bytes out rate       | `kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec,topic=TOPICNAME`
3    | Failed fetch rate    | `kafka.server:type=BrokerTopicMetrics,name=FailedFetchRequestsPerSec,topic=TOPICNAME`
4    | Failed produce rate  | `kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec,topic=TOPICNAME`
5    | Messages in rate     | `kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec,topic=TOPICNAME`
6    | Fetch request rate   | `kafka.server:type=BrokerTopicMetrics,name=TotalFetchRequestsPerSec,topic=TOPICNAME`
7    | Produce request rate | `kafka.server:type=BrokerTopicMetrics,name=TotalProduceRequestsPerSec,topic=TOPICNAME`

### Per-partition metrics
* Tend to be less useful on an ongoing basis than the per-topic metrics
* Can be useful in some limited situations
* Data indicate the amount of data retained for a single topic, which can be useful in allocating costs for Kafka to individual clients
 
\#   |Name                | JMX MBean
:--- |:---                | :---      
1    | Partition size     | `kafka.log:type=Log,name=Size,topic=TOPICNAME,partition=0`
2    | Log segment count  | `kafka.log:type=Log,name=NumLogSegments,topic=TOPICNAME,partition=0`
3    | Log end offset     | `kafka.log:type=Log,name=LogEndOffset,topic=TOPICNAME,partition=0`
4    | Log start offset   | `kafka.log:type=Log,name=LogStartOffset,topic=TOPICNAME,partition=0`

The log end offset and log start offset metrics are the highest and lowest offsets for messages in that partition, respectively. It should be noted, however, that the difference between these two numbers does not necessarily indicate the number of messages in the partition, as log compaction can result in “missing” offsets that have been
removed from the partition due to newer messages with the same key

## JVM Monitoring
### Garbage collection
* The critical thing to monitor is the status of garbage collection (GC)
* This information will vary depending on the particular Java Runtime Environment (JRE) that you are using
* As well as the specific GC settings in use
  

\#    |Name            | JMX MBean
:--- |:---             | :---      
1    | Full GC cycles  | `java.lang:type=GarbageCollector,name=G1 Old Generation`
2    | Young GC cycles | `java.lang:type=GarbageCollector,name=G1 Young Generation`

Each of these metrics also has a `LastGcInfo` attribute. This is a composite value, made up of five fields, that gives you information on the last GC cycle for the type of
GC described by the bean. The important value to look at is the duration value, as this tells you how long, in milliseconds, the last GC cycle took.

### Java OS monitoring
The JVM can provide you with some information on the OS through the `java.lang:type=OperatingSystem` bean. The two attributes that can be collected here that are of use, which are difficult to collect in the OS, are the `MaxFileDescriptorCount` and `OpenFileDescriptorCount` attributes. `MaxFileDescriptorCount` will tell you the maximum number of file
descriptors (FDs) that the JVM is allowed to have open.

## OS Monitoring
The JVM cannot provide us with all the information that we need to know about the system it is running on. For this reason, we must not only collect metrics from the
broker but also from the OS itself. Most monitoring systems will provide agents that will collect more OS information than you could possibly be interested in. The main areas that are necessary to watch are CPU usage, memory usage, disk usage, disk IO, and network usage.

### CPU utilization
You will want to look at the system load average at the very least. This provides a single number that will indicate the relative utilization of the processors. In addition, it may also be useful to capture the percent usage of the CPU broken down by type. Depending on the method of collection and your particular OS, you may have some or all of the following CPU percentage breakdowns (provided with the abbreviation used):

*us*<br/>
The time spent in user space.

*sy*<br/>
The time spent in kernel space.

*ni*<br/>
The time spent on low-priority processes.

*id*<br/>
The time spent idle.

*wa*<br/>
The time spent in wait (on disk).

*hi*<br/>
The time spent handling hardware interrupts.

*si*<br/>
The time spent handling software interrupts.

*st*<br/>
The time waiting for the hypervisor.

### Memory utilization
Memory is less important to track for the broker itself, as Kafka will normally be run with a relatively small JVM heap size. It will use a small amount of memory
outside of the heap for compression functions, but most of the system memory will be left to be used for cache.

### Disk utilization
Disk is by far the most important subsystem when it comes to Kafka. All messages are persisted to disk, so the performance of Kafka depends heavily on the performance of
the disks. Monitoring usage of both disk space and inodes (inodes are the file and directory metadata objects for Unix filesystems) is important, as you need to assure
that you are not running out of space.
It is also necessary to monitor the disk IO statistics, as this will tell us that the disk is being used efficiently.

### Network utilization
This is simply the amount of inbound and outbound network traffic, normally reported in bits per second. Keep in mind that every bit inbound to the Kafka broker will be a number of bits outbound equal to the replication factor of the topics, with no consumers. Depending on the number of consumers, inbound network traffic could easily become an order of magnitude larger on outbound traffic. Keep this in mind when setting thresholds for alerts.

## Logging
By simply logging all messages at the INFO level, you will capture a significant amount of important information about the state of the broker.

### Log files
#### kafka.controller
This logger is used to provide messages specifically regarding the cluster controller. At any time, only one broker will be the controller, and therefore only one broker will be writing to this logger. The information includes topic creation and modification, broker status changes, and cluster activities such as preferred replica elections and partition moves.

#### kafka.server.ClientQuotaManager
This logger is used to show messages related to produce and consume quota activities. While this is useful information, it is better to not have it in the main broker log file.

#### log compaction threads
Enabling the kafka.log.LogCleaner , `kafka.log.Cleaner`, and `kafka.log.LogCleanerManager` loggers at the `DEBUG` level will output information about the status of these threads

#### other useful logging info
1. `kafka.request.logger`: when turned on at either `DEBUG` or `TRACE` levels logs information about every request sent to the broker

## Client Monitoring
### Producer Metrics

\#   |Name             | JMX MBean
:--- |:---             | :---      
1    | Overall Producer| `kafka.producer:type=producer-metrics,client-id=CLIENTID`
2    | Per-Broker      | `kafka.producer:type=producer-node-metrics,client-id=CLIENTID,node-id=node-BROKERID`
3    | Per-Topic       | `kafka.producer:type=producer-topic-metrics,client-id=CLIENTID,topic=TOPICNAME`

#### Overall producer metrics
#### Per-broker and per-topic metrics

### Consumer Metrics
\#    |Name             | JMX MBean
:--- |:---              | :---      
1    | Overall Consumer | `kafka.consumer:type=consumer-metrics,client-id=CLIENTID`
2    | Fetch Manager    | `kafka.consumer:type=consumer-fetch-manager-metrics,client-id=CLIENTID`
3    | Per-Topic        | `kafka.consumer:type=consumer-fetch-manager-metrics,client-id=CLIENTID,topic=TOPICNAME`
4    | Per-Broker       | `kafka.consumer:type=consumer-node-metrics,client-id=CLIENTID,node-id=node-BROKERID`
5    | Coordinator      | `kafka.consumer:type=consumer-coordinator-metrics,client-id=CLIENTID`

#### Fetch manager metrics
#### Per-broker and per-topic metrics
#### Consumer coordinator metrics

## Quotas
\#   |Client         | Bean name
:--- |:---           | :---      
1    | Consumer | bean `kafka.consumer:type=consumer-fetch-manager-metrics,client-id=CLIENTID,attribute fetch-throttle-time-avg`
2    | Producer | bean `kafka.producer:type=producer-metrics,client-id=CLIENTID,attribute produce-throttle-time-avg`




## Lag Monitoring

## End-to-End Monitoring

[kafka-tools-url]: https://github.com/linkedin/kafka-tools