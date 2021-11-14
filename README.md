<!--
# Errata
-->

# GMIT Distributed Systems
# Lab: Fault Tolerant and Scalable Kafka
Instructions and starter code for the Distributed Systems lab Fault Tolerant and Scalable Kafka.

## Lab Objectives
In this lab you'll:
- Configure and run a Kafka cluster with multiple brokers
- Create a partitioned and replicated topic
- Test the fault-tolerance of the multi-broker cluster

## Introduction
In the last lab you set up a simple Kafka cluster consisting of a single Kafka broker. While this was useful as an introduction to Kafka, it's not a very realistic setup, because a single broker Kafka cluster has a single point of failure: if that broker goes down then no data can flow through Kafka,  amd the whole system you've built around Kafka does down with it.

In this lab we'll set up a more realistic scenario. We'll expand the cluster by adding more brokers and see that, not only does this prevent our cluster from having a single point of failure, it also allows us to spread the workload across the brokers.

- **cmder is the recommended terminal for this lab**. Kafka on Windows is managed by .bat scripts, and auto-complete doesn't work well for .bat scripts in git bash on Windows Terminal. The commands will still work fine regardless of the terminal used, though you may need to change the slash directions.


## Getting Started
1. Log in to your [Azure Lab Services](https://labs.azure.com/) VM.
2. In the VM, **open cmder (not Windows Terminal this time)**.

**All subsequent commands and instructions assume you're in the folder `C:/Users/comp08011/dev/kafka-2.5.0`**

## Restarting the Single-Broker Kafka Cluster
First off, let's get back to where we were at the end of the last lab, with a single broker Kafka cluster up and running, and a command-line producer and consumer running. Every time you start up the cluster after your VM restarts, you'll need to do 2 things:
1. Start Zookeeper (the one bundles with Kafka)
2. Start (at least one) Kafka broker
If you're using the command-line consumer and producer (which we are here) then you'll need to start those too.
(Use a new cmder tab for each of these operations. You can rename cmder tabs by right-clicking on them, it could be useful in helping to keep track of what's running in each tab).
- Start Kafka's bundled Zookeeper by running the `zookeeper-server-start.bat` script, passing in the zookeeper config file as a command-line argument
```
.\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
```
- Start a Kafka broker by running the broker start-up script (in a new cmder tab) and passing in the config file we just setup:
``` 
.\bin\windows\kafka-server-start.bat .\config\server.properties
```
- Run `kafka-console-producer.bat`, setting `--broker-list` to the address of our Kafka broker (`localhost:9092`), and `--topic` to our newly created topic, `gmit`:
```
.\bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic gmit
```
- Run the following command to consume messages from the `gmit` topic:
```
.\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic chat
```
- Publish some messages from the producer and verify that they're received at the consumer.

## Exploring Fault Tolerance of the Single-Broker Cluster
Now that the simple cluster is up and running let's explore its ability to tolerate faults.
- Kill the Kafka broker with `CTRL-C`
- Note the Zookeeper server logging messages about `Invalid session`, it's detected that the Kafka broker has done down
- Try to send messages using the command-line producer
  - You should find that it's not possible since the connection between the producer and the Kafka cluster has been lost

We can see that this simple cluster isn't fault-tolerant: the failure of a single broker causes the whole cluster to stop operating.

## Adding Brokers to the Cluster
### Configuring Additional Brokers
To make our cluster capable of withstanding failures, we'll need to add more brokers (we'll start 2 more). When one broker goes down, the others will keep the cluster running.  We'll need to configure these brokers in a similar way to how we configured our original broker.
- Restart the broker that you killed in the last step.
- Verify that messages can be sent from the producer and received by the consumer
- Our new brokers will need somewhere to store their logs, so we'll need to create two new logs folders for them.
```
mkdir logs-1
mkdir logs-2
```
- On the command-line go to the `config` folder in the Kafka install directory.
- Create two new configuration files for our two new brokers by copying the first broker's config:
```
$ cp server.properties server-1.properties
$ cp server.properties server-2.properties
```
- Open `server-1.properties` in a text editor (e.g. atom)
- Each broker will need a unique ID, so set `broker.id=1`
- Since we're testing our cluster by running all the brokers on a single node (our VM), we'll need to give them unique TCP port numbers to listen on:
  - uncomment the line starting with `listeners=`, and set the port number to 9093, i.e.. `listeners=PLAINTEXT://:9093`
- Set `log.dirs` to point to the first of the two new logs folders you created, i.e.  `log.dirs=/C/Users/comp08011/dev/kafka-2.5.0/logs-1`
- Configure the second broker by opening `server-2.properties` in the text editor. You'll update the same fields, but with different values, as follows:
```
broker.id=2
listeners=PLAINTEXT://:9094
log.dirs=/C/Users/comp08011/dev/kafka-2.5.0/logs-2
```

### Running Additional Brokers
- Open a new cmder tab or pane (you can make a new pane in the current tab by splitting it)
- Start a new Kafka broker, passing in one of the new config files we just set up:
```
.\bin\windows\kafka-server-start.bat config\server-1.properties
```
- In another new cmder tab/pane repeat the step above but with the other config file:
```
.\bin\windows\kafka-server-start.bat config\server-2.properties
````
In the logs output you should see the Kafka brokers running and outputting their ids:
```
[2021-11-14 21:57:10,822] INFO [KafkaServer id=1] started (kafka.server.KafkaServer)

[2021-11-14 21:57:46,800] INFO [KafkaServer id=2] started (kafka.server.KafkaServer)
```
- To help you keep track of these cmder panes later, rename each one something like b0, b1, b2 (for broker0, broker1 etc).

You should now have a Kafka cluster running with 3 brokers.

## Creating a Fault-Tolerant Topic
Now that we have a more robust cluster running, we can set up a new topic that makes use of the multiple brokers. This topic will be partitioned and replicated:
**Partitioning**: messages in the topic will be divided across multiple brokers for processing
**Replication**: messages in each broker will be copied to other brokers as a backup

- Create new topic `purchases` with rep-fac=3, part=3
```
.\bin\windows\kafka-topics.bat --bootstrap-server localhost:9092 --create --replication-factor 3 --partitions 3 --topic purchases
```
- Verify topic created with list
```
.\bin\windows\kafka-topics.bat --bootstrap-server localhost:9092 --list
```
- Descript topic created with list
```
.\bin\windows\kafka-topics.bat --bootstrap-server localhost:9092 --describe --topic purchases
```
- Output
```
λ .\bin\windows\kafka-topics.bat --bootstrap-server localhost:9092 --describe --topic purchases
Topic: purchases        PartitionCount: 3       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: purchases        Partition: 0    Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
        Topic: purchases        Partition: 1    Leader: 2       Replicas: 2,0,1 Isr: 2,0,1
        Topic: purchases        Partition: 2    Leader: 1       Replicas: 1,2,0 Isr: 1,2,0
```
- Note
  - partitioncount
  - replication factor
  - partition 0, leader 2, replicas, Isr
- Compare with gmit topic
```
.\bin\windows\kafka-topics.bat --bootstrap-server localhost:9092 --describe --topic gmit
```

### Test New Topic
- Kill consumer/producer
- Restart pointing to purchases
- Send messages
purchase1
purchase2
- Receive messages with consumer (use from-beginning)
- Note global ordering is not preserved across partitions


## Test Fault-Tolerance
- Shutdown broker 1
- Run describe, compare with previous
  - new leader elected for partition
- Run consumer again from beginning
  - see that all messages replicated, nothing lost **persistence**


  












## Conclusion
