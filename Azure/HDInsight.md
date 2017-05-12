# Azure HDInsight + Azure ML + Spark + PowerBI + Data Lake + Storage Account



[TOC]

## Setup

### Problems encountered

1) *From correspondence with MS Support:* `HIVE` queries within `Azure ML` do not support importing or exporting data housed within `data lakes` in `HDInsights` 

2) Be aware of SAS when dealing with different storage. 
​	*For example:* If `Azure MLs` default storage account is "A" and `HDInsight` uses storage account "B" then `Azure ML` will only have read rights on the data controlled by account "B" unless the correct configurations are done. 

***See Correspondence bellow***

> I think the error message is expected and documented here:
>
> [https://msdn.microsoft.com/en-us/library/azure/mt186521.aspx](https://na01.safelinks.protection.outlook.com/?url=https%3A%2F%2Fmsdn.microsoft.com%2Fen-us%2Flibrary%2Fazure%2Fmt186521.aspx&data=02%7C01%7Cgawronr%40microsoft.com%7C9b746429964c4d107a4a08d496df6f62%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C636299333780773951&sdata=ECcgjekIUT4BhH%2BlHir%2Fb0lFq%2FB4ze23TDO2JDjX6QY%3D&reserved=0)
>
> It looks like you do not use the default hdinsight storage account. There are somerestrictions which are documented here:
>
> [https://docs.microsoft.com/en-us/azure/hdinsight/hdinsight-hadoop-use-blob-storage](https://na01.safelinks.protection.outlook.com/?url=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fhdinsight%2Fhdinsight-hadoop-use-blob-storage&data=02%7C01%7Cgawronr%40microsoft.com%7C9b746429964c4d107a4a08d496df6f62%7C72f988bf86f141af91ab2d7cd011db47%7C1%7C0%7C636299333780773951&sdata=%2FnjPzB2unPQyrbBnA22sz4ucQTxKtOu3Tj8a0GOkLkI%3D&reserved=0)
>
> “Here are some considerations when using Azure Storage account withHDInsight clusters.+ 
>
> - **Containers in the storage accounts that are connected to a cluster:** Because the account name and key area associated with the cluster during creation, you have full access to the blobs in those containers.
> - **Public containers or public blobs in storage accounts that are NOTconnected to a cluster:** You have read-only permission to the blobs in the containers.”
>
> You can try following solution:
>
> You have to difine SAS (shared accesssignature) for your blob container:
>
> [https://docs.microsoft.com/en-us/azure/storage/storage-dotnet-shared-access-signature-part-1](https://docs.microsoft.com/en-us/azure/storage/storage-dotnet-shared-access-signature-part-1)
>
> [https://docs.microsoft.com/en-us/azure/storage/storage-dotnet-shared-access-signature-part-2](https://docs.microsoft.com/en-us/azure/storage/storage-dotnet-shared-access-signature-part-2)
>
> You can use Azure Storage Explorer for the above. No need to run c# or python applications. 
>
> In `Ambari` add this key to core-site.xml in 
>
> > -> HDFS -> Configs tab -> Advanced tab -> Custom core-site section.
> > -> Expand the Custom core-site section -> Add property -> Use the following values for the Key and Value fields:
> >
> > > Key: fs.azure.sas.CONTAINERNAME.STORAGEACCOUNTNAME.blob.core.windows.net
> > >
> > > Value: The SAS returned by the C# or Python application you ran previously or from Azure Explorer

3) By default `HDInsights` uses `python 2.7` for `pyspark` . This is managed by `anaconda` and the default settings can be configured through `Ambari`.

```shell
# go to <hdinsight name>.azurehdinsight.net/#/main/services/SPARK2/configs
# Under "Advanced spark2-env" -> "content"
# change the following line 
export PYSPARK_PYTHON=${PYSPARK_PYTHON:-${PYSPARK_PYTHON:-/usr/bin/anaconda/bin/python}
# to this
export PYSPARK_PYTHON=${PYSPARK_PYTHON:-/usr/bin/anaconda/envs/py35/bin/python3}
# where /usr/bin/anaconda/envs/py35/bin/python3 is your python3 address
```

## Hadoop

### Bash Commands

```shell
# accessing data lake
hadoop fs -get adl://<data lake name>.azuredatalakestore.net/clusters/transnetdemohdinsight/user/spark/"Spark Jobs"/TimeSheets.jar
```

```shell
# accessing storage account
hadoop fs -get wasbs:///user/csvFiles/EmpData4.csv
# instead of using /// you can also specify the entire cluster path
```

### PySpark

```python
# accessing storage account in pyspark
import subprocess
cat = subprocess.Popen(["hadoop", "fs", "-cat", "wasbs:///user/csvFiles/EmpData4.csv"], stdout=subprocess.PIPE)
```

```python
# data lake account in pyspark
import subprocess
cat = subprocess.Popen(["hadoop", "fs", "-cat", "adl://<data lake name>.azuredatalakestore.net/clusters/transnetdemohdinsight/user/csvFiles/EmpData4.csv"], stdout=subprocess.PIPE)
```

## Azure ML

1) If you ***data frame column names*** have spaces in Azure ML you will encounter errors when exporting with HIVE. This is because HIVE does not support name space column names. 

```shell
#AZURE ML Error LOG
requestId = 10b6abf3f0a74a3886ba549979af3fd3 errorComponent=Module. taskStatusCode=400. {"Exception":{"ErrorId":"FailedToCreateHiveTable","ErrorCode":"0090","ExceptionType":"ModuleException","Message":"Error 0090: \r\n The Hive table \"test12345\" could not be created. For a HDInsight cluster, please ensure the Azure storage account name associated with cluster is \"transnetstorageaccount\".\r\n ","Exception":{"ErrorId":"InvalidHiveScript","ErrorCode":"0068","ExceptionType":"ModuleException","Message":"Error 0068: Hive script describe test12345; is not correct."}}}Error: Error 0090: The Hive table "test12345" could not be created. For a HDInsight cluster, please ensure the Azure storage account name associated with cluster is "transnetstorageaccount". Process exited with error code -2
```

2) *From correspondence with MS Support:* HIVE quaries within Azure ML do not support importing or exporting data housed within data lakes in HDInsights 

## Hive

### Persisting permanent table 

==NOTE== Make sure column names don't have spaces

```python
# python
from pyspark.sql import HiveContext
from pyspark.sql import SparkSession

warehouse_location = "wasbs:///hive/warehouse" #the hive warehouse was in a Azure storage account
spark = SparkSession \
    .builder \
    .config("spark.sql.warehouse.dir", warehouse_location) \
    .enableHiveSupport() \
    .getOrCreate()

all_data.createOrReplaceTempView("tempTable")
spark.sql("DROP TABLE IF EXISTS RFFeatures")
spark.table("tempTable").write.saveAsTable("RFFeatures")
```

```scala
// Scala
import org.apache.spark._
import org.apache.spark.SparkContext._
import org.apache.spark.sql.DataFrame
import org.apache.spark.sql.functions._

val warehouseLocation = "adl://transnetdemodlstore.azuredatalakestore.net/clusters/transnetdemohdinsight/hive/warehouse"   //hive warehouse housed within data lake  

val spark = SparkSession
      .builder
      .appName("TimeSheetsML")
      //.master("local[*]")
      //.config("spark.sql.warehouse.dir", "file:///C:/temp") // Necessary to work around a Windows bug in Spark 2.0.0; omit if you're not on Windows.
      .config("spark.sql.warehouse.dir", warehouseLocation)
      .enableHiveSupport()
      .getOrCreate()

<dataframe>.createOrReplaceTempView("everything_table");
spark.sql("DROP TABLE IF EXISTS logistic_regression_features")
spark.table("everything_table").write.saveAsTable("logistic_regression_features")
```