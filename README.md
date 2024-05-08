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
Kafka also supports a newer mode of operation where brokers can coordinate with each other directly, without the need for zookeeper. This mode is called KRaft mode and is still experimental.

### Starting a cluster with docker-compose
To start a minimal kafka cluster with docker-compose, run the following command:
```bash
docker-compose up -d 
```
## What is an event?
An event is a record of something that has happened. It can be a user action, a sensor reading, a database change, or any other piece of data that is relevant to the system. Events are immutable, meaning that once they are created they cannot be changed. This makes them ideal for building real-time data pipelines and streaming applications.

## Producing & Consuming messages

### Schema registry
### Partitions & Consumer groups

## Production deployments

### Security
### High-availability & disaster-recovery
### Managed kafka platforms