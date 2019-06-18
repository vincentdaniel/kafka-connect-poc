# Kafka Connect POC

POC to run a stream of json from Kinesis to S3/GCS.
Files are being written every minute or every 1000 messages. Folders are partitioned every minute.

## Setup

- Download the "licensed" connectors and extract them in the connect folder (if versions have changed, edit the Dockerfile)
    - Kinesis -> Kafka - https://www.confluent.io/connector/kafka-connect-kinesis/#download
    - Kafka - GCS - https://www.confluent.io/connector/kafka-connect-gcs/#download
- Put your access/secret keys for AWS in `docker-compose.yml` environment variables `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`
    - Create an S3 bucket (and change its name in the S3 connector config below)
    - The account should have access to the kinesis stream as well as the S3 bucket
- Put your gcs credentials in json format in gcs.json file
    - Create a GCS bucket (and change its name in the GCS connector config below)
    - The account should have access to the GCS bucket
- `docker-compose up -d`
- `docker exec -it kafka-connect-poc_kafka_1 kafka-topics --create --zookeeper "localhost:2181" --replication-factor 1 --partitions 1 --topic events_topic`
- Go to http://localhost:3030 and wait for connectors to be available

## Tear down

- `docker-compose down`

## Debug

- Logs: http://localhost:3030/logs/connect-distributed.log or `docker exec -it kafka-connect-poc_kafka_1 tail -f /var/log/connect-distributed.log`

## Create connectors

### Kinesis -> Kafka connector

```
curl -X POST \
  http://localhost:8083/connectors \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -d '{
        "name": "kinesis-source",
        "config": {
            "connector.class": "io.confluent.connect.kinesis.KinesisSourceConnector",
            "tasks.max": "1",
            "kafka.topic" : "events_topic",
            "kinesis.stream" : "kinesis-test-stream",
            "kinesis.position": "LATEST",
            "kinesis.empty.records.backoff.ms": "1000",
            "errors.log.enable": "true",
            "confluent.topic.bootstrap.servers": "localhost:9092",
            "confluent.topic.replication.factor": "1",
            "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter"
        }
    }'
```

### Kafka -> S3 connector

```
curl -X POST \
  http://localhost:8083/connectors \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -d '{
        "name": "s3-sink",
        "config": {
            "connector.class": "io.confluent.connect.s3.S3SinkConnector",
            "flush.size": "1000",
            "rotate.schedule.interval.ms": "60000",
            "topics": "events_topic",
            "tasks.max": "1",
            "s3.region": "us-east-1",
            "s3.bucket.name": "confluent-kafka-connect-s3-testing1",
            "s3.part.size": "5242880",
            "format.class": "io.confluent.connect.s3.format.bytearray.ByteArrayFormat",
            "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
            "partitioner.class":"io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
            "timestamp.extractor": "Record",
            "partition.duration.ms": "60000",
            "path.format": "YYYY-MM-dd-HH-mm-00",
            "locale": "US",
            "timezone": "UTC",
            "storage.class": "io.confluent.connect.s3.storage.S3Storage",
            "schema.compatibility": "NONE",
            "errors.tolerance": "all",
            "errors.log.enable": "true"
        }
    }'
```

### Kafka -> GCS connector

```
curl -X POST \
  http://localhost:8083/connectors \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -d '{
        "name": "gcs-sink",
        "config": {
            "connector.class": "io.confluent.connect.gcs.GcsSinkConnector",
            "flush.size": "1000",
            "rotate.schedule.interval.ms": "60000",
            "topics": "events_topic",
            "tasks.max": "1",
            "gcs.bucket.name": "confluent-kafka-connect-gcs-testing1",
            "gcs.part.size": "5242880",
            "gcs.credentials.path": "/opt/gcs.json",
            "format.class": "io.confluent.connect.gcs.format.bytearray.ByteArrayFormat",
            "value.converter": "org.apache.kafka.connect.converters.ByteArrayConverter",
            "partitioner.class":"io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
            "timestamp.extractor": "Record",
            "partition.duration.ms": "60000",
            "path.format": "YYYY-MM-dd-HH-mm-00",
            "locale": "US",
            "timezone": "UTC",
            "storage.class": "io.confluent.connect.gcs.storage.GcsStorage",
            "schema.compatibility": "NONE",
            "errors.tolerance": "all",
            "errors.log.enable": "true",
            "confluent.topic.bootstrap.servers": "localhost:9092",
            "confluent.topic.replication.factor": "1"
        }
    }'
```
