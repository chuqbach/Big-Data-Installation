## 1. Prerequisite

Below is the requirement items that needs to be installed/setup before installing Spark
- Passwordless SSH
- Java Installation and JAVA_HOME declaration in .bashrc
- Hadoop Cluster Setup, with start-dfs.sh and start-yarn.sh are executed
- Hostname/hosts config

## 2. Download and Install Spark

In the Apache Spark Download [Website](https://spark.apache.org/downloads.html), choose your Spark release version and package type that use would like to install.

Released Version: currently Spark has 2 stable version
- Spark 2.x: wildly supported by many tools, like Elasticsearch, etc.
- Spark 3.x: introduce new features with performance optimization

Type of packages:
- Spark with Hadoop: currently Spark support 
- Spark without Hadoop: in case you already install Hadoop on your cluster. 

In this document, we will use **Spark 3.0.0 prebuilt with user-provided Apache Hadoop**

### 2.1. Download Apache Spark

**On Master node (or Namenode)**  
The rest of the document only be executed on the Master node (Namenode)

Switch to hadoop user  
`su hadoop`  

In /opt/, create spark directory and grant sudo permission
`cd /opt/`  
`sudo mkdir spark`  
`sudo chown -R hadoop /opt/spark`  

Download Apache Spark to /home/hadoop  
`cd ~`  
`wget -O spark.tgz http://mirrors.viethosting.com/apache/spark/spark-3.0.0/spark-3.0.0-bin-without-hadoop.tgz`   

Untar the file into /opt/spark directory  
`tar -zxf spark.tgz --directory=/opt/spark --strip=1`  

### 2.2. Setup Spark Environment Variable

Edit the .bashrc file  
`nano ~/.bashrc`  

Add the following line to the **.bashrc** file
```
# Set Spark Environment Variable
export SPARK_HOME=/opt/spark
export PYSPARK_PYTHON=python3
export PYSPARK_DRIVER_PYTHON="jupyter" 
export PYSPARK_DRIVER_PYTHON_OPTS="notebook --no-browser --port=8889" 
export SPARK_LOCAL_IP=192.168.0.1 # Your Master IP

export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export LD_LIBRARY_PATH=$HADOOP_HOME/lib/native:$LD_LIBRARY_PATH
export SPARK_DIST_CLASSPATH=$HADOOP_HOME/etc/hadoop
export SPARK_DIST_CLASSPATH=($HADOOP_HOME/bin/hadoop classpath)
```

### 2.3. Edit Spark Config Files

Apache Spark has some config file dedicated for specific purposes, which are:
- spark-env.sh: store default environment variables for Spark
- spark-defaults.conf: default config for spark-submit, including allocated resources, dependencies, proxies, etc.  

In spark/conf directory, make a copy of spark-env.sh.template and name it as spark-env.sh  
`cp spark-env.sh.template spark-env.sh`

Edit the spark-env.sh   
`nano spark-env.sh`  

Add below line to the file for Spark to known the Hadoop location  
`export SPARK_DIST_CLASSPATH=$(${HADOOP_HOME}/bin/hadoop classpath)`  

Make a copy of spark-defaults.conf.template and name it as spark-defaults.conf  
`cp spark-defaults.conf.template spark-defaults.conf`

Edit the spark-defaults.conf  
`nano spark-defaults.conf`  
Add below line to the file  
```
spark.jars.packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.4.5 # Kafka Dependency for Spark
spark.driver.extraJavaOptions -Dhttp.proxyHost=10.56.224.31 -Dhttp.proxyPort=3128 -Dhttps.proxyHost=10.56.224.31 -Dhttps.proxyPort=3128 # Set Proxy for Spark
```

### 2.4. Setup Spark Cluster Manager

Spark support different Cluster Manager, including Spark Standalone Cluster Manager, Yarn, Mesos, Kubernetes. In this document, we will config for Yarn and Spark Standalone Cluster Manager

#### 2.4.1. Spark on Yarn

In the spark-default.conf, declare the spark master config by add the following line

```
spark.master                      yarn
```

The following config should be consider as well

```
spark.eventLog.enabled            true
spark.eventLog.dir                hdfs://192.168.0.1:9000/spark-logs
spark.driver.memory               512m
spark.yarn.am.memory              512m
spark.history.provider            org.apache.spark.deploy.history.FsHistoryProvider
spark.history.fs.logDirectory     hdfs://192.168.0.1:9000/spark-logs
spark.history.fs.update.interval  10s
spark.history.ui.port             18080
```

Create hdfs directory to store spark logs  
`hdfs dfs -mkdir /spark-logs`  

#### 2.4.2. Spark Standalone Cluster Manager

On Spark Master node, copy the spark directory to all slave nodes

`scp -r /opt/spark hadoop@node2:/opt/spark`

In spark/conf directory, make a copy of slaves.template and name it as slaves  
`cp slaves.template slaves`

Edit the spark-defaults.conf  
`nano slaves`

Add the name of slave nodes to the file. 
```
slaves1 # hostname of node2
slaves2 # hostname of node3
```

**Start Spark Standalone Cluster mode**

Execute the following line  
`$SPARK_HOME/sbin/start-all.sh`  

Run jps to check if Spark is running on the Master and Slave Node  
`jps`

To stop Spark Standalone Cluster mode, do  
`$SPARK_HOME/sbin/stop-all.sh`

## 2.5. Spark Monitor UI

**Spark Standalone Cluster mode Web UI**  
`master:8080`

**Spark Application Web UI**  
`master:4040`

**Master IP and port for spark-submit**  
`master:7077`

### Submit jobs to Spark Submit

**Example**
```
spark-submit --deploy-mode client \
               --class org.apache.spark.examples.SparkPi \
               $SPARK_HOME/examples/jars/spark-examples_2.11-2.2.0.jar 10
```

**Run Mobile App Schemaless Consumer**  
```
$SPARK_HOME/bin/spark-submit --conf "spark.driver.extraJavaOptions=-Dhttp.proxyHost=10.56.224.31 -Dhttp.proxyPort=3128 -Dhttps.proxyHost=10.56.224.31 -Dhttps.proxyPort=3128" --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.4.4 --master spark://10.56.237.195:7077 --deploy-mode client /home/odyssey/projects/mapp/kafka_2_hadoop/Schemaless_Structure_Streaming.py
```

**Run HDFS dump data**  
```
$SPARK_HOME/bin/spark-submit --master spark://10.56.237.195:7077 --deploy-mode client /home/odyssey/projects/mapp/dump/hdfs_sstreaming_dump_1_1.py
```

**Run Kafka dump data**  
```
$SPARK_HOME/bin/spark-submit --conf "spark.driver.extraJavaOptions=-Dhttp.proxyHost=10.56.224.31 -Dhttp.proxyPort=3128 -Dhttps.proxyHost=10.56.224.31 -Dhttps.proxyPort=3128" --packages org.apache.spark:spark-sql-kafka-0-10_2.11:2.4.5 --master spark://10.56.237.195:7077 --deploy-mode client /home/odyssey/projects/mapp/dump/kafka_sstreaming_dump_1_1.py
```

**Run Schema Inferno**  
```
$SPARK_HOME/bin/spark-submit --master spark://10.56.237.195:7077 --deploy-mode client /home/odyssey/projects/mapp/schema_inferno/Schema_Saver.py
```

#### Note

Kafka dependencies might not work for version 2.12.  
Downgrade to 2.11 org.apache.spark:spark-sql-kafka-0-10_2.12:2.4.5 => org.apache.spark:spark-sql-kafka-0-10_2.11:2.4.5

### Spark Application Structure

**Python** 
```
import os
from pyspark import SparkContext, SparkConf
from pyspark.sql import SparkSession, SQLContext
from pyspark.sql import functions as F
from pyspark.sql.types import *


def asdf(path):
    print ...

if __name__ == '__main__':
    # Setup Spark Config
    conf = SparkConf()
    conf.set("spark.executor.memory", "2g")
    conf.set("spark.executor.cores", "2")
    conf.set("spark.cores.max", "8")
    conf.set("spark.driver.extraClassPath", "/opt/oracle-client/instantclient_19_6/ojdbc8.jar")
    conf.set("spark.executor.extraClassPath", "/opt/oracle-client/instantclient_19_6/ojdbc8.jar")
    sc = SparkContext(master="spark://10.56.237.195:7077", appName="Schema Inferno", conf=conf)
    spark = SparkSession(sc)
    sqlContext = SQLContext(sc)  

    # Generate Schema
    path = 'hdfs://master:9000/home/odyssey/data/raw_hdfs'
    temp_df = spark.read.json(f'{path}/*/part-00000')
    temp_df.printSchema()
    print("Number of records: ", temp_df.count())
    print("Number of distinct records: ", temp_df.distinct().count())
    #print(temp_df.filter('payload is null').count())
    save_schema(path)
```

### Log Print Control

Create the log4j file from the template in spark/conf directory  
`cp conf/log4j.properties.template conf/log4j.properties`

Replace this line  
`log4j.rootCategory=INFO, console`  
By this  
`log4j.rootCategory=WARN, console`
