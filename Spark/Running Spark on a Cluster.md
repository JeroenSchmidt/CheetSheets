---
typora-copy-images-to: Images
---

# Running Spark on a Cluster

[TOC]

## Checklist

- make sure no hardcoded paths - HDFS, S3, etc.
- Package Scala project into a JAR file (use export in IDE)
- use `spark-submit`

```powershell
spark-submit -class <class object that contains your main function>
--jars <paths to any dependencies>
--files <files yo uwant placed alongside your applcation> 
<your jar file>
```

- If executers fail to start: (usually) adjust memory 

## SBT

*Version Control* - "Maven for scala"

[Download](http://www.scala-sbt.org)

### Setup

Set up a directory structure like this:

![sbt-structure](.\Images/sbt-structure.PNG)

- In the project folder: create an `assembly.sbt` file that contains

```
addSbtPlugin(com.eed3si9n"%"sbt-assembly"%"0.14.3")
```

==Note:== Check sbt documentation for updates (the above works for sbt 0.13.11)

- **Create SBT Build File**  

*Example*

At the root, create a SBT build file (along side the src and project directories)

```
name := "<name>"

version := "1.0"

organization := "com.sundogsoftware" 

scalaVersion := "2.10.6"

libraryDependencies ++= Seq(
"org.apache.spark" %% "spark-core" %
"1.6.1" % "provided"
)
```

==Note:== "provided" -> indicates that the cluster has the dependencies the jar needs 

*Example* (adding dependencies): `"org.apache.spark" %% "spark-streaming-kafka" % "1.6.1"`

### Run SBT

Run from root: `sbt assembly`

JAR will be packaged in `target/scala-2.10` (version of scala built against) 

### Submit JAR

This single line will run everything `spark-submit <jar file>`

## Spark-Submit Parameters

**Spark Structure Recap**

![spark_structure](.\Images/spark_structure.PNG)

==NOTE:== Normally most clusters have environment parameters defined in a configuration file -> we don't need to specify within the spark script explicitly what the master is going to be, what the cluster manager is, etc. (But we can still specify it if needed)

*Configuration Hierarchy*

1. Script
2. ​
3. Spark Cluster Config File

```
#spark-submit configuration

-- master
	- YARN - running on a yarn/hadoop cluster
	- hostname:port - for connecting to a master on a spark standalone cluster
	- mesos://masternote:port
	- A master in your SparkConf will override this
--num-executers
	- Must set explicity with YARN, only 2 by default
--executor-memory
	- Make sure you don't try to use more memory then you have
-total-executor-cores
```

## Amazon EC2 - Elastic MapReduce 

Pre-configured cluster -> `EMR Cluster`

***Security:*** Under master node security, make sure ssh port is open. 

==Note:== AWS has commands that can be used to interface between different services, like `S3` storage

```scala
#Accessing data in s3n is easy
s3n://URL to file path
```

## Partitioning

```scala
rdd.partitionBy(new HashPartitioner(<partition number>))
```

Spark doesn't always know how to distribute data across the cluster. This can create a lot of overhead with expensive shuffle operations. 

==*Recall*:==
Spark brakes down *DAG* into stages where data is shuffled. Each *Stages* is broken into *Tasks* which are distributed to the executers.  

The `partition` operations specifies how many *tasks* we want an operation to be broken up into. The operation result will preserve the partitions. This reduces the number of shuffles $\because$ the following operations can continue working on the existing partitions without shuffling.

==The most benefit:== is realized when several operations can be chained together to use the same partitions. 

==Rule of thumb:== Make sure you have as many partitions as executers. 

### *Operations that benefit from partitions*:

```scala
join()
cogroup()
groupWith
leftOuterJoin()
rightOuterJoin()
groupByKey()
reduceByKey
combineByKey()
lookup()
```

## Debugging 

`lossing hearbeats` -> usually an issue with resources -> solution: more hardware or optimize partitions used.

### Accessing Logs

*Spark standalone:* webUI

*YARN:*

```
yarn -logs -applicationID <app ID>
```

### Things to keep in mind

- Use broadcast variables to share data outside RDDs
  - If we have code that needs to access something on local disk, it will fail if an executer tries to run it as it will only exist on the driver unless setup otherwise. 
- Using packages that arent pre-loaded (might be loaded on driver but not executer)
  - use SBT to package everything
  - submit the libraries with spark-submit