# 1. Prerequisite

Below is the requirement items that needs to be installed/setup before installing Spark
- Passwordless SSH
- Java Installation and JAVA_HOME declaration in .bashrc
- Hostname/hosts config
- Hadoop Cluster Setup, with start-dfs.sh and start-yarn.sh are executed
- Spark on Yarn

# 2. Download and Install Apache Hive

In the Apache Hive Download [https://downloads.apache.org/hive/](https://downloads.apache.org/hive/), choose your Hive release version that use would like to install. 

Released Version: currently Spark has 2 stable version
- Hive 2.x: This release works with Hadoop 2.x.y
- Hive 3.x: This release works with Hadoop 3.x.y 

In this document, we will use **Hive 2.3.7 prebuilt version**

## 2.1. Download Apache Hive

**On Master node (or Namenode)**  
The rest of the document only be executed on the Master node (Namenode)

Switch to hadoop user  
`su hadoop`  

In /opt/, create Hive directory and grant sudo permission
`cd /opt/`  
`sudo mkdir hive`  
`sudo chown -R hadoop /opt/hive`  

Download Apache Hive to /home/hadoop  
`cd ~`  
`wget -O hive.tgz https://downloads.apache.org/hive/hive-2.3.7/apache-hive-2.3.7-bin.tar.gz`   

Untar the file into /opt/spark directory  
`tar -zxf hive.tgz --directory=/opt/hive --strip=1`  

## 2.2. Setup Hive Environment Variable

Edit the .bashrc file  
`nano ~/.bashrc`  

Add the following line to the **.bashrc** file
```
# Hive Environment Configuration
export HIVE_HOME=/opt/hive
export PATH=$HIVE_HOME/bin:$PATH
```
Also Hive uses Hadoop, so you must have Hadoop in your path or your **.bashrc**  must have
`export HADOOP_HOME=<hadoop-install-dir>`  

## 2.3. Create Hive working directory on HDFS

In addition, you must use below HDFS commands to create /tmp and /user/hive/warehouse (aka hive.metastore.warehouse.dir) and set them chmod g+w before you can create a table in Hive.  

`hdfs dfs -mkdir     /tmp`  
`hdfs dfs -mkdir -p  /user/hive/warehouse`  
`hdfs dfs -chmod g+w /tmp`  
`hdfs dfs -chmod g+w /user/hive/warehouse`  


## 2.4. Setup Hive Metastore and HiveServer2

### 2.4.1. Configuring a Remote PostgreSQL Database for the Hive Metastore 

In order to use Hive Metastore, a Database is required to store the Hive Metadata. Though Hive provides an embedded Database (Apache Derby), this mode should only be used for experimental purposes only. Here we will setup a remote PostgreSQL Database to run Hive Metastore.

Before you can run the Hive metastore with a remote PostgreSQL database, you must configure a connector to the remote PostgreSQL database, set up the initial database schema, and configure the PostgreSQL user account for the Hive user.

**Install and start PostgreSQL**  
Run the following to install PostgreSQL Database  
`sudo apt-get install postgresql`

To ensure that your PostgreSQL server will be accessible over the network, you need to do some additional configuration.

First you need to edit the `postgresql.conf` file. Set the `listen_addresses` property to `*`, to make sure that the PostgreSQL server starts listening on all your network interfaces. Also make sure that the `standard_conforming_strings property` is set to `off`.

`nano /etc/postgresql/10/main/postgresql.conf`  

Adjust the required properties  

```
listen_addresses    '*'
standard_conforming_strings property      off
```

You also need to configure authentication for your network in `pg_hba.conf`. You need to make sure that the PostgreSQL user that you will create later in this procedure will have access to the server from a remote host. To do this, add a new line into `pg_hba.conf` that has the following information:  
`nano /etc/postgresql/10/main/pg_hba.conf`  
Add the following line to the file
```
host    all         all         0.0.0.0         0.0.0.0               md5
```

After all is done, start PostgreSQL Server  
`sudo service postgresql start`  


**Install the PostgreSQL JDBC driver**  
On the client, a JDBC driver is required to connect to the PostgreSQL Database. Install the JDBC and copy or link it to hive/lib folder.  
`sudo apt-get install libpostgresql-jdbc-java`  
`ln -s /usr/share/java/postgresql-jdbc4.jar /opt/hive/lib/postgresql-jdbc4.jar`  

**Create the metastore database and user account**  
Login to postgres user and start a psql  
`sudo -u postgres psql`  

Add dedicated User and Database for Hive Metastore

```
postgres=# CREATE USER hive WITH PASSWORD '123123';
postgres=# CREATE DATABASE metastore;
```

Use the Hive Schema Tool to create the metastore tables.  
`/opt/hive/bin/schematool -dbType postgres -initSchema`


The output should be like
```
Metastore connection URL:	 jdbc:postgresql://localhost:5432/metastore
Metastore Connection Driver :	 org.postgresql.Driver
Metastore connection User:	 hive
Starting metastore schema initialization to 2.3.0
Initialization script hive-schema-2.3.0.postgres.sql
Initialization script completed
schemaTool completed
```

### 2.4.2. Adjust the Hive and Hadoop config files for the Hive Metastore 
In /opt/hive/conf, create config file hive-site.xml from hive-default.xml.template (or not)  
`nano hive-site.xml`  

Add the following lines
```
<configuration>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:postgresql://localhost:5432/metastore</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.postgresql.Driver</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hive</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>123123</value>
  </property>

  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>hdfs://192.168.0.5:9000/user/hive/warehouse</value>
  </property>
```

In order to avoid the issue  `Cannot connect to hive using beeline, user root cannot impersonate anonymous` when connecting to HiveServer2 from `beeline`, from hadoop/etc/hadoop, add to the core-site.xml, with `[username]` is your local user that will be use in `beeline`   
```
  <property>
    <name>hadoop.proxyuser.[username].groups</name>
    <value>*</value>
  </property>
  <property>
    <name>hadoop.proxyuser.[username].hosts</name>
    <value>*</value>
  </property>
```

### 2.4.3. Start Hive Metastore Server & HiveServer2

Respectively start Hive Metastore Server and HiveServer2 services  
`hive --service metastore`  
From another connection session, start HiveServer2  
`hiveserver2`  

Run Beeline (the HiveServer2 CLI).  
`beeline`  
Connecto to HiveServer2  
`!connect jdbc:hive2://localhost:10000`
Input any user and password, and the result should be  
```
Connecting to jdbc:hive2://localhost:10000
Enter username for jdbc:hive2://localhost:10000: hadoop
Enter password for jdbc:hive2://localhost:10000: ******
Connected to: Apache Hive (version 2.3.7)
Driver: Hive JDBC (version 2.3.7)
Transaction isolation: TRANSACTION_REPEATABLE_READ
```

From here, you can run your query on Hive

For monitor, you can access HiveServer WebUI on port 10002
`http://localhost:10002`

## 2.6. Setup Hive Execution Engine to Spark

While MR remains the default engine for historical reasons, it is itself a historical engine and is deprecated in Hive 2 line. It may be removed without further warning. Therefore, choosing another execution engine is wise decision. Here, we will config Spark to be Hive Execution Engine.

First, add the following config to `hive-site.xml` in `/opt/hive/conf`  
```
<property>
    <name>hive.execution.engine</name>
    <value>spark</value>
    <description>
      Expects one of [mr, tez, spark]
    </description>
  </property>
```

Then configuring YARN to distribute an equal share of resources for jobs in the YARN cluster.  
```
<property>
  <name>yarn.resourcemanager.scheduler.class</name>
  <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
</property>
```  

Next, add the Spark libs to Hive's class path as below.

Edit `/opt/hive/bin/hive` file (backup this file is if anything wrong happens)  
`cp hive hive_backup`  
`nano hive`  

Add Spark Libs to Hive  
```
for f in ${SPARK_HOME}/jars/*.jar; do
  CLASSPATH=${CLASSPATH}:$f;
done
```

Finally, upload all jars in `$SPARK_HOME/jars` to hdfs folder (for example:hdfs:///xxxx:9000/spark-jars) and add following in `hive-site.xml`  
```
<property>
  <name>spark.yarn.jars</name>
  <value>hdfs://xxxx:9000/spark-jars/*</value>
</property>
```

You may get below exception if you missed the CLASSPATH configuration above.  
```Exception in thread "main" java.lang.NoClassDefFoundError: scala/collection/Iterable```

Another solution should be considered, though the author of this document hasn't successfully experienced yet. [https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started#HiveonSpark:GettingStarted-ConfiguringHive](https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started#HiveonSpark:GettingStarted-ConfiguringHive)

From now on, when doing any execution, the output should be like  

```
hive> insert into pokes values(2, 'hai');
Query ID = hadoop_20200718125927_5e68f5c3-66b5-4459-a145-71b44c658939
Total jobs = 1
Launching Job 1 out of 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Starting Spark Job = 84d25db4-ea71-4cf1-8e17-d9018390d005
Running with YARN Application = application_1595048642632_0006
Kill Command = /opt/hadoop/bin/yarn application -kill application_1595048642632_0006

Query Hive on Spark job[0] stages: [0]

Status: Running (Hive on Spark job[0])
--------------------------------------------------------------------------------------
          STAGES   ATTEMPT        STATUS  TOTAL  COMPLETED  RUNNING  PENDING  FAILED
--------------------------------------------------------------------------------------
Stage-0 ........         0      FINISHED      1          1        0        0       0
--------------------------------------------------------------------------------------
STAGES: 01/01    [==========================>>] 100%  ELAPSED TIME: 7.11 s
--------------------------------------------------------------------------------------
Status: Finished successfully in 7.11 seconds
Loading data to table default.pokes
OK
Time taken: 30.264 seconds
```



## 2.5. Connecting Apache Spark to Apache Hive
### 2.5.1 Config hive-site.xml and spark-default.sh
Create `/opt/spark/conf/hive-site.xml` and define `hive.metastore.uris` configuration property (that is the thrift URL of the Hive Metastore Server).  

```
<configuration>
  <property>
    <name>hive.metastore.uris</name>
    <value>thrift://localhost:9083</value>
  </property>
</configuration>
```

Optionally, from `log4j.properties.template` create `log4j.properties` add the following for a more Hive low-level logging:

`cp log4j.properties.template log4j.properties`

```
log4j.logger.org.apache.spark.sql.hive.HiveUtils$=ALL
log4j.logger.org.apache.spark.sql.internal.SharedState=ALL
log4j.logger.org.apache.spark.sql.hive.client.HiveClientImpl=ALL
```

The following config also should be set on `/opt/spark/conf/spark-default.sh` file  
```
spark.master                      yarn
spark.serializer                 org.apache.spark.serializer.KryoSerializer
spark.eventLog.enabled            true
spark.eventLog.dir                hdfs://192.168.0.5:9000/spark-logs
spark.driver.memory               512m
spark.yarn.am.memory              512m
spark.history.provider            org.apache.spark.deploy.history.FsHistoryProvider
spark.history.fs.logDirectory     hdfs://192.168.0.5:9000/spark-logs
spark.history.fs.update.interval  10s
spark.history.ui.port             18080

# Spark Hive Configuartions
spark.sql.catalogImplementation   hive
```



### 2.5.2. Start your PySpark Hive 

However, since Hive has a large number of dependencies, **these dependencies are not included in the default Spark distribution**. If **Hive dependencies can be found on the classpath**, Spark will load them automatically. Note that these Hive dependencies must also be present on all of the worker nodes, as they will need access to the Hive serialization and deserialization libraries (SerDes) in order to access data stored in Hive. 

By adding the dependencies in the code, PySpark will automatically download it from the internet. The dependencies include:  

`org.apache.spark:spark-hive_2.11:2.4.6` (The 2.12 version might not work)  
`org.apache.avro:avro-mapred:1.8.2`  

From any IDE, execute the following Python script:  
```
import findspark
findspark.init()
findspark.find()
import os
from pyspark.sql import SparkSession
from pyspark.sql import Row

submit_args = '--packages org.apache.spark:spark-hive_2.11:2.4.6 --packages org.apache.avro:avro-mapred:1.8.2 pyspark-shell'
if 'PYSPARK_SUBMIT_ARGS' not in os.environ:
    os.environ['PYSPARK_SUBMIT_ARGS'] = submit_args
else:
    os.environ['PYSPARK_SUBMIT_ARGS'] += submit_args

# warehouse_location points to the default location for managed databases and tables
warehouse_location = 'hdfs://192.168.0.5:9000/user/hive/warehouse'

spark = SparkSession \
    .builder \
    .appName("Python Spark SQL Hive integration example") \
    .config("spark.sql.warehouse.dir", warehouse_location) \
    .enableHiveSupport() \
    .getOrCreate()
```

Then, hopefully you can query into Hive Tables. You can check the SparkSession config by  

`spark.sparkContext.getConf().getAll()`

Such output should be there
```
('spark.sql.warehouse.dir', 'hdfs://192.168.0.5:9000/user/hive/warehouse'),
('spark.sql.catalogImplementation', 'hive')
```

## 2.6. Spark SQL on Hive Tables

Apache Spark provide a machenism to query into Hive Tables. Try the following Python script  

**Select from Hive Table**
```
# With pokes is a Hive table
spark.sql('select * from pokes').show()
```

**Write to Hive Table**
```
df = spark.range(10).toDF('number')
df.registerTempTable('number')
spark.sql('create table number as select * from number')
```

You can check whether the `number` table is available using `hive` or `beeline`
```
hive> show tables;
OK
number
pokes
values__tmp__table__1
Time taken: 0.039 seconds, Fetched: 3 row(s)

hive> select * from number;
OK
0
1
2
3
4
5
6
7
8
9
Time taken: 0.135 seconds, Fetched: 10 row(s)
```


```python

```
