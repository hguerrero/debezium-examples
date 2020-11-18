# Debezium - Avro

This tutorial demonstrates how to use [Debezium](https://debezium.io/) to monitor a MySQL database. As the data in the database changes, you will see the resulting event streams.

Debezium includes multiple connectors. In this tutorial, you will use the [MySQL connector](https://debezium.io/documentation/reference/1.3/connectors/mysql.html).

The default behavior is that the JSON converter includes the record’s message schema, which makes each record very verbose. Alternatively, you can serialize the record keys and values by using [Apache Avro](https://avro.apache.org/). To use Apache Avro serialization, you must deploy a schema registry that manages Avro message schemas and their versions.

The [Apicurio Registry](https://github.com/Apicurio/apicurio-registry) open-source project provides several components that work with Avro:

- An Avro converter that you can specify in Debezium connector configurations. This converter maps Kafka Connect schemas to Avro schemas. The converter then uses the Avro schemas to serialize the record keys and values into Avro’s compact binary form.

- An API and schema registry that tracks:

  - Avro schemas that are used in Kafka topics
  - Where the Avro converter sends the generated Avro schemas

## Prerequisites

- Docker is installed and running.

  This tutorial uses Docker and the Linux container images to run the required services. You should use the latest version of Docker. For more information, see the [Docker Engine installation documentation](https://docs.docker.com/engine/installation/).

## Starting the services

1. Clone this repository

  `git clone https://github.com/hguerrero/amq-examples.git`