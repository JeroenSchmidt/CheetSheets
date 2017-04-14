# Scala & Spark Cheat Sheet

Syntax key: 

| Syntax           |                 Meaning                  |
| ---------------- | :--------------------------------------: |
| $< description>$ | Replace $<>$ and its contents with the relevant code |

 

[TOC]

## Spark Core

### Creating a function that deals with flatmap

```scala
def <functionName>(<input>:<DataType>):Option[<DataType_1>]={
  if (<logic statment>){
    //the flatmap will ignore None when it is returned
    return None 
  }else{
    //flatmap will treat returnVar the same way map would 
    return Some(<returnVar:DataType_1>) 
  }
}
```

### Case Statements / Pattern Matching 

```scala
//match can work with mixed data types like tuples
val y = x match {
    case "one" => 1
    case 2 => "two"
  	case z if (0 <= z && x < 10) // guards can be used to consider ranges
    case _ => "many" // match anything that doesnt match with the first 3
  }
```

## DataFrame & DataSet & RDD

[Overview](https://databricks.com/blog/2016/07/14/a-tale-of-three-apache-spark-apis-rdds-dataframes-and-datasets.html)

### Converting from DataFrame,DataSet[Row] to DataSet[DistinctObject]

You could express each JSON entry as *DeviceIoTData*, a custom object, with a Scala case class.

```scala
case class DeviceIoTData (battery_level: Long, c02_level: Long,cca2: String, cca3: String, cn: String, device_id: Long, device_name: String, humidity: Long, ip: String, latitude: Double, lcd: String, longitude: Double,scale:String,temp: Long,timestamp: Long)
```

Next, we can read the data from a JSON file.

```scala
/ read the json file and create the dataset from the 
// case class DeviceIoTData
// ds is now a collection of JVM Scala objects DeviceIoTData
val ds = spark.read.json(“/databricks-public-datasets/data/iot/iot_devices.json”).as[DeviceIoTData]
```

Three things happen here under the hood in the code above:

1. Spark reads the JSON, infers the schema, and creates a collection of DataFrames.
2. At this point, Spark converts your data into ==*DataFrame = Dataset[Row]*==, a collection of generic Row object, since it does not know the exact type.
3. Now, ==Spark converts the *Dataset[Row] $\rightarrow$ Dataset[DeviceIoTData]*== ***type-specific*** Scala JVM object, as dictated by the **class** *DeviceIoTData*. This is done with `as[]`

## SQL

### Renaming columns in SPARK SQL 

#### Naming New columns produced by functions acting on a DF/DS 

```scala
//you can use the following function with any other function that would produce a new coloum in the DS/DF
DF.groupBy(<"col name">).agg(expr(<"some sql expression">).alias("New col name"))
```

#### Rename existing columns

Explicitly rename a column

```scala
df.withColumnRenamed(<"old name">, <"new name">)
```

Renaming column in a single reference
```scala
df(<"col name">).alias(<"New Name">)
```


```scala
df.select(<"col name">).alias(<"New Name">)
```
### Dollar Notation

See [link](https://bzhangusc.wordpress.com/2015/03/29/the-column-class/) going in depth

### Joining

```scala
//Joining two tables with the same coloum names
df1.join(df2,"col name",joinType=<"joinType">)

//Joining on coloums with different names
df1.join(df2, $"col1" === $"col2",joinType=<"joinType">)
```

#### Joining columns with logic expressions

We must use logic expressions  but this adds a nuance to how we handle the dataframe.

```scala
val newDF = df1.join(df2, "col1" === "col1",joinType="<joinType>")
```

***Note:*** The above code will execute but will produce errors when we try refer to "col1" in the resultant dataframe `newDF`. The reason this happens is because Spark SQL joins the two tables **and** keeps both columns when logic statements are used $\Rightarrow$ you now have a new dataframe with two identically names columns. 

***Work around*** 
Use table aliases *.as()* to distinguish the columns and then drop them afterwards.  

```scala
df1.as('a).join(df2.as('b), $"a.col" === $"b.col" && <more join conditions>,joinType="<joinType>").drop($"a.col")
```

***Note:*** Dollar notation has to be used to drop the alias linked columns as they are defined as expressions  in the logic statement in the the join function. 

### A note on coloum naming

- [ ] why in some functions i can refer to a coloum name that has a space vs why functions break when you give it a col name with a space?

### Add column of the same values

```scala
import org.apache.spark.sql.functions._
df.withColumn("<col name>", lit(<value>))
```

### SQL Function

[Documentation](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$) 

## Dealing with DateTime :cry:

### Example: Find the minutes difference between two dates

```scala
val xx1 = ds.withColumn("<New Col Name>",expr("(unix_timestamp(<Date Col>) - unix_timestamp(<Date Col>))/3600"))
```

***Note:*** for some reason the col names can not have spaces in them when using *unix_timestamp*

### Spark built in functions 

- [ ] link to spark datetime functions

```scala
val dfWithAddedMonths = df.withColumn("FilterDate", add_months(joinedDF("logDate"), <int>))
```

### Using `DateTime` in Scala

```scala
val monthsOfBehavior = 12
val monthsTestWindow = 3
val date:String = DateTime.now().minusMonths(monthsTestWindow+monthsOfBehavior).toString().split("T")(0).replace("(", "").replace(")", "").replace(" ", "")  
val filterDF = joinedDF.filter("Date>"+ "cast('" + date + "' as date)");
```

## ML & MLLib

## 

 