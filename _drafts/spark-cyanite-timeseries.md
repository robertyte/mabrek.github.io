---
layout: post
title:  "Spark vs. Cassandra (Cyanite) for Timeseries Data"
---

_Unsuccessful attempt to scale metrics processing by switching from Graphite/R to Spark/Cassandra/Cyanite_

My typical workflow is to run load tests until the system hits some limit (or observe production failure), extract data from Graphite (whisper files produced by carbon), import it into R, and then run a battery of statistical tests and visual explorations on data to find the root cause of the problem and fix it. There is a strong need for hi-resolution data in load tests (because systems fail really fast under load) but the tools I use are not really good at ingesting and processing data with granularity finer than 10 seconds. It's been a long time since I first tried to escape 'Graphite world' but all attempts failed so far and here I'll tell about another attempt.

Graphite world (carbon-relay, carbon, graphite-web) has a lot of issues (TODO) and is moving slowly (TODO). Feeding metrics with 1s resolution to carbon might be feasible for one hosts but doesn't work at practical scale. This could be done but needs more hardware than monitored hosts.

R has a great number of timeseries related libraries (TODO cran task) but it runs on a single core and is limited by available memory on a single box. There are packages that allow R to fork several computation processes and move data between them but it's still limited to a single box.

Cassandra is said to be good at timeseries data. Spark is said to be fast at distributed data processing and there is spark-cassandra connector. [Cyanite](https://github.com/pyr/cyanite/) is a simple app written in closure that ingests metrics in the same format as Carbon does and saves them into simple table in Cassandra. There is a [graphite-cyanite](https://github.com/brutasse/graphite-cyanite) project that allows to run [graphite-api](https://github.com/brutasse/graphite-api) on top of Cyanite and use all graphite-compatible dashboards like [Grafana](http://grafana.org/).

So I wired them all together and tested with one collectd gathering data with 10s interval. It worked OK so I proceeded by adding more hosts and switching to 1s interval. Cyanite ate all cpu and failed. Profiling showed that the default in-memory index of metric names is the bottleneck so I installed Elasticsearch and switched Cyanite path store to es-native. This time both Cyanite and Elasticsearch ate all cpu and failed again. It turns out that for every metric update Cyanite tries to send metric name to Elasticsearch [(cyanite/#96)](https://github.com/pyr/cyanite/issues/96) which amounts to thousands requests per second. It's mostly a time waste because metric names don't change every time. I ended up disabling metric name indexing and data flowed to Cassandra with little complaints [(cyanite/#73)](https://github.com/pyr/cyanite/issues/73).

[Spark](http://spark.apache.org/) version was 1.3.1. Released version of [spark-cassandra-connector](https://github.com/datastax/spark-cassandra-connector) supported only Spark 1.2 but the master branch was able to work with Spark 1.3.1 (see [SPARKC-98](https://datastax-oss.atlassian.net/browse/SPARKC-98) for more details). There are a lot of changes in internal Spark APIs between 1.2 and 1.3 so DataFrames and SQL was not working with connector ([SPARKC-112](https://datastax-oss.atlassian.net/browse/SPARKC-112)) but RDD operations were quite usable.

First problem encountered was poor locality of spark jobs caused by missing reverse DNS for my virtual machines where spark and cassandra were running. There is a task [SPARK-5113](https://issues.apache.org/jira/browse/SPARK-5113) to help with that but in the mean time if your reverse DNS is broken (as it usually is in intranets) then it's better to force spark to use only ip addresses for everything (set `SPARK_LOCAL_IP`, `SPARK_PUBLIC_DNS`, `SPARK_LOCAL_HOSTNAME` to an ip address in `spark-env.sh`)

Then there were not enough job parallelism. Number of partitions was equal to number of cassandra hosts (replication factor was 1). Data in cyanite table is partitioned by metric name and there were about 1500 unique names. Default connector setting `spark.cassandra.input.split.size` was too large and I lowered it to `5000`.

Then I noticed quite high number of context switches and cpu usage of cassandra process. There were too many queries executed by connector so I increased `spark.cassandra.input.page.row.size` to `10000`. Each datapoint is a single number and at 1s resolution it equals to about 3 hours of data for a single metric.

These parameters (split size and page row size) are better to set per each table individually because number of partitions, cells per row, cell size are different. Setting them manually in Scala seems ugly because you can't provide one implicit for ReadConf and not provide for other parameters:

`val rdd = sc.cassandraTable("metric", "metric")(CassandraConnector(sc.getConf), ReadConf(5000, 10000), implicitly[ClassTag[CassandraRow]], implicitly[RowReaderFactory[CassandraRow]], implicitly[ValidRDDType[CassandraRow]])`

Job speed increased but cassandra cpu usage remained quite high. `perf top`(TODO) profiler showed LZ4 decompression routines so I tried to disable compression for sstables. Compression level was about 0.3 so increasing data size 3 times was not a big deal. It didn't help at all and cassandra was still hogging cpu. JMC(TODO) profiler showed that cassandra was reading sstables (quite expected behaviour). That prompted me to check if the storage format was good enough for the data.

It took about 100 bytes to store 1 timestamp and 1 value of type double without compression and about 30 bytes with compression enabled. It's a lot compared to whisper format which takes 12 bytes for that. For each data cell cassandra stores column name and write timestamp. Column names are identical for the majority of cells stored and compression takes care of that on disk but memory overhead is still high (see discussion at [CASSANDRA-4175](https://issues.apache.org/jira/browse/CASSANDRA-4175)). Saving another timestamp for data cell which has a timestamp inside seems redundant. While it's possible to set cassandra write time by `insert ... using timestamp ...` this internal timestamp can't be used for sorting and filtering data.

After all of that I arrived to the point when reading 51m datapoints took about 40 seconds on 6 cores. That data was produced by 4 hosts with collectd reporting with 1 second interval for about 10 hours. Number of unique metric names was 1500. Data size on disk with compression enabled was 1.3Gb total.

With the data ready to be processed in RDD I stuck on what's next. Usually I'd run positive-only-diff on known counters, filter out flat metrics, calculate different percentilles on moving windows and run Tukey Method to find spikes. But Java/Scala toolbox for time series is almost empty. There is nothing like xts, ggplot2, TSclust, forecast (TODO). The only thing that I found is a recently started [spark-timeseries](https://github.com/cloudera/spark-timeseries)

There are tools beyong Java/Scala at the price of cpu cycles spent on another round of data serialization/deserialization. PySpark could give access to Pandas and IPython visualizations (with [python 3 support](https://issues.apache.org/jira/browse/SPARK-4897) in upcoming Spark 1.4). SparkR is [going to be released](https://issues.apache.org/jira/browse/SPARK-5654) in Spark 1.4 too.

TTL and tombstones