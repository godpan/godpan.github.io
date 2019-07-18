---
title: Kafka producer异步发送在某些情况会阻塞主线程
layout: post
guid: urn:uuid:85cd4b63-e9f0-4459-8ca7-fbdsdc2a2fad
tags:
  - kafka
---

最近发现一个Kafka producer异步发送在某些情况会阻塞主线程，后来在排查解决问题过程中发现这可以算是Kafka的一个说明不恰当的地方。

### 问题说明

在很多场景下我们会使用异步方式来发送Kafka的消息，会使用KafkaProducer中的以下方法：

```java
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {}
```
根据[文档的说明](https://github.com/apache/kafka/blob/1f2d230bfdaafb34c9be12a370ab2eb4d3016039/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L853)它是一个异步的发送方法，按道理不管如何它都不应该阻塞主线程，但实际中某些情况下会出现阻塞线程，比如broker未正确运行，topic未创建等情况，有些时候我们不需要对发送的结果做保证，但是如果出现阻塞的话，会影响其他业务逻辑。

### 问题出现点

从KafkaProducer send这个方法声明上看并没有什么问题，那么我们来看一下她的具体实现：

```java
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // intercept the record, which can be potentially modified; this method does not throw exceptions
    ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
}

/**
  * Implementation of asynchronously send a record to a topic.
  */
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        throwIfProducerClosed();
        // first make sure the metadata for the topic is available
        ClusterAndWaitTime clusterAndWaitTime;
        try {
            clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);  //出现问题的地方
        } catch (KafkaException e) {
            if (metadata.isClosed())
                throw new KafkaException("Producer closed while send in progress", e);
            throw e;
        }
        ...
    } catch (ApiException e) {
        ...
    }
}

private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long maxWaitMs) throws InterruptedException {
    // add topic to metadata topic list if it is not there already and reset expiry
    Cluster cluster = metadata.fetch();

    if (cluster.invalidTopics().contains(topic))
        throw new InvalidTopicException(topic);

    metadata.add(topic);

    Integer partitionsCount = cluster.partitionCountForTopic(topic);
    // Return cached metadata if we have it, and if the record's partition is either undefined
    // or within the known partition range
    if (partitionsCount != null && (partition == null || partition < partitionsCount))
        return new ClusterAndWaitTime(cluster, 0);

    long begin = time.milliseconds();
    long remainingWaitMs = maxWaitMs;
    long elapsed;
    
    //一直获取topic的元数据信息，直到获取成功，若获取时间超过maxWaitMs，则抛出异常
    do {
        if (partition != null) {
            log.trace("Requesting metadata update for partition {} of topic {}.", partition, topic);
        } else {
            log.trace("Requesting metadata update for topic {}.", topic);
        }
        metadata.add(topic);
        int version = metadata.requestUpdate();
        sender.wakeup();
        try {
            metadata.awaitUpdate(version, remainingWaitMs);
        } catch (TimeoutException ex) {
            // Rethrow with original maxWaitMs to prevent logging exception with remainingWaitMs
            throw new TimeoutException(
                    String.format("Topic %s not present in metadata after %d ms.",
                            topic, maxWaitMs));
        }
        cluster = metadata.fetch();
        elapsed = time.milliseconds() - begin;
        if (elapsed >= maxWaitMs) {  //判断1️⃣执行时间是否大于maxWaitMs
            throw new TimeoutException(partitionsCount == null ?
                    String.format("Topic %s not present in metadata after %d ms.",
                            topic, maxWaitMs) :
                    String.format("Partition %d of topic %s with partition count %d is not present in metadata after %d ms.",
                            partition, topic, partitionsCount, maxWaitMs));
        }
        metadata.maybeThrowException();
        remainingWaitMs = maxWaitMs - elapsed;
        partitionsCount = cluster.partitionCountForTopic(topic);
    } while (partitionsCount == null || (partition != null && partition >= partitionsCount));

    return new ClusterAndWaitTime(cluster, elapsed);
}

```
从它的实现我们可以看出，会导致线程阻塞的原因在于以下这个逻辑：

```java
private ClusterAndWaitTime waitOnMetadata(String topic, Integer partition, long maxWaitMs) throws InterruptedException
```
通过KafkaProducer 执行send的过程中需要先获取Metadata，而这是一个不断循环的操作，直到获取成功，或者抛出异常。

其实Kafka本意这么实现并没有问题，因为你要发送消息的前提就是能获取到border和topic的信息，问题在于这个send对外暴露的是Future的方法，但是内部实现却是有阻塞的，那么在有些时候没有考虑到这种情况，一旦出现border或者topic异常，将会阻塞系统线程，导致系统响应变慢，直到奔溃。

### 问题解决

其实解决这个问题很简单，就是单独创建几个线程用于消息发送，这样即使遇到意外情况，也只会阻塞几个线程，不会引起系统线程大面积阻塞，不可用，具体实现：

```scala
import java.util.concurrent.Callable
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors
import org.apache.kafka.clients.producer.{Callback, KafkaProducer, ProducerRecord, RecordMetadata}

class ProducerF[K,V](kafkaProducer: KafkaProducer[K,V]) {

  val executor: ExecutorService = Executors.newScheduledThreadPool(1)

  def sendAsync(producerRecord: ProducerRecord[K,V], callback: Callback) = {
    executor.submit(new Callable[RecordMetadata]() {
      def call = kafkaProducer.send(producerRecord, callback).get()
    })
  }
}
```
这是一种实现方式，当然你也可以自己维护一个Kafka版本，但这样或许有点麻烦，具体用什么方式根据自己场景来做选择。

使用例子:

```scala
import java.util.Properties
import org.apache.kafka.clients.producer.{KafkaProducer, ProducerRecord, RecordMetadata}

object FixExample extends App {

  val props = new Properties()
  props.put("max.block.ms", "3000")
  props.put("bootstrap.servers", "localhost:9092")
  props.put("client.id", "ProducerSendFixExample")
  props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
  props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")

  val producer = new KafkaProducer[String, String](props)
  val topic = "topic-trace-one"
  val userId = "godpan"
  val msg = "login wechat"
  val data = new ProducerRecord[String, String](topic, userId, msg)

  val startTime = System.currentTimeMillis()
  val producerF = new ProducerF(producer)
  producerF.sendAsync(data,(metadata: RecordMetadata, exception: Exception) => {
    println(s"[producerF-sendAsync] data producerRecord: ${data}, exception: ${exception}")
  })

  // 如果想要得到发送结果，可以线程等待4s
  // Thread.sleep(4000)
  System.exit(0)

}
```
相关代码已经传到github上了，地址：[kafka-send-async-bug-fix](https://github.com/godpan/kafka-send-async-bug-fix)。