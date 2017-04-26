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

## SBT

"Maven for scala"

[Download](http://www.scala-sbt.org)

### Setup

Set up a directory structure like this:

![sbt-structure](C:\Users\jeroen.schmidt\Documents\Notes\CheetSheets\Images/sbt-structure.PNG)

In the project folder: create an `assembly.sbt` file that contains

```
addSbtPlugin(com.eed3si9n"%"sbt-assembly"%"0.14.3")
```

==Note:== Check sbt documentation for updates (works for sbt 0.13.11)

**Create SBT Build File**  

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

This single line will run everything`spark-submit <jar file>`