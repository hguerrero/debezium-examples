# Debezium - Avro serialization with Red Hat OpenShift Service Registry

This tutorial demonstrates how to use [Debezium](https://debezium.io/) to monitor a MySQL database. As the data in the database changes, you will see the resulting event streams.

Debezium includes multiple connectors. In this tutorial, you will use the [MySQL connector](https://debezium.io/documentation/reference/1.3/connectors/mysql.html).

## Red Hat OpenShift Service Registry

[Red Hat OpenShift Service Registry](https://www.redhat.com/es/technologies/cloud-computing/openshift/openshift-service-registry) is a fully hosted and managed service that provides an API and schema registry for microservices. OpenShift Service Registry makes it easy for development teams to publish, discover, and reuse APIs and schemas.

Red Hat OpenShift Service Registry is included at no additional charge with these services:                                                                            

- [Red Hat OpenShift API Management](https://www.redhat.com/es/technologies/cloud-computing/openshift/openshift-api-management)
- [Red Hat OpenShift Streams for Apache Kafka](https://www.redhat.com/es/technologies/cloud-computing/openshift/openshift-streams-for-apache-kafka)

## Avro serialization

The default behavior is that the JSON converter includes the record’s message schema, which makes each record very verbose. Alternatively, you can serialize the record keys and values by using [Apache Avro](https://avro.apache.org/). To use Apache Avro serialization, you must deploy a schema registry that manages Avro message schemas and their versions.

OpenShift Service Registry provides an Avro converter that you can specify in Debezium connector configurations. This converter maps Kafka Connect schemas to Avro schemas. The converter then uses the Avro schemas to serialize the record keys and values into Avro’s compact binary form.

### Prerequisites

- Docker is installed and running.

  This tutorial uses Docker and the Linux container images to run the required local services. You should use the latest version of Docker. For more information, see the [Docker Engine installation documentation](https://docs.docker.com/engine/installation/).
  
- [kcat](https://github.com/edenhill/kcat)

- [kcctl](https://github.com/kcctl/kcctl)

- jq (for JSON processing)

## Starting the services

1. Clone this repository:

    ```bash
    git clone https://github.com/hguerrero/debezium-examples.git
    ```

1. Change to the following directory:

    ```bash
    cd debezium-examples/debezium-openshift-registry-avro
    ```

1. Start the environment

    ```bash
    docker-compose up -d
    ```

The last command will start the following components:

- Single node Kafka Connect cluster
- MySQL database (ready for CDC)

## Apicurio converters

Configuring Avro at the Debezium Connector involves specifying the converter and schema registry as a part of the connectors configuration. The connector configuration file configures the connector but explicitly sets the (de-)serializers for the connector to use Avro and specifies the location of the Apicurio registry.

> The container image used  in this environment includes all the required libraries to access the connectors and converters. 

### Configure the connector converters

The following are the lines required to set the **key** and **value** converters and their respective registry configuration:

```json
        "key.converter": "io.apicurio.registry.utils.converter.AvroConverter",
        "key.converter.apicurio.registry.converter.serializer": "io.apicurio.registry.serde.avro.AvroKafkaSerializer",
        "key.converter.apicurio.registry.url": "<your-service-registry-core-api-url>",
        "key.converter.apicurio.auth.service.url": "https://identity.api.openshift.com/auth",
        "key.converter.apicurio.auth.realm": "rhoas",
        "key.converter.apicurio.auth.client.id": "<registry-sa-client-id>",
        "key.converter.apicurio.auth.client.secret": "<registry-sa-client-id>",
        "key.converter.apicurio.registry.as-confluent": "true",
        "key.converter.apicurio.registry.auto-register": "true",
        "value.converter": "io.apicurio.registry.utils.converter.AvroConverter",
        "value.converter.apicurio.registry.converter.serializer": "io.apicurio.registry.serde.avro.AvroKafkaSerializer",
        "value.converter.apicurio.registry.url": "<your-service-registry-core-api-url>",
        "value.converter.apicurio.auth.service.url": "https://identity.api.openshift.com/auth",
        "value.converter.apicurio.auth.realm": "rhoas",
        "value.converter.apicurio.auth.client.id": "<registry-sa-client-id>",
        "value.converter.apicurio.auth.client.secret": "<registry-sa-client-id>",
        "value.converter.apicurio.registry.as-confluent": "true",
        "value.converter.apicurio.registry.auto-register": "true"
```

> The compatibility mode allows you to use other providers tooling to deserialize and reuse the schemas in the Apicurio service registry.

This also includes the information on required for the serializer to authenticate with the service registry using a service account.

### Create the topics in Red Hat OpenShift Streams for Apache Kafka

You will need to manually create the required topics that will be used by Debezium.

1. Create the following topics in your Kafka cluster:

    | Topic Name | Partitions    | Retention Time | Retention Size         |      |
    | ------------------------------------------------------------ | ---- | --------------------- | ----------------- | ---- |
    | avro | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.addresses | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.customers | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.geom | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.orders) | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.products | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.products_on_hand | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | debezium-cluster-configs | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | debezium-cluster-offsets | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | debezium-cluster-status | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | schema-changes.inventory | 1    | 604800000 ms (7 days) | 1 bytes (1 bytes) |      |

You should end with a table of topics like this:

![topics-openshift-streams-debezium.png](topics-openshift-streams-debezium.png)

### Create the connector

Let's create the Debezium connector to start capturing the changes of the database.

1. Configure kcctl context:

    ```sh
    kcctl config set-context --cluster http://localhost:8083 local
    ```
    
1. Create the connector using kcctl

    ```bash
    kcctl apply -f dbz-mysql-openshift-registry-avro.json
    ```

### Check the data

TODO

If you access the *Red Hat OpenShift Service Regtistry* console you should be able to find all the schema artifacts.

![registry-debezium-artifacts.png](registry-debezium-artifacts.png)

## Summary

Although Debezium makes it easy to capture database changes and record them in Kafka, one of the more important decisions you have to make is *how* those change events will be serialized in Kafka. Debezium allows you to select key and value *converters* to select from different type of options. The *Red Hat OpenShift Service Registry* allows you to store externalized versions of the schema to minimize the payload to propagate.

