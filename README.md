# Mysql-Kafka-Debezium
This tutorial walks you through running Debezium 0.9.5.Final for change data capture (CDC). You will use Docker (1.9 or later) to start the Debezium services, run a MySQL database server with a simple example database, use Debezium to monitor the database, and see the resulting event streams respond as the data in the database changes.


FLOW

MYSQL------>>>>> Debezium kafka connect ------->>>>Kafka



What is Debezium?
Debezium is a distributed platform that turns your existing databases into event streams, so applications can see and respond immediately to each row-level change in the databases. Debezium is built on top of Apache Kafka and provides Kafka Connect compatible connectors that monitor specific database management systems. Debezium records the history of data changes in Kafka logs, from where your application consumes them. This makes it possible for your application to easily consume all of the events correctly and completely. Even if your application stops (or crashes), upon restart it will start consuming the events where it left off so it misses nothing.

Debezium 0.9.5.Final includes support for monitoring MySQL database servers with its MySQL connector, and this is what we’ll use in this demonstration. Support for other DBMSes will be added in future releases.

Running Debezium with Docker
Running Debezium involves three major services: ZooKeeper, Kafka, and Debezium’s connector service. This tutorial walks you through starting a single instance of these services using Docker and Debezium’s Docker images. Production environments, on the other hand, require running multiple instances of each service to provide the performance, reliability, replication, and fault tolerance. This can be done with a platform like OpenShift and Kubernetes that manages multiple Docker containers running on multiple hosts and machines, but often you’ll want to install on dedicated hardware.

Starting Docker: You can use given docker-compose file for intial setup or change it as per your needs.just copy paste the docker-file and up your docker-compose.
or for detailed tutorial you can refer https://debezium.io/documentation/reference/0.9/tutorial.html

once all the service are running check the db in mysql.

Check the sample DB
# mysql -u root -p 
Enter password: <enter your password> #debezium
mysql> use inventory
Database changed
mysql> show tables;
+---------------------+
| Tables_in_inventory |
+---------------------+
| addresses           |
| customers           |
| geom                |
| orders              |
| products            |
| products_on_hand    |
+---------------------+

  You should see in your connect container logs the typical output of Kafka, ending with:
  2017-09-21 07:21:14,912 INFO   ||  Kafka version : 0.11.0.0   [org.apache.kafka.common.utils.AppInfoParser]
2022-01-21 07:21:14,912 INFO   ||  Kafka commitId : cb8625948210849f   [org.apache.kafka.common.utils.AppInfoParser]
2022-01-21 07:21:14,929 INFO   ||  Discovered coordinator 172.17.0.4:9092 (id: 2147483646 rack: null) for group 1.   [org.apache.kafka.clients.consumer.internals.AbstractCoordinator]
2022-01-21 07:21:14,931 INFO   ||  Finished reading KafkaBasedLog for topic my_connect_configs   [org.apache.kafka.connect.util.KafkaBasedLog]
2022-01-21 07:21:14,932 INFO   ||  Started KafkaBasedLog for topic my_connect_configs   [org.apache.kafka.connect.util.KafkaBasedLog]
2022-01-21 07:21:14,932 INFO   ||  Started KafkaConfigBackingStore   [org.apache.kafka.connect.storage.KafkaConfigBackingStore]
2022-01-21 07:21:14,932 INFO   ||  Herder started   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2022-01-21 07:21:14,938 INFO   ||  Discovered coordinator 172.17.0.4:9092 (id: 2147483646 rack: null) for group 1.   [org.apache.kafka.clients.consumer.internals.AbstractCoordinator]
2022-01-21 07:21:14,940 INFO   ||  (Re-)joining group 1   [org.apache.kafka.clients.consumer.internals.AbstractCoordinator]
2022-01-21 07:21:15,022 INFO   ||  Successfully joined group 1 with generation 1   [org.apache.kafka.clients.consumer.internals.AbstractCoordinator]
2022-01-21 07:21:15,022 INFO   ||  Joined group and got assignment: Assignment{error=0, leader='connect-1-4d60cb71-cb93-4388-8908-6f0d299a9d94', leaderUrl='http://172.17.0.7:9092/', offset=-1, connectorIds=[], taskIds=[]}   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2022-01-21 07:21:15,023 INFO   ||  Starting connectors and tasks using config offset -1   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
2022-01-21 07:21:15,023 INFO   ||  Finished starting connectors and tasks   [org.apache.kafka.connect.runtime.distributed.DistributedHerder]
  
  
  Using the Kafka Connect REST API
The Kafka Connect service exposes a RESTful API to manage the set of connectors, so let’s use that API using the curl command line tool. Because we mapped port 8083 in the connect container (where the Kafka Connect service is running) to port 8083 on the Docker host, we can communicate to the service by sending the request to port 8083 on the Docker host, which then forwards the request to the Kafka Connect service. We are using localhost in our examples but users of non-native Docker platforms (like Docker Toolbox users on Windows and OS X) should replace localhost with the IP address of their Docker host.

Open a new terminal, and use it to check the status of the Kafka Connect service:

$ curl -H "Accept:application/json" localhost:8083/
The Kafka Connect service should return a JSON response message similar to the following:

{"version":"2.2.0","commit":"cb8625948210849f"}
This shows that we’re running Kafka Connect version 2.2.0. Next, check the list of connectors, again using your IP address in place of localhost:

$ curl -H "Accept:application/json" localhost:8083/connectors/
which should return the following:

[]
This confirms that the Kafka Connect service is running, that we can talk with it, and that it currently has no connectors. Let’s remedy that by starting a connector that will capture changes from our MySQL database.

Monitor the MySQL database
At this point we are running the Debezium services, a MySQL database server with a sample inventory database, and the MySQL command line client that is connected to our database. The next step is to register a connector that will begin monitoring the MySQL database server’s binlog and generate change events for each row that has been (or will be) changed. Since this is a new connector, when it starts it will start reading from the beginning of the MySQL binlog, which records all of the transactions, including individual row changes and changes to the schemas.

Normally we’d likely want to use the Kafka tools to manually create the necessary topics, including specifying the number of replicas. However, for this tutorial, Kafka is configured to automatically create the topics with just 1 replica.

Using the same terminal, we’ll use curl to submit to our Kafka Connect service a JSON request message with information about the connector we want to start. Since this command will not be in a Docker container, we need to use the IP address of our Docker host (so Docker Toolbox users on Windows and OS X should replace localhost with their IP address):
  
  curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql", "database.port": "3306", "database.user": "debezium", "database.password": "dbz", "database.server.id": "184054", "database.server.name": "dbserver1", "database.whitelist": "inventory", "database.history.kafka.bootstrap.servers": "kafka:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'
 
  This command uses the Kafka Connect service’s RESTful API to submit a POST request against /connectors resource with a JSON document that describes our new connector. Here’s the same JSON message in a more readable format:

{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "tasks.max": "1",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.server.id": "184054",
    "database.server.name": "dbserver1",
    "database.whitelist": "inventory",
    "database.history.kafka.bootstrap.servers": "kafka:9092",
    "database.history.kafka.topic": "schema-changes.inventory"
  }
}
The JSON message specifies the connector name as inventory-connector, and provides the detailed configuration properties for our MySQL connector
  
  We can even use the RESTful API to verify that our connector is included in the list of connectors:

$ curl -H "Accept:application/json" localhost:8083/connectors/
which should return the following:

["inventory-connector"]
  
  check the logs of connect
  we see a line reporting that the connector has transitioned from its snapshot mode into continuously reading the MySQL server’s binlog:

...
Sep 21, 2017 7:24:03 AM com.github.shyiko.mysql.binlog.BinaryLogClient connect
INFO: Connected to mysql:3306 at mysql-bin.000003/154 (sid:184054, cid:7)
2017-09-21 07:24:03,373 INFO   MySQL|dbserver1|binlog  Connected to MySQL binlog at mysql:3306, starting at binlog file 'mysql-bin.000003', pos=154, skipping 0 events plus 0 rows   [io.debezium.connector.mysql.BinlogReader]
2017-09-21 07:25:01,096 INFO   ||  Finished WorkerSourceTask{id=inventory-connector-0} commitOffsets successfully in 18 ms   [org.apache.kafka.connect.runtime.WorkerSourceTask]
...
  
  Viewing the change events
We saw in the connector’s output that events were written to five topics:

dbserver1

dbserver1.inventory.products

dbserver1.inventory.products_on_hand

dbserver1.inventory.customers

dbserver1.inventory.orders

As described in the MySQL connector documentation, each topic names start with dbserver1, which is the logical name we gave our connector. The first is our schema change topic to which all of the DDL statements are written. The remaining four topics are used to capture the change events for each of our four tables, and their topic names include the database name (e.g., inventory) and the table name.

Let’s look at all of the data change events in the dbserver1.inventory.customers topic. We’ll use the debezium/kafka Docker image to start a new container that runs one of Kafka’s utilities to watch the topic from the beginning of the topic:

$ docker run -it --name watcher --rm --link zookeeper:zookeeper --link kafka:kafka debezium/kafka:0.9 watch-topic -a -k dbserver1.inventory.customers
Again, we use the --rm flag since we want the container to be removed when it stops, and we use the -a flag on watch-topic to signal that we want to see all events since the beginning of the topic. (If we were to remove the -a flag, we’d see only the events that are recorded in the topic after we start watching.) The -k flag specifies that the output should include the event’s key, which in our case contains the row’s primary key. Here’s the output:

Using ZOOKEEPER_CONNECT=172.17.0.3:2181
Using KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://172.17.0.8:9092
Contents of topic dbserver1.inventory.customers:
Using the ConsoleConsumer with old consumer is deprecated and will be removed in a future major release. Consider using the new consumer by passing [bootstrap-server] instead of [zookeeper].
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1001}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":true,"field":"version"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":null,"after":{"id":1001,"first_name":"Sally","last_name":"Thomas","email":"sally.thomas@acme.com"},"source":{"version":"0.9.5.Final","name":"dbserver1","server_id":0,"ts_sec":0,"gtid":null,"file":"mysql-bin.000003","pos":154,"row":0,"snapshot":true,"thread":null,"db":"inventory","table":"customers"},"op":"c","ts_ms":1490634537160}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1002}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":true,"field":"version"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":null,"after":{"id":1002,"first_name":"George","last_name":"Bailey","email":"gbailey@foobar.com"},"source":{"version":"0.9.5.Final","name":"dbserver1","server_id":0,"ts_sec":0,"gtid":null,"file":"mysql-bin.000003","pos":154,"row":0,"snapshot":true,"thread":null,"db":"inventory","table":"customers"},"op":"c","ts_ms":1490634537160}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1003}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":true,"field":"version"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":null,"after":{"id":1003,"first_name":"Edward","last_name":"Walker","email":"ed@walker.com"},"source":{"version":"0.9.5.Final","name":"dbserver1","server_id":0,"ts_sec":0,"gtid":null,"file":"mysql-bin.000003","pos":154,"row":0,"snapshot":true,"thread":null,"db":"inventory","table":"customers"},"op":"c","ts_ms":1490634537160}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1004}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":true,"field":"version"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":null,"after":{"id":1004,"first_name":"Anne","last_name":"Kretchmar","email":"annek@noanswer.org"},"source":{"version":"0.9.5.Final","name":"dbserver1","server_id":0,"ts_sec":0,"gtid":null,"file":"mysql-bin.000003","pos":154,"row":0,"snapshot":true,"thread":null,"db":"inventory","table":"customers"},"op":"c","ts_ms":1490634537160}}
This utility keeps watching, so any new events would automatically appear as long as the utility keeps running. And this watch-topic utility is very simple and is limited in functionality and usefulness - we use it here simply to get an understanding of the kind of events that our connector generates. Applications that want to consume events would instead use Kafka consumers, and those consumer libraries offer far more flexibility and power. In fact, properly configured clients enable our applications to never miss any events, even when those applications crash or shutdown gracefullly.

These events happen to be encoded in JSON, since that’s how we configured our Kafka Connect service. Each event includes one JSON document for the key, and one for the value. Let’s look at the last event in more detail, by first reformatting the event’s key to be easier to read:

{
  "schema": {
    "type": "struct",
    "name": "dbserver1.inventory.customers.Key"
    "optional": false,
    "fields": [
      {
        "field": "id",
        "type": "int32",
        "optional": false
      }
    ]
  },
  "payload": {
    "id": 1004
  }
}
The event’s key has two parts: a schema and payload. The schema contains a Kafka Connect schema describing what is in the payload, and in our case that means that the payload is a struct named dbserver1.inventory.customers.Key that is not optional and has one required field named id of type int32.

If we look at the value of the key’s payload field, we’ll see that it is indeed a structure (which in JSON is just an object) with a single id field, whose value is 1004.

Therefore, we interpret this event as applying to the row in the inventory.customers table (output from the connector named dbserver1) whose id primary key column had a value of 1004.
  
  Now that we’re monitoring changes, what happens when we change one of the records in the database? Run the following statement in the MySQL command line client:

mysql> UPDATE customers SET first_name='Anne Marie' WHERE id=1004;
which produces the following output:

Query OK, 1 row affected (0.05 sec)
Rows matched: 1  Changed: 1  Warnings: 0
Rerun the select …​ statement to see the updated table:

mysql> select * from customers;
+------+------------+-----------+-----------------------+
| id   | first_name | last_name | email                 |
+------+------------+-----------+-----------------------+
| 1001 | Sally      | Thomas    | sally.thomas@acme.com |
| 1002 | George     | Bailey    | gbailey@foobar.com    |
| 1003 | Edward     | Walker    | ed@walker.com         |
| 1004 | Anne Marie | Kretchmar | annek@noanswer.org    |
+------+------------+-----------+-----------------------+
4 rows in set (0.00 sec)
Now, go back to the terminal running watch-topic and we should see a new fifth event:

{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1004}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":true,"field":"version"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":{"id":1004,"first_name":"Anne","last_name":"Kretchmar","email":"annek@noanswer.org"},"after":{"id":1004,"first_name":"Anne Marie","last_name":"Kretchmar","email":"annek@noanswer.org"},"source":{"version":"0.9.5.Final","name":"dbserver1","server_id":223344,"ts_sec":1490635059,"gtid":null,"file":"mysql-bin.000003","pos":364,"row":0,"snapshot":null,"thread":3,"db":"inventory","table":"customers"},"op":"u","ts_ms":1490635059389}}
Let’s reformat the new event’s key to be easier to read:

{
  "schema": {
    "type": "struct",
    "name": "dbserver1.inventory.customers.Key"
    "optional": false,
    "fields": [
      {
        "field": "id",
        "type": "int32",
        "optional": false
      }
    ]
  },
  "payload": {
    "id": 1004
  }
}
This key is exactly the same key as what we saw in the fourth record. Here’s that new event’s value formatted to be easier to read:

{
  "schema": {
    "type": "struct",
    "fields": [
      {
        "type": "struct",
        "fields": [
          {
            "type": "int32",
            "optional": false,
            "field": "id"
          },
          {
            "type": "string",
            "optional": false,
            "field": "first_name"
          },
          {
            "type": "string",
            "optional": false,
            "field": "last_name"
          },
          {
            "type": "string",
            "optional": false,
            "field": "email"
          }
        ],
        "optional": true,
        "name": "dbserver1.inventory.customers.Value",
        "field": "before"
      },
      {
        "type": "struct",
        "fields": [
          {
            "type": "int32",
            "optional": false,
            "field": "id"
          },
          {
            "type": "string",
            "optional": false,
            "field": "first_name"
          },
          {
            "type": "string",
            "optional": false,
            "field": "last_name"
          },
          {
            "type": "string",
            "optional": false,
            "field": "email"
          }
        ],
        "optional": true,
        "name": "dbserver1.inventory.customers.Value",
        "field": "after"
      },
      {
        "type": "struct",
        "fields": [
          {
            "type": "string",
            "optional": false,
            "field": "version"
          },
          {
            "type": "string",
            "optional": false,
            "field": "name"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "server_id"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "ts_sec"
          },
          {
            "type": "string",
            "optional": true,
            "field": "gtid"
          },
          {
            "type": "string",
            "optional": false,
            "field": "file"
          },
          {
            "type": "int64",
            "optional": false,
            "field": "pos"
          },
          {
            "type": "int32",
            "optional": false,
            "field": "row"
          },
          {
            "type": "boolean",
            "optional": true,
            "field": "snapshot"
          },
          {
            "type": "int64",
            "optional": true,
            "field": "thread"
          },
          {
            "type": "string",
            "optional": true,
            "field": "db"
          },
          {
            "type": "string",
            "optional": true,
            "field": "table"
          }
        ],
        "optional": false,
        "name": "io.debezium.connector.mysql.Source",
        "field": "source"
      },
      {
        "type": "string",
        "optional": false,
        "field": "op"
      },
      {
        "type": "int64",
        "optional": true,
        "field": "ts_ms"
      }
    ],
    "optional": false,
    "name": "dbserver1.inventory.customers.Envelope",
    "version": 1
  },
  "payload": {
    "before": {
      "id": 1004,
      "first_name": "Anne",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "after": {
      "id": 1004,
      "first_name": "Anne Marie",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "source": {
      "name": "0.9.5.Final",
      "name": "dbserver1",
      "server_id": 223344,
      "ts_sec": 1486501486,
      "gtid": null,
      "file": "mysql-bin.000003",
      "pos": 364,
      "row": 0,
      "snapshot": null,
      "thread": 3,
      "db": "inventory",
      "table": "customers"
    },
    "op": "u",
    "ts_ms": 1486501486308
  }
}
When we compare this to the value in the fourth event, we see no changes in the schema section and a couple of changes in the payload section:

The op field value is now u, signifying that this row changed because of an update

The before field now has the state of the row with the values before the database commit

The after field now has the updated state of the row, and here was can see that the first_name value is now Anne Marie.

The source field structure has many of the same values as before, except the ts_sec and pos fields have changed (and the file might have changed in other circumstances).

The ts_ms shows the timestamp that Debezium processed this event.

There are several things we can learn by just looking at this payload section. We can compare the before and after structures to determine what actually changed in this row because of the commit. The source structure tells us information about MySQL’s record of this change (providing traceability), but more importantly this has information we can compare to other events in this and other topics to know whether this event occurred before, after, or as part of the same MySQL commit as other events.

So far we’ve seen samples of create and update events. Now, let’s look at delete events. Since Anne Marie has not placed any orders, we can remove her record from our database using the MySQL command line client:

mysql> DELETE FROM customers WHERE id=1004;
In our terminal running watch-topic, we see two new events:

{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1004}}	{"schema":{"type":"struct","fields":[{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"before"},{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"},{"type":"string","optional":false,"field":"first_name"},{"type":"string","optional":false,"field":"last_name"},{"type":"string","optional":false,"field":"email"}],"optional":true,"name":"dbserver1.inventory.customers.Value","field":"after"},{"type":"struct","fields":[{"type":"string","optional":true,"field":"version"},{"type":"string","optional":false,"field":"name"},{"type":"int64","optional":false,"field":"server_id"},{"type":"int64","optional":false,"field":"ts_sec"},{"type":"string","optional":true,"field":"gtid"},{"type":"string","optional":false,"field":"file"},{"type":"int64","optional":false,"field":"pos"},{"type":"int32","optional":false,"field":"row"},{"type":"boolean","optional":true,"field":"snapshot"},{"type":"int64","optional":true,"field":"thread"},{"type":"string","optional":true,"field":"db"},{"type":"string","optional":true,"field":"table"}],"optional":false,"name":"io.debezium.connector.mysql.Source","field":"source"},{"type":"string","optional":false,"field":"op"},{"type":"int64","optional":true,"field":"ts_ms"}],"optional":false,"name":"dbserver1.inventory.customers.Envelope","version":1},"payload":{"before":{"id":1004,"first_name":"Anne Marie","last_name":"Kretchmar","email":"annek@noanswer.org"},"after":null,"source":{"version":"0.9.5.Final","name":"dbserver1","server_id":223344,"ts_sec":1490635100,"gtid":null,"file":"mysql-bin.000003","pos":725,"row":0,"snapshot":null,"thread":3,"db":"inventory","table":"customers"},"op":"d","ts_ms":1490635100301}}
{"schema":{"type":"struct","fields":[{"type":"int32","optional":false,"field":"id"}],"optional":false,"name":"dbserver1.inventory.customers.Key"},"payload":{"id":1004}}	{"schema":null,"payload":null}
What happened? We only deleted one row, but we now have two events. To understand what the MySQL connector does, let’s look at the first of our two new messages. Here’s the key reformatted to be easier to read:

{
  "schema": {
    "type": "struct",
    "name": "dbserver1.inventory.customers.Key"
    "optional": false,
    "fields": [
      {
        "field": "id",
        "type": "int32",
        "optional": false
      }
    ]
  },
  "payload": {
    "id": 1004
  }
}
Once again, this key is exactly the same key as in the previous two events we looked at. Here’s the value of the first new event, formatted to be easier to read:

{
  "schema": {...},
  "payload": {
    "before": {
      "id": 1004,
      "first_name": "Anne Marie",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    },
    "after": null,
    "source": {
      "name": "0.9.5.Final",
      "name": "dbserver1",
      "server_id": 223344,
      "ts_sec": 1486501558,
      "gtid": null,
      "file": "mysql-bin.000003",
      "pos": 725,
      "row": 0,
      "snapshot": null,
      "thread": 3,
      "db": "inventory",
      "table": "customers"
    },
    "op": "d",
    "ts_ms": 1486501558315
}
Again, the schema is identical to the previous messages, but the payload fragment has a few things of note:

The op field value is now d, signifying that this row was deleted

The before field now has the state of the row that was deleted with the database commit

The after field is null, signifying that the row no longer exists

The source field structure has many of the same values as before, except the ts_sec and pos fields have changed (and the file might have changed in other circumstances).

The ts_ms shows the timestamp that Debezium processed this event.

This event gives a consumer all kinds of information that it can use to process the removal of this row. We include the old values because some consumers might require them in order to properly handle the removal, and without it they may have to resort to far more complex behavior.
