---
typora-copy-images-to: Images\Spark Streaming
---

# Spark Streaming

[TOC]

## Spark Revision

***TL:DR;*** User defines flow of what happens to data -> Spark optimizes work flow of the job with the DAG Engine which it is aware of. Data is kept in memory, very fast as a result.

### Structure

![spark_structure](C:\Users\jeroen.schmidt\Documents\Notes\CheetSheets\Spark\Images\Spark Streaming/spark_structure.PNG)

### Spark Libraries

![spark_libraries](C:\Users\jeroen.schmidt\Documents\Notes\CheetSheets\Spark\Images\Spark Streaming/spark_libraries.PNG) 

### Componenets

#### Spark Context

***TL;DR;***

- It is created by the *driver program*.
- *RDDs* are created within the Spark Context.

```scala
// Sprak Core context

// Set up a SparkContext named <context_name> that runs locally using
// all available cores.
val conf = new SparkConf().setAppName("<context_name>")
conf.setMaster("local[*]")
val sc = new SparkContext(conf)
```

==In Spark Streaming== we define a streaming context.

```scala
// the following streaming context is for a local machine using all its cores "local[*]" with a 1 second batch size
val ssc = new StreamingContext("local[*]", "LogAlarmer", Seconds(1))
```

#### RDD - Resilient Distributed Dataset

***TL;DR*** Allows us to refer to a distributed entity as if it is a single entity which would normally not be possible to contain on a single machine.

```scala
// Creating RDDS from file systems

sc.textFile("file:/// <directory>")
//or 
"s3n:// <directory>"
"hdfs:// <directory>"
```

```scala
// RDDS from HIVE
hiveCtx = HiveContext(sc)
rows = hiveCtx.sql("<sql quary>")
```

We can also create RDDs from:

`JDBC`, `Cassandra`, `HBase`, `Elasticsearch`, `JSON`, `CSV`, `Sequence files`, etc...

*Accessing elements in RDD Tuple*

```scala
rdd._x
// where x is the position of element in tuple
```

##### Transforming RDDs

*TL;DR* Transformation construct the DAG - nothing is run till an action is run

- `map` : n to n transformation
- `flatmap`: n to m transformation 
- `filter`
- `distinct`
- `sample`
- `union, intersection, subtract, cartesian`

```scala
//Example
val input = sc.parallelize(List(1,2,3,4)) //Creating a distributed RDD
val result = input.map(x => x * x)
> (1,4,9,16)
```

##### RDD Action

*TL;DR* "Lazy Evaluation": An action that is performed on an RDD will execute the DAG map relevant to that RDD that has been constructed through the transformations defined. 

***Examples:***

- `collect`
- `count`
- `countByValue`
- `take`
- `top`
- `reduce`

## Spark Streaming Overview

> Analyze data streams in real time. No need for batch jobs.

### High Level View

![sprak_stream_view](C:\Users\jeroen.schmidt\Documents\Notes\CheetSheets\Spark\Images\Spark Streaming/sprak_stream_view.PNG)

Multiple streams supported. The data from these streams are broken up into "micro" batches which can be processed on the distributed system. 

*Example Streams:*

- Kafka
- Flume
- Kinesis

### DStreams - Discretized Streams 

- They generates the *RDD* for each ***batch interval/time step***. 
- *DStreams* are objects that wrap around the batch *RDDs*. 
- It can produce output for each time step. 
- They can be transformed and acted on in the same way as *RDDs* 
- The underlying *RDD's* can be accessed directly. 

***Stateless Transformation***

- `Map`
- `FlatMap`
- `Filter`
- `reduceByKey`

==Important== DStreams can keep track of *Statful data*. Long lived states can be maintained on DStreams. i.e "running totals"

### Intervals

- **Batch Interval:** how often data is captured into a DStream
- **Slide Interval:** how often a windowed transformation is computed 
- **Window Interval:** how far back in time the windowed transformation goes. Allows us to computer results across a longer time period then our batch interval

***Illustration***

Say we have a 1 second batch intervals and we want to process on data that extends beyond that single batch interval. We could then use a window that slides with time. In the illustration below that would be a 4 second window. Computations on the window interval are executed for each *slide interval* of 2 seconds.

![windows](C:\Users\jeroen.schmidt\Documents\Notes\CheetSheets\Spark\Images\Spark Streaming/windows.png)

***Use Case:***  Top sellers in the past hour (we could process data every second and maintain a one hour window)

***Example:***

```scala
//batch interval defined in stream context -> 1 second
val ssc = new StreamContext("local[*]", "<context name>", Seconds(1))

// using reduceByWindow we define the window interval and the slide interval respectivly
val rdd2 = rdd1.reduceByKeyAndWindow((x,y) => x + y , (x,y) => x-y, Seconds(4), Seconds(2))
// x-y is the inverse of the reduce function we are applying. Not neccesary but helps optimise the job
```

### Fault Tolerance

- Incoming data is replicated to at least 2 worker nodes
- Checkpoints 
  - Store state into filesystem (HDFS, S3) which can be recovered incase of failure
  - ==NB== required if suing *stateful data*
  - code: `ssc.checkpoint`

#### Receiver Failure

Receiver that gets its data pushed to it will loss data that is sent to it if it fails. 
This kind of failure extends to *Kafka* and *Flume* in a *push configuration* into spark configuration.

Receivers that can pull from data source can recover from a failure as they can re-request the data it attempted to pull. 
Some examples: *pull based flume*, *directly-consumed kafka* which interface with a replicated, reliable data source. 

#### Driver Failure

Single point of failure.

***Solution:*** 

Instead of getting the context directly, we can store the context else where which allows for context recovery if the driver fails.

```scala
// Use context returned by:
StreamingContext.getOrCreate(<checkpoint dir>,<function that creates new streaming context>)

//This will either return a context in the case where a driver unexpectatly failed or create a new context
```

==Note== If the driver fails, we still need an automated tool to restart the driver script. *ZooKeper* and *Sparks* built in cluster manager can do this. 
***Use:*** `-supervise` on `spark-submit` 

## Aside Code

### Running Totals

```scala
// The running total will persist across all instances of the action being executed on the different RDDs simultaniusly 
// There is a more elegant solution using spark libraries but for this purpose Java's 
// AtomicLong class is enough and ensure that the counter is thread-safe.

var runningTotal = new AtomicLong(0)

rdd1.foreachRDD((rdd2, time) => {
	runningTotal.getAndAdd(rdd2)
}//i.e. there are simultaniuse agreegations of of rdd2 acorss multiple executers all being feed into runningTotal          
```