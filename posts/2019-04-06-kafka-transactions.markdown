---
title: Kafka transactions - the tricky bits
---
Or in other words: why should you use separate transactional Kafka producer per consumer group and partition?

## 1. Intro

While working on exactly-once delivery in systems involving Kafka, or when preparing for giving workshops for financial institutions, I wanted to fully understand the guarantees provided by Kafka. However, there was always one thing that was left for me unclear: in docs (links to kafka doc on transactions, spring docs) as well as in articles (link to confluent transactions, exactly-once processing on confluent blog). The were only hints of the problem:

> For instance, in a distributed stream processing application, suppose topic-partition tp0 was originally processed by transactional.id T0. If, at some point later, it could be mapped to another producer with transactional.id T1, there would be no fencing between T0 and T1. So it is possible for messages from tp0 to be reprocessed, violating the exactly once processing guarantee.
>
> Practically, one would either have to store the mapping between input partitions and transactional.ids in an external store, or have some static encoding of it.

Quite similar explanation was given in Spring docs:

> 2.1.11 Transactional Id
> When a transaction is started by the listener container, the transactional.id is now the transactionIdPrefix appended with &lt;group.id&gt;.&lt;topic&gt;.&lt;partition&gt;. This is to allow proper fencing of zombies as described here. TODO link

So, what I could understand is that I should use separate transactional ids for each partition consumed, to enable proper zombie fencing and prevent duplicate messages. But still, I didn’t fully understand: why?

This blog post is for anyone asking the same question. The assumption is that the reader already knows about Kafka basics (eg partitions, consumer groups) and tries to understand transactions in Kafka eg by reading this confluent blog post.

## 2. Scenario

Let’s start with a simple case of consuming from Kafka topic and producing to another Kafka topic. This looks like that:

<img src="/images/2.1_overview.png" />

How the above could fail to provide exactly-once guarantees?

As you are probably aware, Kafka keeps track of consumed messages using consumer offsets.

<img src="/images/2.2_pull_msg.png" />

So, if after committing the offset, we fail to submit the output message:


<img src="/images/3.3_failed_send.png" />

Then, after restarting the worker, we will start from the latest committed offset and the message will be lost:


<img src="/images/3.4_failed_send_cont.png" />

Or, if we submit the message, but fail to commit the offset:

<img src="/images/3.5_offset_commit_fail.png" />

Then, we will get duplicated messages:

<img src="/images/3.6_offset_commit_fail_cont.png" />

To make the above setup fault-tolerant we need to make sure that consumer offsets are atomically saved together with effects of our computation. In our case our output effect is producing a message to another Kafka topic. Fortunately, Kafka producer’s API supports this with a method:

```Java
KafkaProducer##sendOffsetsToTransaction(offsets, consumerGroup)
```

For more details see the javadoc.

Using the above method in our transactions will make our processing exactly-once for the above setup. But what will happen when we want to scale up?

## 3. More complicated scenario

Increasing the parallelism of our processing can be done in multiple ways, depending on where our bottleneck is:
Increasing number of partitions (in/out)
Increasing number of workers
Both of the above

Typically, the bottleneck of our setup will be the number of workers, as single worker usually won’t be able to saturate decent network links or the Kafka broker, but let’s assume first that we scale by increasing the number of partitions:

<img src="/images/4.1_second_scenario.png" />

In such setup, the main concerns for failure are communication problems or the failure of the worker itselfs. Failure of the broker is out of scope of this blog post, as Kafka has its own fault-tolerance mechanisms for that when used with proper configuration.

When committing offsets atomically with the output of our computation, the main cause of concern for us is the failure of the worker during the ongoing transaction or during commit. Let’s consider 1 scenario:

An ongoing transaction:

<msg seq diagram, processing first message sending offsets, sending output message, failure, restarting worker, transactional producer init>

DONE: find out what happens, when restarting process with an ongoing transaction
DONE: ongoing is aborted
DONE: test it on my own

TODO: add code used to generate the below

```
$ ./bin/kafka-dump-log.sh --files /tmp/kafka-logs/topic-test-0/00000000000000000000.log \
> --print-data-log
Dumping /tmp/kafka-logs/topic-test-0/00000000000000000000.log
Starting offset: 0
offset: 0 ... producerEpoch: 17 ... key: thisIsMessageKey payload: thisIsMessageValue1
offset: 1 ... producerEpoch: 18 ... endTxnMarker: ABORT coordinatorEpoch: 0
offset: 2 ... producerEpoch: 19 ... key: thisIsMessageKey payload: thisIsMessageValue2
offset: 3 ... producerEpoch: 19 ... endTxnMarker: COMMIT coordinatorEpoch: 0
```
<!--
$ ./bin/kafka-dump-log.sh --files /tmp/kafka-logs/topic-test-0/00000000000000000000.log --print-data-log 
Dumping /tmp/kafka-logs/topic-test-0/00000000000000000000.log
Starting offset: 0
offset: 0 position: 0 CreateTime: 1552341204207 isvalid: true keysize: 16 valuesize: 19 magic: 2 compresscodec: NONE producerId: 0 producerEpoch: 17 sequence: 0 isTransactional: true headerKeys: [] key: thisIsMessageKey payload: thisIsMessageValue1
offset: 1 position: 103 CreateTime: 1552341204531 isvalid: true keysize: 4 valuesize: 6 magic: 2 compresscodec: NONE producerId: 0 producerEpoch: 18 sequence: -1 isTransactional: true headerKeys: [] endTxnMarker: ABORT coordinatorEpoch: 0
offset: 2 position: 181 CreateTime: 1552341204644 isvalid: true keysize: 16 valuesize: 19 magic: 2 compresscodec: NONE producerId: 0 producerEpoch: 19 sequence: 0 isTransactional: true headerKeys: [] key: thisIsMessageKey payload: thisIsMessageValue2
offset: 3 position: 284 CreateTime: 1552341204665 isvalid: true keysize: 4 valuesize: 6 magic: 2 compresscodec: NONE producerId: 0 producerEpoch: 19 sequence: -1 isTransactional: true headerKeys: [] endTxnMarker: COMMIT coordinatorEpoch: 0
-->

Thanks to using the same transactional id, Kafka’s zombie fencing mechanism works properly: the Kafka broker will assign the new worker a new epoch and abort transactions for previous instance of the producer (old epoch + the same transactional id). The initialization of the new producer will complete only after aborting transaction for previous producer instance:

This method does the following: 1. Ensures any transactions initiated by previous instances of the producer with the same transactional.id are completed. If the previous instance had failed with a transaction in progress, it will be aborted. If the last transaction had begun completion, but not yet finished, this method awaits its completion.

In this scenario, everything worked properly, no duplicates were produced, no messages dropped.

## 4. Failure of transactions caused by invalid usage

Let’s see what would happen if we try to scale our processing by increasing the number of worker processes:

<TODO describe possibilities for scaling: threads, processes, nodes>

DONE <diagram showing 2 in partitions, 2 workers with separate transactional producers, 2 out partitions>

In this case, following scenario will cause problems:

<msg seq diagram showing, first producer sends offsets, message, hangs (GC or network problems), consumer rebalance happens, second producer processes messages sends and commits, first producer wakes up and commits, separate transactional ids -> duplicates!!!>

So, our exactly-once guarantee does not hold. How can we fix this?

## 5. Fixing fault-tolerance for parallel processing

The solution to this problem is already mentioned in the docs and articles:

> Practically, one would either have to store the mapping between input partitions and transactional.ids in an external store, or have some static encoding of it.

<Kafka Streams uses TaskId -> transactional producer, is TaskId one for each partition?>

To avoid handling an external store, we will use a static encoding the same as in spring-kafka:

> the transactional.id is now the transactionIdPrefix appended with <group.id>.<topic>.<partition>

The drawback is that we will require separate transactional producer for each partition, but now we will be able to handle the last failure scenario correctly:

<msg seq diagram showing, first producer sends offsets, message, hangs (GC or network problems), consumer rebalance happens, second producer processes messages sends using the same transactional id and commits, first producer wakes up and is fenced off, the same transactional id -> zombie fencing>

Zombie fencing works properly now, thanks to using the same transactional id.

## 6. Summary

Handling transactions between producer sessions has its own nuances. One of our main decisions, as Kafka clients, is to pick right transactional ids for producer. Hopefully, the example given above makes it clear when should we use separate transactional id for each consumed partition. Only when consumer rebalance can result in assigning partitions to different workers those separate ids should be used.

