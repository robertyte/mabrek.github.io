---
layout: post
title:  "Prototyping Long Term Time Series Storage with Kafka and Parquet"
---

_Another attempt to find better storage for time series data, this time it looks quite promising_

Last time I tried to switch from [Graphite](http://graphite.readthedocs.org/en/latest/) time series storage to [Cyanite/Cassandra]({{ site.url }}/blog/spark-cassandra-timeseries/) but the attempt failed and I stayed with [Whisper](http://graphite.readthedocs.org/en/latest/whisper.html) files. After struggling with keeping disk IOPS sane while ingesting hi-resolution performance data I ended up putting [Whisper](http://graphite.readthedocs.org/en/latest/whisper.html) files into [tmpfs](https://en.wikipedia.org/wiki/Tmpfs) and shortening data retention interval to just one day because my load tests usually don't last more than several hours. Then I export data into [R](http://r-project.org/) and do analysis. Large scale projects like [Gorilla(pdf)](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf) and [Atlas](http://techblog.netflix.com/2014/12/introducing-atlas-netflixs-primary.html) do similar things. They store recent data in RAM for dashboards and real-time analytics and then dump it to slow long term storage for offline analysis.

[Whisper file format](http://graphite.readthedocs.org/en/latest/whisper.html#database-format) is relatively good in terms of storage size (12 bytes per datapoint). It's columnar because it saves each metric in its own file. It contains redundant data because it saves a timestamp with each value and many values from different metrics share the same timestamp. There is no compression. I need to find something better than Whisper.

 [Gorilla(pdf)](http://www.vldb.org/pvldb/vol8/p1816-teller.pdf) paper inspired me to look into column storage formats with efficient encoding of repeated data. I decided to try [Parquet](https://parquet.apache.org/). Unfortunately floating point compression is [not there yet](https://github.com/Parquet/parquet-mr/issues/306) but my values have type `double` (as a side note [ORCFile](http://orc.apache.org/) [lacks it](https://issues.apache.org/jira/browse/ORC-15) too). Many metrics (counters) actually have integer type but there is no information about their types upfront.

To achieve good compression data needs to be written in large chunks so I needed something to buffer data. The way [Kafka](http://kafka.apache.org/) works with streaming writes and read offsets made me think that it's a good fit for storing data until it's picked up by periodical job. That job would start, read all the data available from the last read offset, compress and store it, and sleep until the next cycle.

My data comes in [Graphite plaintext protocol](http://graphite.readthedocs.org/en/latest/feeding-carbon.html#the-plaintext-protocol) which is quite verbose:

    <metric_name> <value> <timestamp>

Metric names and timestamps are repeating. Metrics are sent periodically at the same time so many lines share the same timestamp. The same set of metrics is being sent each time. Values for many metrics are not changing a lot over time (like disk usage) which makes it a good target for [dictionary](https://en.wikipedia.org/wiki/Dictionary_coder) or [delta encoding](https://en.wikipedia.org/wiki/Delta_encoding).

### Feeding graphite data into Kafka

I set up single node Kafka as described in [manual](http://kafka.apache.org/documentation.html#quickstart)

Feeding graphite data into Kafka turned out to be one-liner with [nc](http://netcat.sourceforge.net/) and [kafkacat](https://github.com/edenhill/kafkacat):

    nc -4l localhost 2003 | kafkacat -P -b localhost -t metrics -K ' '

Metric name is used as message key in kafka and `value timestamp` is a payload.

Then I started [collectd](https://collectd.org/) with [write graphite plugin](https://collectd.org/wiki/index.php/Plugin:Write_Graphite) pointing to localhost and reporting interval of 1 second. After several hours I got 1.8 Gb queue in kafka.

Dumping the data back into text format is a one-liner too:

    kafka-console-consumer.sh --zookeeper localhost:2181 --topic metrics \
      --from-beginning --property print.key=true

Text file had size of 1.4Gb which means kafka has some overhead for storing uncompressed data. There were ~19000000 lines in the file.

### Fetching data from Kafka

I needed to handle read offsets manually so I chose [SimpleConsumer](http://kafka.apache.org/documentation.html#simpleconsumerapi). Its API turned out to be quite confusing and not that simple. It doesn't talk to Zookeeper and allows to specify offsets. Handling all corner cases requires lots of [code](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example) but simple prototype turned out to be quite short in Scala:

    val consumer = new SimpleConsumer("localhost", 9092, 5000,
        BlockingChannel.UseDefaultBufferSize, name)
    val fetchRequest = new FetchRequestBuilder().clientId(name)
        .addFetch(topic, partition, offset, fetchSize).build()
    val fetchResponse = consumer.fetch(fetchRequest)
    val messages = fetchResponse.messageSet(topic, partition)

It just reads all messages from specified `offset` up to `fetchSize`.

### Saving data into Parquet

[Parquet](https://parquet.apache.org/) API documentation doesn't seem to be published anywhere. Javadoc for [org.apache.parquet.schema.Types](https://github.com/apache/parquet-mr/blob/master/parquet-column/src/main/java/org/apache/parquet/schema/Types.java#L30) contains several schema examples. Writing local files from standalone application is not described anywhere but `parquet-benchmarks` module contains class [org.apache.parquet.benchmarks.DataGenerator](https://github.com/apache/parquet-mr/blob/master/parquet-benchmarks/src/main/java/org/apache/parquet/benchmarks/DataGenerator.java#L68) which writes several variants of local files. It depends on Hadoop classes so you'll need it as a project dependency ([issue](https://github.com/Parquet/parquet-mr/issues/305))

I decided to use 'wide' schema when each metric has its own column to make delta encoding possible and a single column for timestamps (like [xts](https://cran.r-project.org/web/packages/xts/index.html) does for multivariate series):

    val types = mutable.Set[Type]()
    ...
    // for each message collect unique keys as types
    types += Types.optional(DOUBLE).named(key)
    ...
    val schema = new MessageType("GraphiteLine",
        (types + Types.required(INT64).named("timestamp")).toList) 

Boilerplate to create [ParquetWriter](https://github.com/apache/parquet-mr/blob/master/parquet-hadoop/src/main/java/org/apache/parquet/hadoop/ParquetWriter.java) object:

    val configuration = new Configuration
    GroupWriteSupport.setSchema(schema, configuration)
    val gf = new SimpleGroupFactory(schema)
    val outFile = new Path("data-file.parquet")
    val writer = new ParquetWriter[Group](outFile, 
        new GroupWriteSupport, UNCOMPRESSED, DEFAULT_BLOCK_SIZE, 
        DEFAULT_PAGE_SIZE, 512, true, false, PARQUET_2_0, 
        configuration)

For each unique timestamp a row (called group in Parquet) is added which contains values for all metrics (columns) at that time:

    for (timestamp <- timestamps) {
        val group = gf.newGroup().append("timestamp", timestamp)
            for ((metric, column) <- columns) {
                column.get(timestamp).foreach(group.append(metric, _))
            }
        writer.write(group)
    }
    writer.close()

And that's it.

### Effect of compression

Enabling gzip compression in Parquet reduced file size 3 times compared to uncompressed. The result took 12Mb for ~19000000 input lines which is quite impressive. Storing the same data in whisper format would take at least 230Mb (actually more because it reserves space for whole retention interval).

I tried enabling Snappy compression for Kafka publisher:

    kafkacat -P -b localhost -t metricz -K ' ' -z snappy < metrics.txt

and got ~ 500Mb queue size for 1.4Gb original data. The only problem is that log compaction is not yet compatible with compressed topics.

The result looks quite good: temporal buffer in Kafka needs 1/3 size of original data  and long term storage takes ~ 0.6 bytes per datapoint (while whisper takes 12 bytes).

### Open questions

Can Parquet handle very wide schema with 100k columns and more?

How to guess effective metric type (double or integer) from incoming graphite data?

How to get data from Parquet into R without Spark? ([issue](https://github.com/Parquet/parquet-format/issues/72))

### Source code

Available on github [kafka-timeseries](https://github.com/mabrek/kafka-timeseries) with build instructions. Actually it's just a [single Scala file](https://github.com/mabrek/kafka-timeseries/blob/master/src/main/scala/KafkaTimeseries.scala) with 100 lines of code 20 of which are import statements.