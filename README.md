# Getting started with Kafka

## What's in this tutorial?
This tutorial is a hands-on introduction to Apache Kafka with step-by-step interactive chapters for getting up to speed as fast as possible. It covers the basics of Kafka, how to set up a Kafka cluster, how to produce and consume messages, how to manage and configure topics, consumers, schemas, and how to deploy Kafka in production.
This tutorial is designed for software engineers, data engineers, and system administrators who want to get started with Kafka. It utilizes docker-compose to set up a local Kafka cluster and UI tools to interact with the cluster.

## Prerequisites
To run the examples in this repo you need:
- docker with compose (version >=2.x)

## What is kafka?
Kafka is a distributed event streaming platform that can be used to build real-time data pipelines and streaming applications. It is designed to be fault-tolerant, scalable and high-performance.
Kafka is built around the concept of topics, which are logs of messages that are stored in a distributed manner across a cluster of servers. Producers write messages to topics and consumers read messages from topics. There can be any number (or zero) consumers and producers on a topic, which de-couples the producing and consuming systems from each other. 

### Kafka architecture
[![Kafka architecture](images/kafka_plane.png)](images/kafka_plane.png)

Kafka supports multiple sources and sinks for data, including databases, message queues, custom services, and sensors. It can be used to build real-time data pipelines, stream processing applications, and event-driven architectures.

## Starting a cluster
Let's have a look at the individual components of a Kafka deployment and set up a minimal kafka cluster using docker-compose.

A key feature of kafka is its fault tolerance and high availability. To achieve this kafka is designed to run in a cluster of multiple servers, in kafka-lingo these are called brokers. Brokers need to coordinate with each other on who handles what requests and who has which data. This coordination is normally done in a zookeeper ensamble. Zookeeper is a distributed coordination service that is used by many distributed systems to manage their state.

Kafka also supports a newer mode of operation where brokers can coordinate with each other directly, without the need for zookeeper. This mode is called KRaft mode and is still experimental, you can have a look at the [KRaft documentation](https://developer.confluent.io/learn/kraft/) for more information, or look in the [kafka-kraft.yml](kafka-kraft.yml) file in this repo for an example of how to set up a kafka cluster in KRaft mode.

### Starting a cluster with docker-compose
To start a minimal kafka cluster with docker-compose, run the following command:
```bash
docker compose -f kafka-minimal.yml up
```
This will start a single kafka broker and a single zookeeper instance. 
You'll see the logs of both the kafka broker and zookeeper in the terminal.



It might not be the most interesting setup, since we're not doing anything with the cluster yet, but it's a good starting point to get familiar with the kafka components.
You can look through the [kafka-minimal file](kafka-minimal.yml) to see how the cluster is set up.
[![Kafka minimal cluster](images/kafka_minimal.png)](images/kafka_minimal.png)
What you've now set up is a single kafka broker in a cluster managed by a single zookeeper.
The zookeeper instance will keep metadata information about the kafka cluster, such as what topics and their partitions exist on what broker and general health of the brokers, which enables high-availability.
The kafka broker is set up with three ports exposed:
* `DOCKER_HOST_IP:9092` for external access, bound to the docker host ip, which may be your local ip if you're using a host-based network in docker (default).
* `HOST.DOCKER.INTERNAL:29092` for docker-wide access for all other containers running in docker. 
* `kafka1:19092` which is available for all containers in the same docker-compose network. The last one is the one we'll use to interact with the broker in this tutorial.



To stop the cluster, press `Ctrl+C` in the terminal.
Then to de-provision the cluster run:
```bash
docker compose -f kafka-minimal.yml down
```
It's important to de-provision the cluster as it registers a broker id with the zookeeper instance. If you don't de-provision the cluster, you'll get an error when starting the cluster again.


Now that we have the basic cluster set up, let's start using it to send and receive some data!

## Consuming messages
We'll start by setting up a small kafka cluster and a [kafka-connect-datagen](https://github.com/confluentinc/kafka-connect-datagen) container. [Kafka-connect](https://docs.confluent.io/platform/current/connect/index.html) is an integration framework tha enables kafka to connect to various data sources and sinks. It also includes a number of random datagenerators that we'll use to push some data into our kafka topic.
The kafka-connect container will be built locally with some additional environment variables set, so it might take a longer time to start.
```bash
docker compose -f consuming.yml up -d --build 
```
Next we'll upload a kafka-connect config to the kafka-connect container.
```bash
curl -X POST -H "Content-Type: application/json" --data @shared/config/user_datagen.config http://localhost:8083/connectors
```
If you don't have curl installed, you can use the following command instead to use curl inside the kafka-connect container: 
```bash
docker compose -f consuming.yml exec connect curl -X POST -H "Content-Type: application/json" --data '{
  "name": "datagen-users",
  "config": {
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic": "users",
    "quickstart": "users",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "max.interval": 1000,
    "iterations": 10000000,
    "tasks.max": "1"
  }
}' http://localhost:8083/connectors
```
The user_datagen.config file has the following content:
```json
{
  "name": "datagen-users",
  "config": {
    "connector.class": "io.confluent.kafka.connect.datagen.DatagenConnector",
    "kafka.topic": "users",
    "quickstart": "users",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter.schemas.enable": "false",
    "max.interval": 1000,
    "iterations": 10000000,
    "tasks.max": "1"
  }
}
```
It sets up a connector using the Datagen Connector class, which is the random data generator. It will use the built-in users quickstart schema and push 10 million records to the users topic, one every second.
The user schema looks like this:
```avro
{
        "namespace": "ksql",
        "name": "users",
        "type": "record",
        "fields": [
                {"name": "registertime", "type": {
                    "type": "long",
                    "arg.properties": {
                        "range": {
                            "min": 1487715775521,
                            "max": 1519273364600
                        }
                    }
                }},
                {"name": "userid", "type": {
                    "type": "string",
                    "arg.properties": {
                        "regex": "User_[1-9]"
                    }
                }},
                {"name": "regionid", "type": {
                    "type": "string",
                    "arg.properties": {
                        "regex": "Region_[1-9]"
                    }
                }},
                {"name": "gender", "type": {
                    "type": "string",
                    "arg.properties": {
                        "options": [
                            "MALE",
                            "FEMALE",
                            "OTHER"
                        ]
                    }
                }}
        ]
}
```
It has four fields: registertime, userid, region and gender. It will use the userid as the key for the messages (Not included in the schema above).


Next we'll start the built-in [kafka-console-consumer](https://docs.confluent.io/kafka/operations-tools/kafka-tools.html#kafka-console-consumer-sh) to consume the messages from the users topic. We'll read it from the beginning, so we'll see all the messages that have been pushed to the topic, and then a new one every second.
```bash
docker compose -f consuming.yml exec connect kafka-console-consumer --topic users --bootstrap-server kafka1:19092  --property print.key=true --from-beginning
```
You should see the messages being printed to the terminal. You can stop the consumer by pressing `Ctrl+C` in the terminal.
```bash
User_3  {"registertime":1491380392989,"userid":"User_3","regionid":"Region_2","gender":"MALE"}
User_4  {"registertime":1518583869842,"userid":"User_4","regionid":"Region_5","gender":"FEMALE"}
User_1  {"registertime":1515552966375,"userid":"User_1","regionid":"Region_3","gender":"FEMALE"}
User_8  {"registertime":1490022276012,"userid":"User_8","regionid":"Region_9","gender":"MALE"}
User_3  {"registertime":1517391727563,"userid":"User_3","regionid":"Region_9","gender":"MALE"}
User_6  {"registertime":1503053130484,"userid":"User_6","regionid":"Region_1","gender":"MALE"}
User_6  {"registertime":1502943511939,"userid":"User_6","regionid":"Region_2","gender":"FEMALE"}
User_1  {"registertime":1506169641495,"userid":"User_1","regionid":"Region_9","gender":"OTHER"}
User_1  {"registertime":1509560532058,"userid":"User_1","regionid":"Region_9","gender":"OTHER"}
User_3  {"registertime":1494983587808,"userid":"User_3","regionid":"Region_5","gender":"MALE"}
User_6  {"registertime":1505247982243,"userid":"User_6","regionid":"Region_9","gender":"OTHER"}
```
The first part of each line is the key and the json object is the value of the message.

Before continuing, stop and deprovision the containers:
```bash
docker compose -f consuming.yml down
```
## What is an event?
An event is a record of something that has happened. It can be a user action, a sensor reading, a database change, or any other piece of data that is relevant to the system. Events are immutable, meaning that once they are created they cannot be changed. This makes them ideal for building real-time data pipelines and streaming applications.
Events carry both information of what has happened (State) and when it happened.

An event can either be transactional in nature, meaning that it contains both the previous value and the new value, enabling validations before a consuming system actualizes the change, or (more commonly) it can be a fact, meaning that it only contains the new value and the consuming system needs to be able to handle the change, possibly without the previous value (using upsert methodology).

Events are usually small in nature, containing only the relevant data for the event. 

Kafka models events as key/value pairs. The value part of the event is typically a representation of an application domain object and is often serialized as [JSON](https://www.json.org/json-en.html), [Avro](https://avro.apache.org/docs/), or [Protobuf](https://protobuf.dev/).

The key part can be a complex object, but is often a primitive type such as a string or integer. The key represents a unique identifier for a domain object, such as a UserID or a SensorID, and does not uniquely identify the event itself. The key is used to partition the data on the topic, enabling parallel processing of the data, without compromising the order of events for a specific key.

Let's produce some events to a topic to see how this works.

```bash
docker compose -f producing.yml up -d
```

This will set up a kafka cluster, a consumer that we can view the messages with and a producer which we will use to produce messages to the topic.

After everything is started, attach to the consumer with the following command:
```bash
docker compose -f producing.yml attach consumer
```
And then start a new terminal window, navigate to the project root folder and run the following command to start the producer:
```bash

docker compose -f producing.yml exec producer kafka-console-producer --topic my-topic --bootstrap-server kafka1:19092 --property parse.key=true --property key.separator=":"
```
This will give you a prompt where you can write messages to the topic. The key and value are separated by a colon. 
```bash
>key1:{"name":"John","age":30}
```
You should see the message being printed in the consumer terminal.
```bash
key1  {"name":"John","age":30}
```
You can stop the producer by pressing `Ctrl+C` in the terminal.
And again remember to stop and deprovision the containers:
```bash
docker compose -f producing.yml down
```

Of course manually producing messages to a topic is not very useful in a real-world scenario. And the typical scenario is that either the producer, consumer or both are using a [language-specific client library](https://docs.confluent.io/kafka-client/overview.html) to interact with the kafka cluster, [kafka-connect](https://docs.confluent.io/platform/current/connect/index.html), or a 3rd party system compatible with Kafka, such as [Debezium](https://debezium.io/) for exporting database changelogs, [Apache Flink](https://flink.apache.org/), [kSQLdb](https://ksqldb.io/) or similar systems. 



### Partitions & Consumer groups
coming soon
### Schema registry
coming soon
## Production deployments
coming soon
### Security
coming soon
### High-availability & disaster-recovery
coming soon
### Managed kafka platforms
coming soon