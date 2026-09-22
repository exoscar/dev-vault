Created by LinkedIn, now open source mainly maintained by Confluent, IBM, Cloudera

Why Kafka
- distributed ,resilient architecture, fault tolerant
- Horizontal scalability
	- Can scale to 100s of brokers
	- can scale to millions of messages per second
- High performance(latency of less than 10ms) - real time
- used by the 2000+ firms, 80% of the Fortune 100

Apache Kakfa: Use Cases
- Messaging systems
- activity Tracking
- gather metrics from many diff locations
- application logs gathering
- Stream processing 
- Integration with Flink, Spark, Storm, Hadoop, and many other big data technologies

Real World Ex
- Netflix uses kafka to apply recommendations in real time while watching shows
- Uber uses Kafka to gather user, taxi and trip data in real-time to compute and forecast demand, and compute surge pricing in real time


#### Kafka Topics
- `Topics`: a particular stream of data
- it can be logs, purchases, tweets and etc
- like a table in a database(without all the constraints)
- Can have as many topics as you want
- A topic is identified by its name
- support any kind of message format
- The sequence of messages is called a `data stream`
- you cannot query topics, instead use kafka producers to send and Kafka Consumers to read the data
### Partitions and Offsets
- Topics are split in partitions
	- Messages within each partition are ordered
	- each message withing a partition gets an incremental id, called `offset`
- Kafka topics are immutable: once data is written to a partition, it cannot be changed



- Data is kept only for a limited time (default is one week - configurable)
- Offset only have a meaning for a specific partition
	- Offsets are no re-used even if pervious messages have been deleted
- Data is assigned randomly to partition unless a key is provided
- you can have as many partitions per topics as you want

### Producers
![[Kafka producers ills.png]]
- producers write data to topics(which are made of paritions)
- producers know to which partition to write to(and which kafka broker has it)
- in case of kakfa broker failures, Producers will automatically recover

#### Message Keys
- producers can choose to send a key with the message(string, number, binary, etc)
- if key=null, data is sent in round robin way(partition -0, then 1, then 2)
- if key!=null, then all messages for that key will always go to the same partition (hashing)

Kafka Messages anatomy
![[kafka_message_anatomy.png]]

### Kafka Message Serializer
- Kafka only accepts bytes as an input from producers and sends bytes out as an output to consumers
- Message Serialization means transforming objects / data into bytes
- they are used on the value and the key
- Common Serializers
	- String(incl. JSON)
	- Int, Float
	- Avro
	- Protobuf
#### Kafka Message Key Hashing
![[kafka key hasking.png]]


### Consumers
- Consumers read data from a topic (identified by name) - pull model
-  Consumers automatically know which broker to read from
- in case of broker failures, consumers know how to recover
- Data is read in order from low to high offset within each partitions

#### Consumer Deserializer
- Deserailize indicates how to transform bytes into objects/data
- they are used on the value and the key of the message
- Common Deserializers
	- String(incl. JSON)
	- Int, Float
	- Avro
	- Protobuf
- The serialization / deserailization type must not change during a topic lifecycle

### Consumer Groups
- All the consumers in an application read data as a consumer group
- Each consumer within a group reads from exclusive partitions

#### What if too many consumers
- if you have more consumers than paritions, some consumers will be inactive

![[manyconsumer.png]]
#### Multiple Consumers on one topic
- In Apache Kafka it is acceptable to have multiple consumer groups on the same topics 
![[multiple consumer one topic.png]]

### Consumer Offsets

- Kafka stores the offsets at which consumer group has been reading
- The offsets commited are in Kafka topic named __consumer_offsets
- When a consumer in a group has processed data received from Kafka, it should be periodically commitng the offsets(the Kafka broker will write to __consumer_offsets,not the group itself)
- if a consumer dies, it will be able to read back from where it left

### Delivery Semantics for consumers
- By default, Java Consumers will automatically commit offsets(at least once)
- There are 3 delivery semantics if you choose to commit naturally
- At least once(usually preferred)
	- Offsets are commited after the message is processed
	- if the processing goes wrong, the message will read again
	- this can result in duplicate processing of messages. Make sure your processing is idempotent(i.e processing again the messages wont impact your systems)
- At most once
	- Offsets are commited as soon as messages are received
	- if the processing goes wrong, some messages will be lost(they wont be read again)
- Exactly once
	- For Kafka => Kafka workflow: use the Transactional API(easy with Kafka Streams API)
	- For Kafka => External System: use an idempotent consumer

>Idempotent:  an action that can be applied multiple times without changing the final result or system state beyond the very first time you do it


## Kafka Brokers
- A kafka cluster is composed of multiple brokers(servers)
- Each broker is identified with its ID (integer)
- Each broker contains certain topic partitions
- After connecting to any broker(called a bootstrap broker), you will be connected to the cluster(Kafka clients have smart mechanics for that)
- A good number to get started is 3 brokers, but some big cluster have over 1000 brokers

### Brokers and Topics
partitions are distributed across all brokers
![[brokers and topics.png]]

### Kafa Broker Discovery
- Every Kafka broker is also called a "bootstrap server"
- that means that you only need to connect to one broker, and the clients will know how to be connected to entire cluster(smart_client)
- each brokers knows about all brokers, topics and partitions (metadata)

### Topic replication factor
- topics should have a replication factor >1 (usually between 2 and 3)
- This way if a broker is down, another broker can serve the data 

#### Concept of Leader for a partition
- At anytime only one broker can be a leaders for a given partition
- producers can only send data to the broker that is leader of a partition
- the other brokers will replicate the data
- therefore, each partition has one leader and multiple ISR (in-sync replica)
- ![[Leaders.png]]
#### Default producers & consumer behaviour with leaders
- Kafka producers can only write to the leader broker for a partition
- Kafka consumers by default will read from the leader broker for a partition



### Kafka Consumers Replica Fetching(Kafka v2.4+)
- since Kafka 2.4, it is possible to configure consumers to read from the closest replica
- this may help improve latency, and also decrease network cost if using the cloud

### Producer Acknowledgements (acks)
![[producer ack.png]]

![[durability.png]]


### Zookeeper
- Zookeeper manages brokers(keeps a list of them)
- Zookeeper helps in performing leader election for partitions
- Zookeeper sends notification to Kafka in case of changes(eg. new topic, broker dies, broker comes up, delete topics, etc.. )
- Kafka 2.x can't work without Zookeeper
- Kafka 3.x can work without zookeeper(KIP-500) - using Kafka Raft instead
- Kafka 4.x will not have Zookeeper
- Zookeeper by design operates with an odd number of servers (1, 3, 5, 7)
- Zookeeper has a leader(writes) the rest of the servers are followers(reads)
- (Zookeeper does not store consumer offsets with Kafka>v0.10)

Zookeeper Cluster (ensemble)

![[Zookeeper.png]]

![[zookeeper2.png]]

## About Kafka KRaft
 ![[Zookeeper3.png]]
 