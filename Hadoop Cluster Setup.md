# Concept
Setup Hadoop in one node, then replicate to others.

## **1. Prerequisites**
The following must be done on all node in the cluster, including installation of Java, SSH, user creation and other software utilities

### 1.1. Configure hosts and hostname on each node
Here we will edit the /etc/hosts and /ect/hostname fiel, so that we can use hostname instead of IP everytime we wish to use or ping any of these servers
* **Change hostname**
`sudo nano /etc/hostname`<br/>
Set your hostname to relative name (node1, node2, node3, etc.)

* **Change your hosts file**<br/>
`sudo nano /etc/hosts`<br/>
Add the following line in the structure `IP name`<br/>
```
192.168.0.1 node1
192.168.0.2 node2
192.168.0.3 node3
```

**Notice: Remember to delete/comment the following line if exists**<br/>
> ```# 127.0.1.1 node1 ```

### 1.2. Install OpenSSH
`sudo apt install openssh-client` <br/>
`sudo apt install openssh-server`

### 1.3. Install Java and config Java Environment Variable
Here we use JDK 8, as it is still the most stable and widely support version

* **For Oracle Java <br/>**
`sudo add-apt-repository ppa:webupd8team/java`<br/>
`sudo apt update`<br/>
`sudo apt install oracle-java8-installer` 

* **For OpenJDK Java <br/>**
`sudo apt install openjdk-8-jdk`

To verify the java version you can use the following command: <br/>
`java -version`

Set Java Environment Variable <br/>
* **Locate where java is installed<br/>**
`update-alternatives --config java`<br/>
The install path should be like this 
>  /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.252.b09-2.el7_8.x86_64/
* **Add the JAVA_HOME variable to bashrc file:<br/>**
`nano ~/.bashrc`<br/>
`export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.252.b09-2.el7_8.x86_64/`<br/>

### 1.4. Create dedicated user and group for Hadoop
We will use a dedicated Hadoop user account for running Hadoop applications. While that’s not required but it is recommended because it helps to separate the Hadoop installation from other software applications and user accounts running on the same machine (security, permissions, backups, etc).
* **Create Hadoop group and Hadoop user<br/>**
`sudo addgroup hadoopgroup`<br/>
`sudo adduser --ingroup hadoopgroup hadoop`<br/>
`sudo adduser hadoop sudo`<br/>

After this step we will only work on **hadoop** user. You can change your user by:<br/>
`su - hadoop`<br/>

### 1.5. SSH Configuration
Hadoop requires SSH access to manage its different nodes, i.e. remote machines plus your local machine.
* **Generate SSH key value pair**<br/>
`ssh-keygen -t rsa`<br/>
`cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys`<br/>
`sudo chmod 0600 ~/.ssh/authorized_keys`

To check whether your ssh works, runs:<br/>
`ssh localhost`

* **Config Passwordless SSH**<br/>
From each node, copy the ssh public key to others<br/>
`ssh-copy-id -i /home/hadoop/.ssh/id_rsa.pub hadoop@node2`<br/>
`ssh-copy-id -i /home/hadoop/.ssh/id_rsa.pub hadoopuser@node3`<br/>

To check your passwordless SSH, try:<br/>
`ssh hadooop@node2`<br/>

## **2. Download and Configure Hadoop**
In this article, we will install Hadoop on three machines

| IP | Host Name | Namenode | Datanode |
| ------ | ------ | ------ | ------ |
| 192.168.0.1 | node1 | Yes | No |
| 192.168.0.2 | node2 |  No | Yes |
| 192.168.0.3 | node3 |  No | Yes |

### 2.1. Download and setup hadoop<br/>
Firstly, The following directory also need to be create on /opt/ directory
```
/opt/
 |-- hadoop
 |    |-- logs
 |-- hdfs
 |    |-- datanode (if act as datanode)
 |    |-- namenode (if act as namenode)
 |-- mr-history (if act as namenode)
 |    |-- done
 |    |-- tmp
 |-- yarn (if act as namenode)
 |    |-- local 
 |    |-- logs
```

Assign permission for user hadoop on these folders<br/>
`sudo chown -R hadoop /opt/hadoop`<br/>
`sudo chown -R hadoop /opt/hdfs`<br/>
`sudo chown -R hadoop /opt/yarn`<br/>
`sudo chown -R hadoop /opt/mr-history`

Locate to /home/hadoop directory  
`cd ~`  

Download the installation Hadoop package from its website: [https://hadoop.apache.org/releases.html](https://hadoop.apache.org/releases.html)<br/>
`wget -c -O hadoop.tar.gz http://mirrors.viethosting.com/apache/hadoop/common/hadoop-3.2.1/hadoop-3.2.1.tar.gz`

Extract the file<br/>
`sudo tar -xvf hadoop.tar.gz --directory=/opt/hadoop --strip 1`

### 2.2. Configuration for Namenode
Inside the /hadoopx.y.z/etc/hadoop/ directory, edit the following files: **core-site.xml**, **hdfs-site.xml**, **yarn-site.xml**, **mapred-site.xml**. 

**On Namenode Server**<br/>
* *core-site.xml*
```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.0.1:9000/</value>
        <description>NameNode URI</description>
    </property>

    <property>
        <name>io.file.buffer.size</name>
        <value>131072</value>
        <description>Buffer size</description>
    </property>
</configuration>
```
* *hdfs-site.xml*
```
<configuration>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:///opt/hdfs/namenode</value>
    <description>NameNode directory for namespace and transaction logs storage.</description>
  </property>

  <property>
    <name>fs.checkpoint.dir</name>
    <value>file:///opt/hdfs/secnamenode</value>
    <description>Secondary Namenode Directory</description>
  </property>

  <property>
    <name>fs.checkpoint.edits.dir</name>
    <value>file:///opt/hdfs/secnamenode</value>
  </property>

  <property>
    <name>dfs.replication</name>
    <value>2</value>
    <description>Number of replication</description>
  </property>

  <property>
    <name>dfs.permissions</name>
    <value>false</value>
  </property>

  <property>
    <name>dfs.datanode.use.datanode.hostname</name>
    <value>false</value>
  </property>

  <property>
    <name>dfs.namenode.datanode.registration.ip-hostname-check</name>
    <value>false</value>
  </property>
</configuration>
```
* *yarn-site.xml*
```
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>192.168.0.1</value>
    <description>IP of hostname for Yarn Resource Manager Service</description>
  </property>

  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
    <description>Yarn Node Manager Aux Service</description>
  </property>

  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>

  <property>
    <name>yarn.nodemanager.local-dirs</name>
    <value>file:///opt/yarn/local</value>
  </property>

  <property>
    <name>yarn.nodemanager.log-dirs</name>
    <value>file:///opt/yarn/logs</value>
  </property>
</configuration>
```
* *mapred-site.xml*
```
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
    <description>MapReduce framework name</description>
  </property>

  <property>
    <name>mapreduce.jobhistory.address</name>
    <value>192.168.0.1:10020</value>
    <description>Default port is 10020.</description>
  </property>

  <property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>192.168.0.1:19888</value>
    <description>MapReduce JobHistory WebUI URL</description>
  </property>

  <property>
    <name>mapreduce.jobhistory.intermediate-done-dir</name>
    <value>/opt/mr-history/tmp</value>
    <description>Directory where history files are written by MapReduce jobs.</description>
  </property>

  <property>
    <name>mapreduce.jobhistory.done-dir</name>
    <value>/opt/mr-history/done</value>
    <description>Directory where history files are managed by the MR JobHistory Server.</description>
  </property>
</configuration>
```

* **workers**<br/>
Add the datanodes' IP to the file
```
192.168.0.2
192.168.0.3
```

### 2.3. Configuration for Datanode
Inside the /hadoopx.y.z/etc/hadoop/ directory, edit the following files: **core-site.xml**, **hdfs-site.xml**, **yarn-site.xml**, **mapred-site.xml**. 

**On Datanode Server**
* *core-site.xml*
```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://192.168.0.1:9000/</value>
        <description>NameNode URI</description>
    </property>
</configuration>
```
* *hdfs-site.xml*
```
<configuration>
  <property>
    <name>dfs.datanode.name.dir</name>
    <value>file:///opt/hdfs/datanode</value>
    <description>DataNode directory for namespace and transaction logs storage.</description>
  </property>

  <property>
    <name>dfs.replication</name>
    <value>2</value>
    <description>Number of replication</description>
  </property>

  <property>
    <name>dfs.permissions</name>
    <value>false</value>
  </property>

  <property>
    <name>dfs.datanode.use.datanode.hostname</name>
    <value>false</value>
  </property>

  <property>
    <name>dfs.namenode.datanode.registration.ip-hostname-check</name>
    <value>false</value>
  </property>
</configuration>
```
* *yarn-site.xml*
```
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>192.168.0.1</value>
    <description>IP of hostname for Yarn Resource Manager Service</description>
  </property>

  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
    <description>Yarn Node Manager Aux Service</description>
  </property>
</configuration>
```
* *mapred-site.xml*
```
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
    <description>MapReduce framework name</description>
  </property>
</configuration>
```

For more configuration information, see [https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/ClusterSetup.html](url)

### 2.4. Configure Hadoop Environment Variables
Add the following lines to the **.bashrc** file
```
export HADOOP_HOME=/opt/hadoop
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
export HDFS_NAMENODE_USER=hadoop
export HDFS_DATANODE_USER=hadoop
export HDFS_SECONDARYNAMENODE_USER=hadoop
export HADOOP_MAPRED_HOME=/opt/hadoop
export HADOOP_COMMON_HOME=/opt/hadoop
export HADOOP_HDFS_HOME=/opt/hadoop
export YARN_HOME=/opt/hadoop
```

Also add the following lines to the /opt/hadoop/etc/hadoop/hadoop-env.sh file
```
export HDFS_NAMENODE_USER=hadoop
export HDFS_DATANODE_USER=hadoop
export HDFS_SECONDARYNAMENODE_USER=hadoop
export YARN_RESOURCEMANAGER_USER=hadoop
export YARN_NODEMANAGER_USER=hadoop
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export HADOOP_HOME=/opt/hadoop
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
export HADOOP_LOG_DIR=/opt/hadoop/logs
```

## 3. Start HDFS, Yarn and monitor services on browser
### 3.1. Format the namenode, start Hadoop basic services
To format the namenode, type<br/>
`hdfs namenode –format`<br/>

Start the hdfs service<br/>
`start-dfs.sh`<br/>

The output should look like this<br/>
```
Starting namenodes on [hadoop-namenode]
Starting datanodes
Starting secondary namenodes [hadoop-namenode]
```

To start yarn service<br/>
`start-yarn.sh`<br/>
```
Starting resourcemanager
Starting nodemanagers
```

Start MapReduce JobHistory as *daemon*<br/>
`$HADOOP_HOME/bin/mapred --daemon start historyserver`

To check if the services is successfully started, *jps* to show the services

* On Namenode
```
16488 NameNode
16622 JobHistoryServer
17087 ResourceManager
17530 Jps
16829 SecondaryNameNode
```

* On Datanode
```
2306 DataNode
2479 NodeManager
2581 Jps
```

### 3.2. Monitor the services on browser

For Namenode<br/>
`https://IP:9870`

For Yarn<br/>
`https://IP:8088`

For MapReduce Job History<br/>
`https://IP:19888`

## 4. Run your first HDFS command & Yarn Job
### 4.1. Put and Get Data to HDFS
Create a books directory in HDFS  
`hdfs dfs -mkdir /books`

Grab a few books from the Gutenberg project  
cd ~
```
wget -O alice.txt https://www.gutenberg.org/files/11/11-0.txt
wget -O holmes.txt https://www.gutenberg.org/files/1661/1661-0.txt
wget -O frankenstein.txt https://www.gutenberg.org/files/84/84-0.txt
```

Then put the three books through HDFS, in the booksdirectory  
`hdfs dfs -put alice.txt holmes.txt frankenstein.txt /books`  

List the contents of the book directory
`hdfs dfs -ls /books`

Move one of the books to the local filesystem
`hdfs dfs -get /books/alice.txt`

### 4.2. Submit MapReduce Jobs to YARN
YARN jobs are packaged into jar files and submitted to YARN for execution with the command yarn jar. The Hadoop installation package provides sample applications that can be run to test your cluster. You’ll use them to run a word count on the three books previously uploaded to HDFS.  

Submit a job with the sample jar to YARN. On node-master, run
`yarn jar /opt/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.1.jar wordcount "/books/*" output`

After the job is finished, you can get the result by querying HDFS with hdfs dfs -ls output. In case of a success, the output will resemble:  
```
Found 2 items
-rw-r--r--   2 hadoop supergroup          0 2019-05-31 17:21 output/_SUCCESS
-rw-r--r--   2 hadoop supergroup     789726 2019-05-31 17:21 output/part-r-00000
```

Print the result with:  
`hdfs dfs -cat output/part-r-00000 | less`


