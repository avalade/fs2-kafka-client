# Simple Kafka client for functional streams for scala
This is a tiny fs2 wrapper around the Kafka java client.

[![CircleCI](https://circleci.com/gh/ovotech/fs2-kafka-client.svg?style=svg)](https://circleci.com/gh/ovotech/fs2-kafka-client)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/7a33158494af4a17a682106fef376625)](https://www.codacy.com/app/filippo-deluca/fs2-kafka-client?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=ovotech/fs2-kafka-client&amp;utm_campaign=Badge_Grade)
[![Download](https://api.bintray.com/packages/ovotech/maven/fs2-kafka-client/images/download.svg) ](https://bintray.com/ovotech/maven/fs2-kafka-client/_latestVersion)

## Installation
To get started with SBT, simply add the following lines to your build.sbt file.

```sbtshell
resolvers += "Ovotech" at "https://dl.bintray.com/ovotech/maven"

libraryDependencies += "com.ovoenergy" %% "fs2-kafka-client" % "<latest version>"
```

## Consuming
To consume records without committing or with auto-commit:
```tut:silent
import com.ovoenergy.fs2.kafka._
import scala.concurrent.duration._
import org.apache.kafka.common.serialization._
import org.apache.kafka.clients.consumer._
import cats.effect.IO

val settings = ConsumerSettings(
    pollTimeout = 250.milliseconds,
    maxParallelism = 4,
    nativeSettings = Map(
      ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG -> "false",
      ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG -> s"localhost:9092",
      ConsumerConfig.GROUP_ID_CONFIG -> "my-group-id",
      ConsumerConfig.AUTO_OFFSET_RESET_CONFIG -> "earliest"
    )
)

val consumedRecords = consume[IO](
  TopicSubscription(Set("my-topic")),
  new StringDeserializer,
  new StringDeserializer,
  settings
).take(1000).compile.toVector

```

To consume records, apply a function for each record and commit:
```tut:silent
import com.ovoenergy.fs2.kafka._
import scala.concurrent.duration._
import org.apache.kafka.common.serialization._
import org.apache.kafka.clients.consumer._
import cats.effect.IO
import scala.concurrent.ExecutionContext

implicit val ec = ExecutionContext.global

val settings = ConsumerSettings(
    pollTimeout = 250.milliseconds,
    maxParallelism = 4,
    nativeSettings = Map(
      ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG -> "false",
      ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG -> s"localhost:9092",
      ConsumerConfig.GROUP_ID_CONFIG -> "my-group-id",
      ConsumerConfig.AUTO_OFFSET_RESET_CONFIG -> "earliest"
    )
)

def processRecord(r: ConsumerRecord[String, String]): IO[Int] = IO {
  // Apply some side-effect
  1
}

val ints = consumeProcessAndCommit[IO](
    TopicSubscription(Set("my-topic")),
    new StringDeserializer,
    new StringDeserializer,
    settings
)(processRecord).take(1000).compile.toVector
```

The record processing order is guaranteed within the same partition, while records from different partitions are processed
in parallel up to the parallelism set into the `ConsumerSettings`.

To consume batch of records, and process records with fs2.Pipe then commit:
```tut:silent
import com.ovoenergy.fs2.kafka._
import scala.concurrent.duration._
import org.apache.kafka.common.serialization._
import org.apache.kafka.clients.consumer._
import cats.effect.IO
import fs2.Pipe
import scala.concurrent.ExecutionContext

implicit val ec = ExecutionContext.global

val settings = ConsumerSettings(
    pollTimeout = 250.milliseconds,
    maxParallelism = 4,
    nativeSettings = Map(
      ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG -> "false",
      ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG -> s"localhost:9092",
      ConsumerConfig.GROUP_ID_CONFIG -> "my-group-id",
      ConsumerConfig.AUTO_OFFSET_RESET_CONFIG -> "earliest"
    )
)

def pipe[K, V]: Pipe[IO, ConsumerRecord[K, V], Long] = {
  _.evalMap(c => IO(c.offset()))
    .filter(_ % 10 == 0)
    .evalMap(o => IO(o * 3))
}

consumeProcessBatchWithPipeAndCommit[IO](
    TopicSubscription(Set("my-topic")),
    new StringDeserializer,
    new StringDeserializer,
    settings
)(pipe).take(1).compile.drain
```

## Producing
The producer is available as effectfull function:

```tut:silent
import com.ovoenergy.fs2.kafka._
import scala.concurrent.duration._
import org.apache.kafka.common.serialization._
import org.apache.kafka.clients.producer._
import cats.effect.IO
import scala.concurrent.ExecutionContext
import java.util.Properties

implicit val ec = ExecutionContext.global

val topic = "my-topic"
val key = "my-key"
val value = "my-value"

val props = new Properties()
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092")
props.put(ProducerConfig.CLIENT_ID_CONFIG, "KafkaExampleProducer")
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, classOf[StringSerializer].getName)
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, classOf[StringSerializer].getName)

val producer: Producer[String, String] = new KafkaProducer[String, String](props)

produceRecord[IO](producer, new ProducerRecord[String, String](topic, key, value))
```

The producer itself is available trough a `Stream`:
```tut:silent
import com.ovoenergy.fs2.kafka._
import org.apache.kafka.clients.producer._
import org.apache.kafka.common.serialization._
import cats.effect.IO
import fs2._

val producerSettings = ProducerSettings(
  Map(
    ProducerConfig.BOOTSTRAP_SERVERS_CONFIG -> s"localhost:9092",
    ProducerConfig.ACKS_CONFIG -> "all",
    ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION -> "1"
  )
)

val topic = "my-topic"

producerStream[IO](
  producerSettings,
  new StringSerializer,
  new StringSerializer
).flatMap { producer =>

  val key = "my-key"
  val value = "my-value"

  Stream.eval {
    produceRecord[IO](producer, new ProducerRecord[String, String](topic, key, value))
  }
}
```



