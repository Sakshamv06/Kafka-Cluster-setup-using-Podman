# Kafka Cluster Setup with Podman

## Table of Contents
1. [Overview](#1-overview)
2. [Why Use This Setup?](#2-why-use-this-setup)
   - [Apache Kafka](#21-apache-kafka)
   - [Zookeeper](#22-zookeeper)
   - [Kafka Exporter](#23-kafka-exporter)
   - [Prometheus](#24-prometheus)
   - [Grafana](#25-grafana)
3. [Network and Container Setup](#3-network-and-container-setup)
   - [Create a Network](#31-create-a-network)
4. [Zookeeper Container](#4-zookeeper-container)
5. [Kafka Brokers](#5-kafka-brokers)
   - [Kafka1 Broker](#51-kafka1-broker)
   - [Kafka2 Broker](#52-kafka2-broker)
6. [Producing and Consuming Messages](#6-producing-and-consuming-messages)
   - [Producer on Kafka2](#61-producer-on-kafka2)
   - [Consumer on Kafka1](#62-consumer-on-kafka1)
7. [Monitoring](#7-monitoring)
   - [Kafdrop](#71-kafdrop)
   - [Kafka Exporter](#72-kafka-exporter)
   - [Prometheus](#73-prometheus)
   - [Grafana](#74-grafana)

## 1. Overview
This document details the setup of a Kafka cluster using Podman containers. It covers the purpose, configuration, and explanation of each container involved, along with how to produce, consume, and monitor Kafka messages using Prometheus, Grafana, and Kafdrop.

## 2. Why Use This Setup?

### 2.1 Apache Kafka
Apache Kafka is a distributed event streaming platform used for building real-time data pipelines and streaming applications. It is designed to handle large-scale data streams efficiently. The main use cases include:
- **Message Broker**: Acts as a buffer between services, enabling asynchronous communication.
- **Real-Time Data Streaming**: Kafka streams real-time event data to analytics systems or databases.
- **Data Integration**: Used for data replication and integration between different systems.
- **Event Sourcing**: Kafka records every event as a log entry, making it ideal for event-driven architectures.

### 2.2 Zookeeper
Zookeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and group services. In Kafka:
- **Broker Coordination**: Keeps track of the status of Kafka brokers and coordinates them.
- **Metadata Management**: Manages metadata for Kafka topics and partitions.
- **Leader Election**: Ensures that only one broker serves as the leader for a partition at a time.

### 2.3 Kafka Exporter
Kafka Exporter is a tool used to extract metrics from Kafka brokers and expose them in a format compatible with Prometheus. Its benefits include:
- **Monitoring Broker Health**: Tracks broker health and performance.
- **Consumer Group Metrics**: Monitors consumer lag and group offsets.
- **Exporting Metrics**: Makes Kafka metrics available to Prometheus.

### 2.4 Prometheus
Prometheus is an open-source system monitoring and alerting toolkit. In this setup, it:
- **Scrapes Kafka Metrics**: Collects metrics from Kafka Exporter.
- **Stores Time-Series Data**: Stores Kafka metrics in a time-series format.
- **Alerting and Visualization**: Enables alerting and serves as a data source for Grafana.

### 2.5 Grafana
Grafana is an open-source platform for monitoring and observability. It allows you to visualize metrics collected by Prometheus. Its role includes:
- **Visualizing Kafka Metrics**: Creates dashboards with Kafka metrics.
- **Real-Time Monitoring**: Displays broker performance and consumer lag in real time.
- **Custom Dashboards**: Provides flexible and interactive data visualization.

### 2.6 Why Are We Doing This?
- **Real-Time Data Processing**: Kafka allows processing large volumes of real-time data with fault tolerance.
- **Distributed and Scalable**: Kafka can handle massive data streams with high availability.
- **Monitoring and Visualization**: Prometheus and Grafana provide insight into Kafka's health and performance.
- **Simplified Management**: Kafdrop offers a UI for managing Kafka clusters easily.

## 3. Network and Container Setup
### 3.1 Create a Network
All the containers will communicate over the same custom network called `kafka-network`.
```sh
podman network create kafka-network
```

## 4. Zookeeper Container
### 4.1 Command
```sh
podman run -d --rm \
    --name zookeeper \
    --network kafka-network \
    -p 2181:2181 \
    -e ALLOW_ANONYMOUS_LOGIN=yes \
    docker.io/wurstmeister/zookeeper
```

### 4.2 Explanation
- **Zookeeper**: Coordinates and manages Kafka brokers.
- **Port 2181**: Zookeeper's client connection port.
- **ALLOW_ANONYMOUS_LOGIN=yes**: Allows unauthenticated connections for simplicity (not recommended for production).

## 5. Kafka Brokers
### 5.1 Kafka1 Broker
```sh
podman run -d \
    --name kafka1 \
    --network kafka-network \
    -p 9092:9092 \
    -p 7071:7071 \
    -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka1:9092 \
    -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
    -e EXTRA_ARGS=-javaagent:/opt/kafka/prometheus/jmx_prometheus_javaagent-0.18.0.jar=7071:/opt/kafka/prometheus/prom-JMX-agent-config.yml \
    -v /home/saksham/praveen/kafka-jmx-exporter/jmx_prometheus_javaagent-0.18.0.jar:/opt/kafka/prometheus/jmx_prometheus_javaagent-0.18.0.jar \
    -v /home/saksham/praveen/kafka-jmx-exporter/prom-JMX-agent-config.yml:/opt/kafka/prometheus/prom-JMX-agent-config.yml \
    docker.io/wurstmeister/kafka
```

### 5.2 Kafka2 Broker
```sh
podman run -d \
    --name kafka2 \
    --network kafka-network \
    -p 9093:9093 \
    -p 7072:7072 \
    -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka2:9093 \
    -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9093 \
    -e EXTRA_ARGS=-javaagent:/opt/kafka/prometheus/jmx_prometheus_javaagent-0.18.0.jar=7072:/opt/kafka/prometheus/prom-JMX-agent-config.yml \
    -v /home/saksham/praveen/kafka-jmx-exporter/jmx_prometheus_javaagent-0.18.0.jar:/opt/kafka/prometheus/jmx_prometheus_javaagent-0.18.0.jar \
    -v /home/saksham/praveen/kafka-jmx-exporter/prom-JMX-agent-config.yml:/opt/kafka/prometheus/prom-JMX-agent-config.yml \
    docker.io/wurstmeister/kafka
```

### 5.3 Explanation
- **Kafka**: Open-source event streaming platform.
- **Ports**:
  - `9092`: Kafka1 broker listener port.
  - `7071`: JMX exporter port for Kafka1.
  - `9093`: Kafka2 broker listener port.
  - `7072`: JMX exporter port for Kafka2.
- **JMX Exporter**: Exposes Kafka metrics for Prometheus monitoring.

## 6. Producing and Consuming Messages
### 6.1 Producer on Kafka2
```sh
podman exec -it kafka2 bash
kafka-console-producer.sh --bootstrap-server kafka2:9093 --topic Saksham
```

### 6.2 Consumer on Kafka1
```sh
podman exec -it kafka1 bash
kafka-console-consumer.sh --bootstrap-server kafka1:9092 --topic Saksham --from-beginning
```

## 7. Monitoring with Kafdrop, Kafka Exporter, Prometheus, and Grafana
### 7.1 Kafdrop
```sh
podman run -d \
    --name kafdrop \
    --network kafka-network \
    -p 9000:9000 \
    -e KAFKA_BROKERCONNECT=kafka1:9092,kafka2:9093 \
    --restart always \
    obsidiandynamics/kafdrop
```

### 7.2 Kafka Exporter
```sh
podman run -d --name kafka-exporter --network kafka-network -p 9308:9308 docker.io/danielqsj/kafka-exporter --kafka.server=kafka1:9092 --kafka.server=kafka2:9093
```

### 7.3 Prometheus
```sh
podman run -d \
    --name prometheus \
    --network kafka-network \
    -p 9090:9090 \
    -v ./prometheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus
```

### Prometheus Configuration
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'kafka1'
    static_configs:
      - targets: ['192.168.122.1:7071']

  - job_name: 'kafka2'
    static_configs:
      - targets: ['192.168.122.1:7072']

  - job_name: 'kafka-exporter'
    static_configs:
      - targets: ['192.168.122.1:9308']

```

### 7.4 Grafana
```sh
podman run -d \
    --name grafana \
    --network kafka-network \
    -p 3000:3000 \
    grafana/grafana
```

## Explanation
- **Prometheus**: Monitors Kafka metrics.
- **Port 9090**: Access via `http://localhost:9090`.
- **Config File**: Specifies Kafka Exporter as the target.
- **Grafana**: Visualization tool.
- **Port 3000**: Access via `http://localhost:3000`.
- **Data Source**: Add Prometheus as a data source in Grafana to view Kafka metrics.
