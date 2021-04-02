# MySQL_CDC
### Change Data Capture for MySQL

Debezium is a CDC tool that can stream changes from MySQL, MongoDB, and PostgreSQL into Kafka, using Kafka Connect. We use it here to capture MySQL DB change in realtime when we make change from zabbix GUI

```
$ export DEBEZIUM_VERSION=1.4
$ docker-compose -f docker-compose-mysql.yaml up -d
$ docker-compose -f docker-compose-mysql.yaml ps

$ docker-compose -f docker-compose-mysql.yaml  down
```

Debezium uses MySQL’s binlog facility to extract events, and you need to configure MySQL to enable it. Here is the bare-basics necessary to get this working.

```
$ mysqladmin -h127.0.0.1 variables -uroot -p'secret' | grep log_bin
|log_bin                  | ON
```

Create debezium user with required permissions;

```
$ mysql -h127.0.0.1 -uroot -p'secret'
mysql> GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium' IDENTIFIED BY 'dbz';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'debezium'@'%';
```

Load the connector configuration into Kafka Connect using the REST API:

```
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" http://192.168.0.27:8083/connectors/ \
        -d '{
         "name": "zabbix-connector",
         "config": {
               "connector.class": "io.debezium.connector.mysql.MySqlConnector",
               "tasks.max": "1",
               "database.hostname": "192.168.0.27",
               "database.port": "3306",
               "database.user": "debezium",
               "database.password": "dbz",
               "database.server.id": "42",
               "database.server.name": "zabbix",
               "database.history.kafka.bootstrap.servers": "kafka:9092",
               "database.history.kafka.topic": "dbhistory.demo"
     }
}'
```

If everything config correctly you will see following output

```
$ curl -H "Accept:application/json" localhost:8083/connectors/
["zabbix-connector"]

$ curl -i -X GET -H "Accept:application/json" localhost:8083/connectors/zabbix-connector
HTTP/1.1 200 OK
Date: Mon, 29 Mar 2021 12:27:46 GMT
Content-Type: application/json
Content-Length: 494
Server: Jetty(9.4.33.v20201020)
```

You can check linked DB tables like this as well

```
$ docker exec -it tutorial_kafka_1 ./bin/kafka-topics.sh --zookeeper 192.168.0.27:2181 --list
zabbix.zabbix.actions
zabbix.zabbix.application_prototype
zabbix.zabbix.application_template
zabbix.zabbix.applications
zabbix.zabbix.auditlog
...
...
```

Now you can make change on zabbix GUI and see exactly how many tables context has been changed

```
$ docker exec -it mysqlcdc_kafka_1 ./bin/kafka-console-consumer.sh --bootstrap-server 192.168.0.27:9092 --whitelist '.*' | jq -c '(.payload)'

"payload": {
  "before": {
    "autoreg_tlsid": 1,
    "tls_psk_identity": "",
    "tls_psk": ""
  },
  "after": {
    "autoreg_tlsid": 1,
    "tls_psk_identity": "2020-03-29-zbxidentity1",
    "tls_psk": "XXXXXXXXXXXXXX"
  },
  "source": {
    "version": "1.4.2.Final",
    "connector": "mysql",
    "name": "zabbix",
    "ts_ms": 1617066859000,
    "snapshot": "false",
    "db": "zabbix",
    "table": "config_autoreg_tls",
    "server_id": 42,
    "gtid": null,
    "file": "mysql-bin.000003",
    "pos": 28327048,
    "row": 0,
    "thread": 8410,
    "query": null
  },
  "op": "u",
  "ts_ms": 1617066859612,
  "transaction": null
}
}
```
