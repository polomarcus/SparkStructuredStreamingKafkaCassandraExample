# Kafka / Cassandra / Spark Structured Streaming Example
Stream the number of time **Drake is broadcasted** on each radio.
And also, see how easy is Spark Structured Streaming 2.2.0 to use using Spark SQL's Dataframe API

## Run the Project
### Step 1 - Start containers
Start the ZooKeeper, Kafka, Cassandra containers in detached mode (-d)
```
docker-compose up -d
```
### Step 2 - Create cassandra schema
```
# create Cassandra schema
docker-compose exec cassandra cqlsh -f /schema.cql;

# confirm schema
docker-compose exec cassandra cqlsh -e "DESCRIBE SCHEMA;"
```

### Step 3 - start spark structured streaming
```
sbt run
```

### Run the project after another time
As checkpointing enables us to process our data exactly once, we need to delete the checkpointing folders to re run our examples.
```
rm -rf checkpoint/
sbt run
```

## Requirements
* SBT
* [docker compose](https://github.com/docker/compose/releases/tag/1.17.1)
** Linux
```
curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
**  MacOS
```
brew install docker-compose
```

## Input data
Coming from radio stations stored inside a parquet file, the stream is emulated with ` .option("maxFilesPerTrigger", 1)` option.

The stream is after read to be sink into Kafka.
Then, Kafka to Cassandra

## Output data 
Stored inside Kafka and Cassandra for example only.
Cassandra's Sinks uses the [ForeachWriter](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.ForeachWriter) and also the [StreamSinkProvider](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.StreamSinkProvider) to compare both sinks.

One is using the **Datastax's Cassandra saveToCassandra** method. The other another method, messier (untyped), that uses CQL on a custom foreach loop.

From Spark's doc about batch duration:
> Trigger interval: Optionally, specify the trigger interval. If it is not specified, the system will check for availability of new data as soon as the previous processing has completed. If a trigger time is missed because the previous processing has not completed, then the system will attempt to trigger at the next trigger point, not immediately after the processing has completed.

### Kafka topic
One topic `test` with only one partition

#### List all topics
```
docker-compose exec kafka  \
  kafka-topics --list --zookeeper zookeeper:32181
```
#### Read all messages
```
docker-compose exec kafka  \
 kafka-console-consumer --bootstrap-server localhost:9092 --topic test --from-beginning
```
```
{"radio":"nova","artist":"Drake","title":"From Time","count":18}
{"radio":"nova","artist":"Drake","title":"4pm In Calabasas","count":1}
```

#### Send a message to be processed
```
docker-compose exec kafka  \
 kafka-console-producer --broker-list localhost:9092 --topic test

> {"radio":"skyrock","artist":"Drake","title":"Hold On We’Re Going Home","count":38}
```

### Cassandra Table
There are 3 tables. 2 used as sinks, and another to save kafka metadata.
Have a look to [schema.cql](https://github.com/polomarcus/Spark-Structured-Streaming-Examples/blob/e9afaf6691c860ffb4da64e311c6cec4cdee8968/src/conf/cassandra/schema.cql) for all the details.

```
 docker-compose exec cassandra cqlsh -e "SELECT * FROM structuredstreaming.radioOtherSink;"

 radio   | title                    | artist | count
---------+--------------------------+--------+-------
 skyrock |                Controlla |  Drake |     1
 skyrock |                Fake Love |  Drake |     9
 skyrock | Hold On We’Re Going Home |  Drake |    35
 skyrock |            Hotline Bling |  Drake |  1052
 skyrock |  Started From The Bottom |  Drake |    39
    nova |         4pm In Calabasas |  Drake |     1
    nova |             Feel No Ways |  Drake |     2
    nova |                From Time |  Drake |    34
    nova |                     Hype |  Drake |     2

```

### Kafka Metadata
@TODO Verify this below information. Cf this [SO comment](https://stackoverflow.com/questions/46153105/how-to-get-kafka-offsets-for-structured-query-for-manual-and-reliable-offset-man/46174353?noredirect=1#comment79536515_46174353)

When doing an application upgrade, we cannot use [checkpointing](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#recovering-from-failures-with-checkpointing), so we need to store our offset into a external datasource, here Cassandra is chosen.
Then, when starting our kafka source we need to use the option "StartingOffsets" with a json string like 
```
""" {"topicA":{"0":23,"1":-1},"topicB":{"0":-2}} """
```
Learn more [in the official Spark's doc for Kafka](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html#creating-a-kafka-source-for-batch-queries).

In the case, there is not Kafka's metadata stored inside Cassandra, **earliest** is used.

```
docker-compose exec cassandra cqlsh -e "SELECT * FROM structuredstreaming.kafkametadata;"
 partition | offset
-----------+--------
         0 |    171
```

## Useful links
* [Processing Data in Apache Kafka with Structured Streaming in Apache Spark 2.2](https://databricks.com/blog/2017/04/26/processing-data-in-apache-kafka-with-structured-streaming-in-apache-spark-2-2.html)
* https://databricks.com/blog/2017/04/04/real-time-end-to-end-integration-with-apache-kafka-in-apache-sparks-structured-streaming.html
* https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#using-foreach
* https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#output-modes

### Docker-compose
* [The last pickle's docker example](https://github.com/thelastpickle/docker-cassandra-bootstrap/blob/master/docker-compose.yml)
* [Confluence's Kafka docker compose](https://docs.confluent.io/current/installation/docker/docs/quickstart.html#getting-started-with-docker-compose)

## Inspired by
* https://github.com/ansrivas/spark-structured-streaming
* [Holden Karau's High Performance Spark](https://github.com/holdenk/spark-structured-streaming-ml/blob/master/src/main/scala/com/high-performance-spark-examples/structuredstreaming/CustomSink.scala#L66)
* [Jay Kreps blog articles](https://medium.com/@jaykreps/exactly-once-support-in-apache-kafka-55e1fdd0a35f)
