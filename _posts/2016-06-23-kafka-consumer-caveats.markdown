---
layout: post
title:  "Caveats for working with Kafka consumers"
date:   2016-06-23 12:14:07 +0200
categories: kafka scala
author: Jasper A. Visser
---

My team at [Portavita][portavita] has been working a lot recently with an open-source message broker called [Apache Kafka][kafka]. It describes itself as a distributed commit log system. To understand what that means I recommend you read [this excellent blog post of Jay Kreps at linkedin][the-log].

In this post I give a simple example of a consumer that is constantly processing messages, and look at some of the conditions in which it can fail (and what you can do about it).

It is assumed that you have a basic understanding of what kafka does. More specifically you should know what a topic, consumer, offset & broker are, and how the broker stores consumer positions. The [intro page][kafka-intro] describes all of this very nicely.

Very succinctly:

- a *topic* is a partitioned log file 
- a *producer* appends data to a topic
- a *consumer* reads from a topic sequentially
- a *broker* stores the topic data, and orchestrates access of producers and consumers
- a *cluster* is a collection of brokers that replicate and partition logs
- a *consumer group* is a collection of consumers that share the task of processing messages from a topic between themselves (each consumer handles some partitions of the topic)

#### Example of a simple consumer loop

{% highlight scala %}
// Requires: org.apache.kafka % kafka-clients % 0.9.0.1

val consumer = new KafkaConsumer[String, String] {
  val deserializer = classOf[StringDeserializer].getCanonicalName
  val properties = new Properties
  properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka-broker:9092")
  properties.put(ConsumerConfig.GROUP_ID_CONFIG, "example")
  properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, deserializer)
  properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, deserializer)
  properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false")
  properties.put(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG, "30000")
  properties
}

val topic = "example"
consumer.subscribe(List(topic).asJava)

def process(msgs: Iterable[String]) = ???

while (true) {
  val msgs = consumer.poll(timeout = 100).asScala.map(_.value)
  process(msgs)
  consumer.commitSync()
}
{% endhighlight %}

After some mostly-boilerplate setup, this code does the following:

- Poll the kafka broker to obtain new messages;
- Process the messages (do whatever your business logic requires);
- Commit the update consumer position back to the kafka broker. 

Note that in this example the consumer position is only committed kafka *after* the messages are processed. If the application running this loop dies, it will start consuming at the last committed consumer position. Effectively, this guarantees you that you will process each message *at least once*. It can very well happen that the same message is processed multiple times, e.g. when the application is either unable to commit the consumer positions or when it is unable to ascertain whether the processing was completely successful. It is therefore important that your processing is [idempotent][idempotence].

Some alternatives are:

- commit before processing;
- automatically commit every N milliseconds;
- commit asynchronously (don't wait until the commit is acknowledged).

The first solution offers *at most once* delivery guarantee. The other two solutions are non-blocking, and offer neither *at least once* nor *at most once*.

It is important for our change data capture system that no messages are lost, so the rest of the post will look at the *at least once* delivery example only. The example is not production-ready, of course. However, it can already illustrate some of the problems we ran into. 

#### consumer.poll() has no timeout
Okay, this is a bit of a confusing statement. Especially since the poll() function actually has a parameter called *timeout*. It's important to realize that this timeout only applies to part of what the poll() function does internally.

The timeout parameter is the number of milliseconds that the network client inside the kafka consumer will wait for sufficient data to arrive from the network to fill the buffer. If no data is sent to the consumer, the poll() function will take at least this long. If data is available for the consumer, poll() might be shorter.

However, before it gets to that part of the poll() function, the consumer will also do a check to ensure that the broker is available (specifically the broker that acts as the coordinator for the consumer group). That part does *not* respect the timeout. It will try infinitely long to fetch metadata from the cluster. 

See also [KAFKA-1894][kafka-1894].

In effect, if the cluster is unavailable and the consumer attempts to fetch metadata, the consumer will hang until the cluster is available.

Our solution was to wrap the poll() call in a future and limit the time to wait on its completion.

{% highlight scala %}
import scala.concurrent.ExecutionContext.Implicits.global
// ...
val pollTimeout = 30 seconds
// ...
while (true) {
  val msgs = Await.result(Future(underlyingConsumer.poll()), pollTimeout)
  // ...
}
{% endhighlight %}

#### consumer.commitSync() must happen within 30 seconds of consumer.poll()
If the processing of messages is expensive (e.g. complex calculations, or long blocking I/O), you may run into a *CommitFailedException*. The reason for this is that the consumer is expected to send a heartbeat to the broker every so often. This heartbeat informs the broker that the consumer is still alive. When the heartbeat doesn't arrive in time, the broker will mark the consumer as dead and kick it from the consumer group. The time is defined by the *session.timeout.ms* configuration of the broker (default is 30 seconds).

Both the poll() and commitSync() functions send this heartbeat. However, if the time between the two function calls is 30 seconds, then by the time commitSync() is called, the broker will already have marked the consumer as dead. As a result you get a *CommitFailedException*.

Increasing *session.timeout.ms* is a possibility that comes with its own drawbacks. See below.

Our solution was to simply live with the occasional exception, catch it and retry.

{% highlight scala %}
// ...
while (true) {
  Try {
    val msgs = consumer.poll(timeout = 100).asScala.map(_.value)
    process(msgs)
    consumer.commitSync()
  } match {
    case Success(_) =>
    case Failure(e: CommitFailedException) =>
      log.error(s"Commit failed on $topic!")
    case Failure(e) => throw e
  }
}
{% endhighlight %}

#### If you don't call consumer.close(), you have to wait
The consumer.close() function sends a signal to the broker that the consumer has left the consumer group gracefully. If you forget to call this function, the broker will consider the consumer to be alive for at most *session.timeout.ms*.

You will notice that in the example above, there is no graceful way to shutdown the application. If this application is killed and restarted immediately, the broker has no idea that the old consumer is dead. The new consumer will happily start up, but it will not be assigned any partitions of the topic until the session timeout of the old consumer has expired.

The obvious solution is to always try to gracefully shutdown the application and call consumer.close(). This is a little difficult to implement nicely in a single-threaded example. We wrap [akka][akka] actors around the consumers, which close the consumer in postStop().

You cannot always guarantee a graceful shutdown, so in general it is best not to put *session.timeout.ms* very high.

#### Conclusion
These are some of the lessons we learned working with kafka consumers. In all we are extremely happy with the choice to use kafka. It offers a very simple model for dealing with streams of data, and provides great performance with a relatively easy way of scaling.

[akka]: http://akka.io/
[idempotence]: https://en.wikipedia.org/wiki/Idempotence
[kafka]: http://kafka.apache.org/
[kafka-intro]: http://kafka.apache.org/documentation.html#introduction
[kafka-1894]: https://issues.apache.org/jira/browse/KAFKA-1894
[portavita]: https://www.portavita.com/
[the-log]: https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying
