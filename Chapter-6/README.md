# Reliable Data Delivery

- Attribute of the whole system, since its conception
- Shouldn't only be relied on a unique component
- What does a system guarantee?
  - ACID for databases: atomicity, consistency, isolation, durability

## What does Kafka guarantee?

- Message order inside a partition
  - One producer sends A and then B, the partition will receive A then B, the offset of B will be higher then A, and a consumer will receive A then B
- Produced messages are considered "committed" when they are written to the partitions in all its in-sync replicas (not necessarily flushed to disk -> page cache)
- Producers can receive acknowledgements when a message is committed
- Committed messages will not be lost as long as one replica remain alive
- Consumers can only read committed messages

There are trade-offs that administrators and developers must be aware

Examples: o reliably and consistently store messages versus other important considerations such as availability, high throughput, low latency, and hardware costs

## Replication (Chapter 5 review)

  Multiple replicas per partition: core of all Kafka guarantees

### Remembering
#### About partitions
- Each Kafka topic is broken into partitions, which are the basic data building blocks
- Each partition is stored on a single disk
- Kafka guarantees the message order inside a partition
- A partition can be online (available) or offline (unavailable)

#### About replicas
- Each partition can have multiple replicas, one being a designated leader
- All events are produced to and consumed from the leader replica
- Other replicas just need to stay in sync with the leader and replicate all the recent events on time
- If the leader becomes unavailable, one of the in-sync replicas becomes the new leader

##### In-sync replica
A replica is considered in-sync if it is the leader for a partition, or if it is a follower that:
- Has an active session with Zookeeper, meaning, it sent a heartbeat to Zookeeper in the last 6 seconds (configurable).
- Fetched messages from the leader in the last 10 second (configurable).
- Fetched the most recent messages from the leader in the last 10 seconds. That is, it isn’t enough that the follower is still getting messages from the leader; it must have almost no lag.

##### Out-of-sync replica
An out-of-sync replica can become in-sync again if:
- Connects again with Zookeeper
- Catches up with the most recent messages written by the leader

A flipping state of replicas can mean a misconfiguration of Java's garbage collection on a broker, which can make a broker lose connection to Zookeeper
An in-sync replica that is slightly behind can slow down _producers_ and _consumers_. In the other hand, if this replica is marked as out-of-sync, Kafka no longer waits for it to get messages, avoiding a performance impact. The problem with this is that with fewer in-sync replicas, the replication factor is lower and, therefore, there is risk of downtime or data loss.

## Broker Configuration
We're going to analyse 3 broker configuration parameters that changes Kafka's behavior regarding reliable message storing:

### Replication factor
#### Configuration names
1. `replication.factor`: topic level configuration
2. `default.replication.factor`: for automatically created topics

#### Advantages
- Higher availability
- Higher reliability
- Fewer disasters

#### On the other side...
- Higher disk space is needed
- Synchronization takes longer

#### So, what to do?
Replication factor of 1: if you are ok with a topic being sometimes, and you will for sure! As Kafka brokers usually are restarted.
Replication factor of 2: Nice and neat, but if one of the brokers is lost, it can send the cluster to an unstable state, forcing you to restart another broker, meaning: unavailability!
Replication factor of 3: For most of the cases, this is the ideal. In rare cases it's not safe enough.

#### That's not all folks!
Placing replicas in different brokers seems to be a good idea, except when one broker down means that another broker is down too. Like when the brokers are in the same rack. When a top-of-rack switch misbehaves, the topic can lose its availability.
To protect against rack-level misfortune there's a broker configuration parameter to configure the rack name for each broker: `broker.rack`

### Unclean Leader Election
#### Configuration names
1. `unclean.leader.election.enable` if true, allows out-of-sync replicas to become leaders

#### The clean/unclean dilemma
Deciding between a clean or an unclean leader election can be tricky if you don't know what you are doing. When choosing a clean leader election, you can make your system unavailable for some time, as it can make your system wait for your currently offline leader to be online again. In the other hand, an unclean leader election can do a mess in the messages consumers receive.
##### Clean leader election
###### Advantages
If you choose it, only in-sync replicas can become a leader, this guarantees that all acknowledge and committed messages will be sent to the consumers, which makes the system more reliable.

###### Disadvantages
A system with this configuration can have a lower availability, since, in some scenarios, the leader is the only in-sync replica and it must be back online for the topic to be available again.

##### Unclean leader election
###### Advantages
As the leader goes down, any other replica can be a leader, avoiding the system of being unavailable

###### Disadvantages
An unclean leader election can lead to loss of already committed produced messages, messing up with the reliability of consumed messages

### Minimum In-Sync Replicas
#### Configuration names
1. `min.insync.replicas` both topic and broker levels have this name and indicates the minor number of in-sync replicas for a message to be considered as committed.
   
If you set this config to, for example, with 2 minimum in-sync replicas and there's only one in-sync replica, producers that attempt to send data will receive `NotEnoughReplicasException`. Consumers can continue reading existent data. This prevents the undesirable situation where data is produced and consumed, only to disappear when unclean election occurs.

## Using Producers in a Reliable System
- We learned about reliable replication, if the replicas configurations can lead to unreliable scenarios, the producers can be found in a unrecoverable scenario, which them don't event know that a message was lost
- Nonetheless, if the replicas configurations avoid unreliable scenarios but the producer can't treat kafka error responses, the system will lose messages an the reliability will be compromised

To avoid this, Kafka producers must pay attention to:
- Use the correct acks configuration to match reliability requirements
- Handle errors correctly both in configuration and in code

### Send Acknowledgments
Producers acknowledgment modes:
1. `acks=0` means that a message is considered to be written successfully to Kafka if the producer managed to send it over the network. Leader election? Don't care! Sometimes I fell like an UDP in the application layer.
2. `acks=1` means that the leader will send either an acknowledgment or an error the moment it got the message and wrote it to the partition data file. Doesn't care about replication, meaning if the leader crashes before replication, bye bye messages!
3. `acks=all` means that the leader will wait until all in-sync replicas got the message before sending back an acknowledgment or an error. In conjunction with the `min.insync.replica` configuration on the broker, this lets you control how many replicas get the message before it is acknowledged.

### Configuring Producer Retries
#### Retriable errors
This kind of errors can be handled by producers without any action needed by the developers. For example, if the broker returns the error `LEADER_NOT_AVAILABLE`, the producer can try to send the message again

#### Unretriable errors
Means that retrying will have no effect whatsoever is done by the producer. The error `INVALID_CONFIG` is one example.

#### What's the best approach?
As always it depends on your objective. If your objective is never to lose a message, then you should let the producer keep trying to send the message when it encounters a retriable error.
Note that when retrying there's a risk of sending the message twice. The approach to solve this is in the application level, like, for example, having an message id to avoid it of being processed twice.

#### Additional Error Handling
Developers are still responsible for managing:
- Nonretriable errors
- Serialization errors
- Actions after too many retries

## Using Consumers in a Reliable System
- Consumers only get consistent data, i.e., committed messages
- Consumers need only to keep track of which messages they've read

### Important Consumer Configuration Properties for Reliable Processing
There are four consumer configuration properties that are important to understand in order to configure your consumer for a desired reliability behavior.
1. `group.id` consumers in the same group id and same topic will balance the message reading between the topics
2. `auto.offset.reset` what to do when there's no committed offset? `earliest` minimizes data loss but can make messages to be read twice. `latest` minimizes duplicated processing, but it can lose some messages.
3. `enable.auto.commit` can be handy if you do all message processing inside the consumer poll loop, otherwise, you may need to commit the offsets manually.
4. `auto.commit.interval.ms` if it's low, can add some overhead but avoids duplicated messages when a consumer stops.
   
### Explicitly Committing Offsets in Consumers
What do we need to pay attention when you choose to commit offsets manually?
1. Always commit offsets after events were processed
2. Commit frequency is a trade-off between performance and number of duplicates in the event of a crash
3. Make sure you know exactly what offsets you are committing
4. Handle consumer rebalances properly
5. Consumers may need to retry
6. Consumers may need to maintain state
7. Handling long processing times
8. Exactly-once delivery

## Validating System Reliability
### Validating Configuration
- It helps to test if the configuration you’ve chosen can meet your requirements.
- It is good exercise to reason through the expected behavior of the system. This chapter was a bit theoretical, so checking your understanding of how the theory applies in practice is important.

Tools to help with validation (inside `org.apache.kafka.tools`):
- VerifiableProducer
- VerifiableConsumer
They can run as command-line tools, or be embedded in an automated testing framework

Scenarios to test:
- Leader election
- Controller election
- Rolling restart
- Unclean leader election test

### Validating Applications
Recommended failure conditions to be tested by an application
- Clients lose connectivity to the server (your system administrator can assist you in simulating network failures)
- Leader election
- Rolling restart of brokers
- Rolling restart of consumers
- Rolling restart of producers

The results depends on the expected behavior that were planned while developing the application.

### Monitoring Reliability in Production
There are more details on monitoring the cluster in the Chapter 9, but it's also important to monitor the clients and the data flow through the system.

#### JMX Metrics
Kafka's Java clients include JMX, which provides
- Client-side status and events 
- Producer error-rate per second
- Producer retry-rate per second
- Producer log errors 
- Consumer lag

#### Monitoring data flow
Kafka provides a timestamp when the message was produced. You will need to keep track if the messages are being consumed within a reasonable amount of time.