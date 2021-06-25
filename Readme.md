# ksqlDB Workshop

This repository contains the starter code for the **ksqlDB** workshop.

## Overview

This workshop will demonstrate how to integrate ksqlDB with an external data source (Elasticsearch) to build **Driver
Location Tracking & Monitoring** app.

## Getting Started

Use Docker Compose ([docker-compose.yml](docker/docker-compose.yml)) for setting up and running the infrastructure,
which includes ksqlDB and Kafka Connect in embedded mode.

The following tools are available when you run the whole infrastructure:

* [Apache Kafka](https://kafka.apache.org/)
* [Zookeeper](https://zookeeper.apache.org/)
* [ksqlDB](https://docs.ksqldb.io/en/latest/reference/)
* [Kafka Connect](https://kafka.apache.org/documentation/#connect)
* [Schema Registry](https://docs.confluent.io/platform/current/schema-registry/index.html)
* [Kowl UI](https://github.com/cloudhut/kowl)
* [Kafka Connect UI](https://github.com/lensesio/kafka-connect-ui)
* [Elasticsearch](https://www.elastic.co/elasticsearch/)
* [Kibana](https://www.elastic.co/kibana)

### Prerequisite

* Docker

### Usage

#### 1. Start Docker containers

```shell
cd docker
docker compose up -d
```

#### 2. Check if all components are up and running

```shell
docker compose ps

# NAME                SERVICE             STATUS              PORTS
# elasticsearch       elasticsearch       running             0.0.0.0:9300->9300/tcp, :::9300->9300/tcp, 0.0.0.0:9200->9200/tcp, :::9200->9200/tcp
# kafka               kafka               running             0.0.0.0:9092->9092/tcp, :::9092->9092/tcp, 0.0.0.0:9101->9101/tcp, :::9101->9101/tcp
# kafka-connect-ui    kafka-connect-ui    running             0.0.0.0:8000->8000/tcp, :::8000->8000/tcp
# kibana              kibana              running             0.0.0.0:5601->5601/tcp, :::5601->5601/tcp
# kowl                kowl                running             0.0.0.0:8080->8080/tcp, :::8080->8080/tcp
# ksqldb-cli          ksqldb-cli          running             
# ksqldb-server       ksqldb-server       running             0.0.0.0:8083->8083/tcp, :::8083->8083/tcp, 0.0.0.0:8088->8088/tcp, :::8088->8088/tcp
# schema-registry     schema-registry     running             0.0.0.0:8081->8081/tcp, :::8081->8081/tcp
# zookeeper           zookeeper           running             0.0.0.0:2181->2181/tcp, :::2181->2181/tcp, 2888/tcp, 3888/tcp
```

#### 3. Start ksqlDB's interactive CLI

```shell
docker compose exec ksqldb-cli bash ksql http://ksqldb-server:8088
```

#### 4. Create driver profile table in ksqlDB

```shell
CREATE TABLE driver_profile_table (driver_id BIGINT PRIMARY KEY, name VARCHAR)
  WITH ( kafka_topic='driver.profile.events', value_format='AVRO', partitions=6);
```

#### 5. Create stream for driver location events

```shell
CREATE STREAM driver_location_stream (driver_id BIGINT KEY, location STRUCT<lat DOUBLE, lon DOUBLE>)
  WITH (kafka_topic='driver.location.events', value_format='AVRO', partitions=6);
```

#### 6. Enrich driver_location stream by joining with driver_profile table

```shell
CREATE STREAM enriched_driver_location_stream  WITH (
  kafka_topic='driver.location.enriched.events',
  value_format='AVRO') AS
  SELECT
    driver_profile.driver_id       AS driver_id,
    driver_profile.name            AS driver_name,
    CAST(location->lat AS STRING) + ',' + CAST(location->lon AS STRING) AS location
  FROM driver_location_stream driver_location JOIN driver_profile_table driver_profile
    ON driver_location.driver_id = driver_profile.driver_id
  EMIT CHANGES;
```

#### 7. List defined streams

```shell
show streams;

#  Stream Name                     | Kafka Topic                        | Key Format | Value Format | Windowed 
# -------------------------------------------------------------------------------------------------------------
#  DRIVER_LOCATION_STREAM          | driver.location.events             | KAFKA      | AVRO         | false    
#  ENRICHED_DRIVER_LOCATION_STREAM | driver.location.enriched.events    | KAFKA      | AVRO         | false    
#  KSQL_PROCESSING_LOG             | ksql_connect_01ksql_processing_log | KAFKA      | JSON         | false    
# -------------------------------------------------------------------------------------------------------------
```

#### 8. List defined tables

```shell
show tables;

#  Table Name           | Kafka Topic           | Key Format | Value Format | Windowed 
# -------------------------------------------------------------------------------------
#  DRIVER_PROFILE_TABLE | driver.profile.events | KAFKA      | AVRO         | false    
# -------------------------------------------------------------------------------------
```

#### 9. Write data to driver profile table

```shell
INSERT INTO driver_profile_table (driver_id, name) VALUES (1, 'Driver 1');
INSERT INTO driver_profile_table (driver_id, name) VALUES (2, 'Driver 2');
INSERT INTO driver_profile_table (driver_id, name) VALUES (3, 'Driver 3');
INSERT INTO driver_profile_table (driver_id, name) VALUES (4, 'Driver 4');
INSERT INTO driver_profile_table (driver_id, name) VALUES (5, 'Driver 5');
INSERT INTO driver_profile_table (driver_id, name) VALUES (6, 'Driver 6');
```

#### 10. Write data to driver location stream

```shell
INSERT INTO driver_location_stream (driver_id, location) VALUES (1, STRUCT(lat := 50.055355899352136, lon := 19.935877982646776));
INSERT INTO driver_location_stream (driver_id, location) VALUES (2, STRUCT(lat := 50.061660189205384, lon := 19.92373795331377));
INSERT INTO driver_location_stream (driver_id, location) VALUES (3, STRUCT(lat := 50.06767789514652, lon := 19.913912678521136));
INSERT INTO driver_location_stream (driver_id, location) VALUES (4, STRUCT(lat := 49.984300646662135, lon := 20.053662685583486));
INSERT INTO driver_location_stream (driver_id, location) VALUES (5, STRUCT(lat := 50.05768283673512, lon := 19.85473522517211));
INSERT INTO driver_location_stream (driver_id, location) VALUES (6, STRUCT(lat := 50.06721095767255, lon := 19.87215179908983));
```

#### 11. Print the raw data from `driver.profile.events` topic

```shell
PRINT 'driver.profile.events' FROM BEGINNING;
```

#### 12. Tell ksqlDB to start all queries from `earliest` point in each topic

```shell
SET 'auto.offset.reset' = 'earliest';
```

#### 13. Run a continuous query

```shell
SELECT * from enriched_driver_location_stream EMIT CHANGES;
```

#### 14. Exit ksqlDB interactive CLI

```shell
exit

# ksql> exit
# Exiting ksqlDB.
```

#### 15. Check if ElasticSearch is up and running

```shell
curl localhost:9200

# {
#   "name" : "74c66e90824c",
#   "cluster_name" : "docker-cluster",
#   "cluster_uuid" : "CPyD4K3QT4KOqs-ut9HPCA",
#   "version" : {
#     "number" : "7.8.0",
#     "build_flavor" : "default",
#     "build_type" : "docker",
#     "build_hash" : "757314695644ea9a1dc2fecd26d1a43856725e65",
#     "build_date" : "2020-06-14T19:35:50.234439Z",
#     "build_snapshot" : false,
#     "lucene_version" : "8.5.1",
#     "minimum_wire_compatibility_version" : "6.8.0",
#     "minimum_index_compatibility_version" : "6.0.0-beta1"
#   },
#   "tagline" : "You Know, for Search"
# }
```

#### 16. Add dynamic template for `driver.location.enriched.events`

```shell
curl -X PUT "localhost:9200/driver.location.enriched.events/?pretty" -H 'Content-Type: application/json' -d'
{
"mappings": {
"dynamic_templates": [{
"locations": {
"mapping": {
"type": "geo_point"
},
"match": "*LOCATION"
}}]}}
'

# {
#   "acknowledged" : true,
#   "shards_acknowledged" : true,
#   "index" : "driver.location.enriched.events"
# }
```

#### 17. Import Kibana 'Driver live-location' dashboard

Open your web browser and go to Kibana
[Saved Objects page](http://localhost:5601/app/kibana#/management/kibana/objects).

Import [driver_live_location.ndjson](kibana/driver_live_location.ndjson) file.

#### 18. Create elasticsearch sink connector

Start ksqlDB's interactive CLI

```shell
docker compose exec ksqldb-cli bash ksql http://ksqldb-server:8088
```

In the ksqlDB CLI, run the following command to create sink connector.

```shell
CREATE SOURCE CONNECTOR elasticsearch_sink_demo WITH (
'connector.class'                       = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
'connection.url'                        = 'http://elasticsearch:9200',
'topics'                                = 'driver.location.enriched.events',
'type.name'                             = '_doc',
'errors.log.enable'                     = 'true',
'schema.registry.url'                   = 'http://schema-registry:8081',
'key.converter'                         = 'org.apache.kafka.connect.storage.StringConverter',
'value.converter'                       = 'io.confluent.connect.avro.AvroConverter',
'value.converter.schema.registry.url'   = 'http://schema-registry:8081'
); 
```

When the sink connector is created, it exports any data matching the specified `topics`.

#### 19. Describe created connector

```shell
DESCRIBE connector elasticsearch_sink_demo;
```

#### 20. Check Kibana dashboard

Open your web browser and go to Kibana [Maps page](http://localhost:5601/app/maps) then open `Driver live-location`.

Update `Driver 1` position, by running following command.

```shell
INSERT INTO driver_location_stream (driver_id, location) VALUES (1, STRUCT(lat := 50.078682625477754, lon := 19.7877404489687));
```

Use `Driver live location` Layer -> **Fit to Data** to move to current location of driver in Kibana dashboard.

#### 21. Experiment with ksqlDB queries and build awesome stream apps.

This workshop tutorial was just the beginning and only touched some basic concepts and useful commands that can be used during ksqlDB development.

Below few useful links that can help improve Your knowledge about ksqlDB.

* [Introducing ksqlDB](https://www.confluent.io/blog/intro-to-ksqldb-sql-database-streaming/)
* [An introduction to ksqlDB](https://www.youtube.com/watch?v=7mGBxG2NhVQ)
* [ksqlDB HOWTO: Joins](https://www.youtube.com/watch?v=_0Ktp2eB-as)
* [ksqlDB HOWTO: Handling Time](https://www.youtube.com/watch?v=scpbbl71CD8)
* [ksqlDB HOWTO: Split and Merge Kafka Topics](https://www.youtube.com/watch?v=5NoU7D4OGA0)
* [4 Incredible ksqlDB Techniques (#2 Will Make You Cry)](https://www.confluent.io/blog/ksqldb-techniques-that-make-stream-processing-easier-than-ever/)

#### 22. Destroy demo infrastructure

When you're done, stop Docker containers by running.

```shell
docker compose down -v
```

## Important Endpoints

| Name | Endpoint | 
| -------------:|:--------:|
| `Ksql` | [http://localhost:8083/](http://localhost:8083/) |
| `Ksql - Embedded Kafka Connect` | [http://localhost:8088/](http://localhost:8088/) |
| `Kafka Connect UI` | [http://localhost:8000/](http://localhost:8000/) |
| `Kowl UI` | [http://localhost:8080/](http://localhost:8080/) |
| `Schema-registry` | [http://localhost:8081/](http://localhost:8081/) |
| `Elasticsearch` | [http://localhost:9200/](http://localhost:9200/) |
| `Kibana` | [http://localhost:5601/](http://localhost:5601/) |

## References

* [ksqlDB](https://docs.ksqldb.io/en/latest/reference/)
* [Kafka Docker Images](https://github.com/confluentinc/kafka-images)
* [Elasticsearch Sink Connector](https://docs.confluent.io/kafka-connect-elasticsearch/current/index.html)
* [Confluent Hub](https://www.confluent.io/hub/)
* [Kafka Connect UI](https://github.com/lensesio/kafka-connect-ui)
* [cloudhut/kowl](https://github.com/cloudhut/kowl)
* [How to Visualize Geo Data on a Map with Kibana - Version 7.10](https://www.youtube.com/watch?v=slwlQQLesKA)

## License

Distributed under the MIT License. See `LICENSE` for more information.