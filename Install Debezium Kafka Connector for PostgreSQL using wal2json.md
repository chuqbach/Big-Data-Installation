# Install Debezium Connector for PostgreSQL using wal2json plugin

## Overview
Debezium’s PostgreSQL Connector can monitor and record row-level changes in the schemas of a PostgreSQL database.

The first time it connects to a PostgreSQL server/cluster, it reads a consistent snapshot of all of the schemas. When that snapshot is complete, the connector continuously streams the changes that were committed to PostgreSQL 9.6 or later and generates corresponding insert, update and delete events. All of the events for each table are recorded in a separate Kafka topic, where they can be easily consumed by applications and services.

## Install wal2json

On Ubuntu:

```bash
sudo apt install postgresql-9.6-wal2json
```
Check if `wal2json.so` have already in `/usr/lib/postgresql/9.6`

## Configure PostgreSQL Server

### Setting up libraries, WAL and replication parameters

***Find and modify*** the following lines at the end of the `postgresql.conf` PostgreSQL configuration file in order to include the plug-in at the shared libraries and to adjust some **WAL** and **streaming replication** settings. You may need to modify it, if for example you have additionally installed `shared_preload_libraries`.

```
listeners = '*'
shared_preload_libraries = 'wal2json'
wal_level = logical  
max_wal_senders = 4  
max_replication_slots = 4
```

1. `max_wal_senders` tells the server that it should use a maximum of `4` separate processes for processing WAL changes
2. `max_replication_slots` tells the server that it should allow a maximum of `4` replication slots to be created for streaming WAL changes

### Setting up replication permissions

In order to give a user replication permissions, define a PostgreSQL role that has _at least_ the `REPLICATION` and `LOGIN` permissions.
 
Login to `postgres` user and run `psql`
```bash
sudo -u postgres psql
```
For example:

```sql
CREATE  ROLE  datalake REPLICATION LOGIN;
```
> However, Debezium need futher permissons to do initial snapshot. I used postgres superuser to fix this issue. You should test more to grant right permission for user role.

Next modify `pg_hba.conf` to accept remote connection from debezium kafka connector

```
host replication postgres 0.0.0.0/0 trust  
host replication postgres ::/0 trust  
host all all 192.168.1.2/32 trust
```
> **Note:** Debezium is installed on 192.168.1.2

### Database Test Environment

Back to `psql` console.

```sql
CREATE  DATABASE  test; 
CREATE  TABLE test_table ( 
	id  char(10) NOT  NULL, 
	code char(10), 
	PRIMARY KEY (id) 
);
```

-   **Create a slot**  named  `test_slot`  for the database named  `test`, using the logical output plug-in  `wal2json`
    

```bash
$ pg_recvlogical -d test --slot test_slot --create-slot -P wal2json
```

-   **Begin streaming changes**  from the logical replication slot  `test_slot`  for the database  `test`
    

```bash
$ pg_recvlogical -d test --slot test_slot --start -o pretty-print=1 -f -
```

-   **Perform some basic DML**  operations at  `test_table`  to trigger  `INSERT`/`UPDATE`/`DELETE`  change events
    

_Interactive PostgreSQL terminal, SQL commands_

```sql
test=# INSERT INTO test_table (id, code) VALUES('id1', 'code1');
INSERT 0 1
test=# update test_table set code='code2' where id='id1';
UPDATE 1
test=# delete from test_table where id='id1';
DELETE 1
```

Upon the  `INSERT`,  `UPDATE`  and  `DELETE`  events, the  `wal2json`  plug-in outputs the table changes as captured by  `pg_recvlogical`.

_Output for  `INSERT`  event_

```json
{
  "change": [
    {
      "kind": "insert",
      "schema": "public",
      "table": "test_table",
      "columnnames": ["id", "code"],
      "columntypes": ["character(10)", "character(10)"],
      "columnvalues": ["id1       ", "code1     "]
    }
  ]
}
```

_Output for  `UPDATE`  event_

```json
{
  "change": [
    {
      "kind": "update",
      "schema": "public",
      "table": "test_table",
      "columnnames": ["id", "code"],
      "columntypes": ["character(10)", "character(10)"],
      "columnvalues": ["id1       ", "code2     "],
      "oldkeys": {
        "keynames": ["id"],
        "keytypes": ["character(10)"],
        "keyvalues": ["id1       "]
      }
    }
  ]
}
```

_Output for  `DELETE`  event_

```json
{
  "change": [
    {
      "kind": "delete",
      "schema": "public",
      "table": "test_table",
      "oldkeys": {
        "keynames": ["id"],
        "keytypes": ["character(10)"],
        "keyvalues": ["id1       "]
      }
    }
  ]
}
```

When the test is finished, the slot  `test_slot`  for the database  `test`  can be removed by the following command:

```bash
$ pg_recvlogical -d test --slot test_slot --drop-slot
```

## Install Debezium Kafka Connector

Download Debezium Connector:

```bash
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/1.2.0.Final/debezium-connector-postgres-1.2.0.Final-plugin.tar.gz
```
Unzip to /opt/kafka/connect

```bash
tar -xvf debezium-connector-postgres-1.2.0.Final-plugin.tar.gz --directory=/opt/kafka/connect
```

Check if folder `debezium-connector-postgres` in `/opt/kafka/connect`

### Test connector

Edit `config/connect-file-source.properties` as following:

```properties
name=postgres-cdc-source  
connector.class=io.debezium.connector.postgresql.PostgresConnector  
#snapshot.mode=never  
tasks.max=1  
plugin.name=wal2json  
database.hostname=192.168.1.1  
database.port=5432  
database.user=postgres  
#database.password=postgres  
database.dbname=test  
# slot.name=test_slot  
database.server.name=fullfillment  
#table.whitelist=public.inventory
```

Append kafka plugin folder to `plugin.path` in file `config/connect-standalone.properties`

```properties
plugin.path=/usr/local/share/java,/usr/local/share/kafka/plugins,/opt/connectors,/opt/kafka/connect
```
Run kafka connector in standalone mode

```bash
bin/connect-standalone.sh config/connect-standalone.properties config/connect-file-source.properties
```

#### Topics Names

The PostgreSQL connector writes events for all insert, update, and delete operations on a single table to a single Kafka topic. By default, the Kafka topic name is  _serverName_._schemaName_._tableName_  (configuration in `connect-file-source.properties`) where  _serverName_  is the logical name of the connector as specified with the  `database.server.name`  configuration property,  _schemaName_  is the name of the database schema where the operation occurred, and  _tableName_  is the name of the database table on which the operation occurred.

For example, consider a PostgreSQL installation with a  `postgres`  database and an  `inventory`  schema that contains four tables:  `products`,  `products_on_hand`,  `customers`, and  `orders`. If the connector monitoring this database were given a logical server name of  `fulfillment`, then the connector would produce events on these four Kafka topics:

-   `fulfillment.inventory.products`
    
-   `fulfillment.inventory.products_on_hand`
    
-   `fulfillment.inventory.customers`
    
-   `fulfillment.inventory.orders`
    

If on the other hand the tables were not part of a specific schema but rather created in the default  `public`  PostgreSQL schema, then the name of the Kafka topics would be:

-   `fulfillment.public.products`
    
-   `fulfillment.public.products_on_hand`
    
-   `fulfillment.public.customers`
    
-   `fulfillment.public.orders`

In this example, the topic's name is `fulfillment.public.test_table`

Run:

```bash
bin/kafka-topics.sh --list --bootstrap-server 192.168.1.2:9092
```
You should see `fullfillment.public.test_table` in the ouput.

Start receive message from debezium:

```bash
bin/kafka-console-consumer.sh --bootstrap-server 192.168.1.2:9092 --from-beginning --topic fullfillment.public.test_table
```
