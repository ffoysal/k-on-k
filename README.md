# Kafka On Kubernetes by Kustomize

In this tutorial we will install kafka in a kubernetes cluster (tried in docker-desktop for mac). After deployment is done will do the following

- create a topic
- insert data into the topic
- create a database in mysql so that those messages from topic get pushed
- create mysql sink connector
- show data in mysql table

## Deploy Kafka

to deploy kafaka run this

```console
kubectl apply -k github.com/ffoysal/k-on-k
```

once deployed these are the pods will show up

```console

kubectl get pods

NAME                                  READY   STATUS    RESTARTS   AGE
cp-control-center-5b797964c4-tbk2k    1/1     Running   0          35m
cp-kafka-0                            2/2     Running   1          35m
cp-kafka-1                            2/2     Running   1          34m
cp-kafka-2                            2/2     Running   1          34m
cp-kafka-connect-6589c98678-cr4jm     2/2     Running   0          35m
cp-ksql-server-67f4bbbc99-f5zmr       2/2     Running   0          35m
cp-schema-registry-5c974dbc9f-k9rxg   2/2     Running   0          35m
cp-zookeeper-0                        2/2     Running   0          35m
mysql-5fdc8fc68-m9jzm                 1/1     Running   0          35m
postgres-77bd4fd98d-2xpsl             1/1     Running   0          35m
```

These are the services

```console
kubectl get service

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
cp-control-center       ClusterIP   10.99.29.253    <none>        9021/TCP            36m
cp-kafka                ClusterIP   10.106.20.199   <none>        9092/TCP            36m
cp-kafka-connect        ClusterIP   10.111.49.17    <none>        8083/TCP            36m
cp-kafka-headless       ClusterIP   None            <none>        9092/TCP            36m
cp-ksql-server          ClusterIP   10.107.252.13   <none>        8088/TCP            36m
cp-schema-registry      ClusterIP   10.107.2.49     <none>        8081/TCP            36m
cp-zookeeper            ClusterIP   10.99.47.22     <none>        2181/TCP            36m
cp-zookeeper-headless   ClusterIP   None            <none>        2888/TCP,3888/TCP   36m
kubernetes              ClusterIP   10.96.0.1       <none>        443/TCP             44d
mysql-service           ClusterIP   10.99.34.152    <none>        3306/TCP            36m
postgres-service        ClusterIP   10.107.0.224    <none>        5432/TCP            36m
```

## Create Topic

will use `ksql` to create topic. There is already a ksqdb server `cp-ksql-server` is running. So we need a ksql-cli container and since all the services type are clusterIP, need a pod to play with. Will use ksql-cli container to create a temporary pod so that we can write some ksql

```console
kubectl run temp-ksql --image=confluentinc/ksqldb-cli:0.11.0 --restart=Never --rm -it -- sh
```

once inside the container, then run

```console
ksql http://cp-ksql-server:8088
```

it will open the ksql promt in the terminal like this

```console
ksql http://cp-ksql-server:8088
OpenJDK 64-Bit Server VM warning: Option UseConcMarkSweepGC was deprecated in version 9.0 and will likely be removed in a future release.

                  ===========================================
                  =       _              _ ____  ____       =
                  =      | | _____  __ _| |  _ \| __ )      =
                  =      | |/ / __|/ _` | | | | |  _ \      =
                  =      |   <\__ \ (_| | | |_| | |_) |     =
                  =      |_|\_\___/\__, |_|____/|____/      =
                  =                   |_|                   =
                  =  Event Streaming Database purpose-built =
                  =        for stream processing apps       =
                  ===========================================

Copyright 2017-2020 Confluent Inc.

CLI v0.11.0, Server v5.5.0 located at http://cp-ksql-server:8088

Having trouble? Type 'help' (case-insensitive) for a rundown of how things work!

ksql>
```

create the topic

```console
CREATE STREAM MYTEST (COL1 INT, COL2 VARCHAR)
  WITH (KAFKA_TOPIC='mytest', PARTITIONS=1, VALUE_FORMAT='AVRO');
```

Lets insert some records into the topic

```console
INSERT INTO MYTEST (ROWKEY, COL1, COL2) VALUES ('X',1,'FOO');
INSERT INTO MYTEST (ROWKEY, COL1, COL2) VALUES ('Y',2,'BAR');
```

## Create Sink Connector

Will use rest endpoint of kafka connect to create sink connector. Before that need to create a database in mysql so that `kafka-connect` can create table in the database.

To create the database will use another container that has `mysql-client` so that it can talk to mysql server. open another terminal window keep the ksql terminal window open, will come back to that window latter on.

```console
kubectl run temp-tool --image=ffoysal/swiss-knife --restart=Never --rm -it -- sh

mysql
## Run ksqldb cli

```console
kubectl run temp --image=confluentinc/ksqldb-cli:0.11.0 --restart=Never --rm -it -- sh

mysql -h mysql-service -u root -padmin
```

it will open the mysql prompt like this

```console
mysql -h mysql-service -u root -padmin
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 3
Server version: 5.7.31 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]>
```

create the database

```console
MySQL [(none)]> create database mydemo;
```

database should be shown now

```console
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mydemo             |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

exit from mysql prompt

```console
MySQL [(none)]> exit
Bye
```

Now issue a curl command to kafka-connect to create the connector using the database name as follows

```console
curl -X PUT http://cp-kafka-connect:8083/connectors/sink-jdbc-mysql-mydemo/config \
     -H "Content-Type: application/json" -d '{
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "connection.url": "jdbc:mysql://mysql-service:3306/mydemo",
    "topics": "mytest",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "connection.user": "root",
    "connection.password": "admin",
    "auto.create": true,
    "auto.evolve": true,
    "insert.mode": "insert",
    "pk.mode": "record_key",
    "pk.fields": "MESSAGE_KEY"
}'
```

after executing the above command go to the ksql window and show the connectors

```console
ksql> show connectors;

 Connector Name         | Type | Class                                       | Status
-----------------------------------------------------------------------------------------------------------
 sink-jdbc-mysql-mydemo | SINK | io.confluent.connect.jdbc.JdbcSinkConnector | RUNNING (1/1 tasks RUNNING)
-----------------------------------------------------------------------------------------------------------
```

Now connect to mysql again to see if those records are in the table or not

```console
mysql -h mysql-service -u root -padmin

MySQL [(none)]> use mydemo;

MySQL [mydemo]> show tables;
+------------------+
| Tables_in_mydemo |
+------------------+
| mytest           |
+------------------+
1 row in set (0.000 sec)

MySQL [mydemo]> select * from mytest;
+------+------+-------------+
| COL1 | COL2 | MESSAGE_KEY |
+------+------+-------------+
|    1 | FOO  | X           |
|    2 | BAR  | Y           |
+------+------+-------------+
2 rows in set (0.001 sec)

```

incase if you want to delete connector 

```console
curl -X DELETE http://cp-kafka-connect:8083/connectors/sink-jdbc-mysql-mydemo
```

TODO:

- create postgres source connector

References And Copied From:

- https://github.com/confluentinc/cp-helm-charts

- https://medium.com/swlh/configuring-kafka-connect-on-kubernetes-8547641acdbb
