# Hello world in Kafka using Python

Howdy! Welcome to blog series by Timber.io on various tools and technologies.

This blog is for you if you've,

- Ever marvelled what is Kafka? 
- Heard of streaming/queueing/messaging systems but wondering why should one use it? 
- What are the true benefits of using them?
- How would it fit with your current backend architecture? 
- You might just be eager to get started with using Kafka! 

<Illustration>

## What is this blog about ? 
You might have got a hint of it already. This blog is about understanding what is Kafka, apprehending the need for a tool like Kafka and then getting started with it using Python.

---
## What is Kafka? Why should one use it?
In short, Kafka is a distributed streaming and message queueing platform.

> Oh wait! What does that even mean?

A streaming platform is a system that can perform the following:
1. Store and process streams of data in real time
2. Allow applications and/or services to publish and subscribe to these data streams at scale

> Interesting! How different or similar is it from traditional databases?

Kafka is NOT a database. Nevertheless, it has few similarities and quite an amount of differences with respect to traditional databases.

Like traditional databases, you can store data into Kafka which can be persistent, checksummed and replicated for fault tolerance. This is one reason why Kafka is not just a message queue but a streaming platform. In addition, Kafka has something called as topics which is similar to tables in PostgreSQL or collections in MongoDB. 

Unlike traditional databases, Kafka deals with real time data that can be published or consumed by multiple applications at the same time. Alongside, Kafka allows you to process the data streams but this is totally different when compared to performing CRUD operations or running simple to complex queries on traditional databases. 

> That sounds convincing! But why would one need a system like this?

### 1. Simplify the backend architecture

Look at how a complex architecture can be simplified and streamlined with the help of Kafka

![Without Kafka](https://www.confluent.io/wp-content/uploads/data-flow-ugly-1-768x427.png "Without Kafka")

![With Kafka](https://www.confluent.io/wp-content/uploads/data-flow-768x584.png "With Kafka")

### 2. Universal pipeline of data

Kafka is build for scale and thus it ensures that the data is always reliable at any point in time. It also supports strong mechanisms for recovery from failures.

All these features allow Kafka to become the true source of data or a universal pipeline of data for your architecture. This will enable you to easily add new services and applications to your existing infrastructure or even allow you to rebuild existing databases or migrate legacy systems with less time and effort.

# Getting Started with Kafka

## Installation

Installing Kafka is a fairly simple process. Just follow the given steps below:

1. Download the latest 1.1.0 release of [Kafka](https://www.apache.org/dyn/closer.cgi?path=/kafka/1.1.0/kafka_2.11-1.1.0.tgz)
2. Un-tar the download using the following command:
`tar -xzf kafka_2.11-1.1.0.tgz`
3. cd to kafka directory to start working with it:
`cd kafka_2.11-1.1.0`

## Starting the Server

Kafka makes use of something called as [ZooKeeper](https://zookeeper.apache.org/). Thus, we need to first start the ZooKeeper server followed by the Kafka server. This can be achieved using the following commands:

```bash
# Start ZooKeeper Server
bin/zookeeper-server-start.sh config/zookeeper.properties

# Start Kafka Server
bin/kafka-server-start.sh config/server.properties
```

## Understanding Kafka

Here is a quick introduction to some of the core concepts of Kafka architecture:
1. Kafka is run as a cluster on one or more servers
2. Kafka stores streams of records in categories called `topics`. Each record consists of a key, value and a timestamp
3. Kafka works on the publish-subscribe pattern. Thus, it allows some of the applications to act as `producers` and publish the records to Kafka topics. Similarly, it allows some of the applications to act as `consumers` and subscribe to Kafka topics and process the records produced by it
4. Alongside, `Producer API` and `Consumer API`, Kafka also offers `Streams API` for an application to work as a stream processor and `Connector API` through which we can connect Kafka to other existing applications and data systems

## Creating Kafka Topics

Let us start by creating a `sample` kafka topic with a single partition and replica. This can be done using the following command:
```
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic sample
```

Now, let us list down all of our Kafka topics to check if we have successfully created our `sample` topic. We can make use of the `list` command here:
```
bin/kafka-topics.sh --list --zookeeper localhost:2181
```

Optionally, you can also make use of the `describe topics` command for more details on a particular Kafka topic. This can be done as follows:
```
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic sample
```

## Creating Producer and Consumer

Creating a producer and consumer can be a perfect `Hello, World!` example to learning Kafka but there are multiple ways through which we can achieve it. Some of them are listed below:

1. Command line client provided as default by Kafka
2. [kafka-python](https://github.com/dpkp/kafka-python)
3. [PyKafka](https://github.com/Parsely/pykafka)
4. [confluent-kafka](https://github.com/confluentinc/confluent-kafka-python)

While each of them have their own set of advantages and disadvantages, we will be making use of `kafka-python` in this blog to achieve a simple producer and consumer setup in kafka using python.

# Kafka with Python

Before you get started with the following examples, ensure that you have `kafka-python` installed in your system:

```
pip install kafka-python
```

## Kafka Consumer

Enter the following code snippet in an interactive python shell:

```
from kafka import KafkaConsumer
consumer = KafkaConsumer('sample')
for message in consumer:
    print (message)
```

## Kafka Consumer

Now that we have a consumer listening to us, let us create a producer which generates messages that are published to Kafka and thereby consumed by our consumer created earlier:

```
from kafka import KafkaProducer
producer = KafkaProducer(bootstrap_servers='localhost:9092')
producer.send('sample', b'Hello, World!')
producer.send('sample', key=b'message-two', value=b'This is Kafka-Python')
```

You can now revisit the consumer shell to check if it has received the records sent from the producer through our kafka setup. 

Thus, a simple `Hello, World! in Kafka using Python`.
