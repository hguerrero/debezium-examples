# Debezium - Avro serialization with Red Hat OpenShift Service Registry

This tutorial demonstrates how to use[ Debezium](https://debezium.io/) to monitor a MySQL database. As the data in the database changes, you will see the resulting event streams reflected in[ Red Hat OpenShift Streams for Apache Kafka](https://www.redhat.com/es/technologies/cloud-computing/openshift/openshift-streams-for-apache-kafka).

Debezium includes multiple connectors. In this tutorial, you will use the[ MySQL connector](https://debezium.io/documentation/reference/1.3/connectors/mysql.html).

## Red Hat OpenShift Service Registry

[Red Hat OpenShift Service Registry](https://www.redhat.com/es/technologies/cloud-computing/openshift/openshift-service-registry) is a fully hosted and managed service that provides an API and schema registry for microservices. OpenShift Service Registry makes it easy for development teams to publish, discover, and reuse APIs and schemas.The following services include Red Hat OpenShift Service Registry at no additional charge:                                      

- [Red Hat OpenShift API Management](https://www.redhat.com/es/technologies/cloud-computing/openshift/openshift-api-management)
- [Red Hat OpenShift Streams for Apache Kafka](https://www.redhat.com/es/technologies/cloud-computing/openshift/openshift-streams-for-apache-kafka)

## Debezium schema serialization

The default behavior is that the JSON converter includes the record’s message schema, making each record very verbose. Alternatively, you can serialize the record keys and values using[ Apache Avro](https://avro.apache.org/). To use Apache Avro serialization, you must deploy a schema registry that manages Avro message schemas and their versions.

OpenShift Service Registry provides an Avro converter that you can specify in Debezium connector configurations. This converter maps Kafka Connect schemas to Avro schemas. The converter then uses the Avro schemas to serialize the record keys and values into Avro’s compact binary form.

### Prerequisites

- Docker is installed and running.

  This tutorial uses Docker and the Linux container images to run the essential local services. You should use the latest version of Docker. For more information, see the[ Docker Engine installation documentation](https://docs.docker.com/engine/installation/).

- [kcat](https://github.com/edenhill/kcat)

- [kcctl](https://github.com/kcctl/kcctl)

- [rhoas](https://github.com/redhat-developer/app-services-cli/releases/latest) - Red Hat Openshift Application Services CLIjq (for JSON processing)

- A Red Hat developer account

  As part of the developer program for OpenShift Streams for Apache Kafka, everybody with a Red Hat account can create a Kafka instance free of charge.

- A running [OpenShift Streams cluster](https://developers.redhat.com/articles/2021/07/07/getting-started-red-hat-openshift-streams-apache-kafka#)

- A running [OpenShfit Service Registry instance](https://developers.redhat.com/articles/2021/10/11/get-started-openshift-service-registry)

## Starting the local services

For this demo, the MySQL database and the Kafka Connect cluster will be running locally on your machine. We will use Docker compose to start the required services, so there is no need to install anything beyond the prerequisites listed above.

To start the local services, follow these steps:

1. Clone this repository:

    ```bash
    git clone https://github.com/hguerrero/debezium-examples.git
    ```

1. Change to the following directory:

    ```bash
    cd debezium-examples/debezium-openshift-registry-avro
    ```

1. Open the `docker-compose.yaml` file and edit the following properties:

    ```yaml
          KAFKA_CONNECT_BOOTSTRAP_SERVERS: <your-boostrap-server>:<port>
          KAFKA_CONNECT_TLS: 'true'
          KAFKA_CONNECT_SASL_MECHANISM: plain
          KAFKA_CONNECT_SASL_USERNAME: <kafka-sa-client-id>
          KAFKA_CONNECT_SASL_PASSWORD_FILE: cpass
    ```
    
    > You will need your Kafka bootstrap server and the service account you will use to connect. The image then takes the password from a file called `cpass`
    
1. Open the provided `cpass` file and replace the placeholder with your service account secret.

    ```
    <kafka-sa-client-secret>
    ```

1. Start the environment:

    ```bash
    docker-compose up -d
    ```

The last command will start the following components:

- Single node Kafka Connect cluster
- MySQL database (ready for CDC)

## Apicurio converters

The open source[ Apicurio Registry project](https://www.apicur.io/registry/) is the original community of OpenShift Service Registry. Apicurio Registry provides Kafka Connect converters for Apache Avro and JSON Schema. Configuring Avro at the Debezium Connector involves specifying the converter and schema registry as a part of the connector's configuration. The connector configuration customizes the connector but explicitly sets the (de-)serializers for the connector to use Avro and specifies the location of the Apicurio registry.

> The container image used in this environment includes all the required libraries to access the Debezium connectors and Apicurio Registry converters.

### Configure the converters

The following are the lines required in the connector configuration to set the **key** and **value** converters and their respective registry configuration. Replace the corresponding values with the information from your OpenShift services.

```json
        "key.converter": "io.apicurio.registry.utils.converter.AvroConverter",
        "key.converter.apicurio.registry.converter.serializer": "io.apicurio.registry.serde.avro.AvroKafkaSerializer",
        "key.converter.apicurio.registry.url": "<your-service-registry-core-api-url>",
        "key.converter.apicurio.auth.service.url": "https://identity.api.openshift.com/auth",
        "key.converter.apicurio.auth.realm": "rhoas",
        "key.converter.apicurio.auth.client.id": "<registry-sa-client-id>",
        "key.converter.apicurio.auth.client.secret": "<registry-sa-client-secret>",
        "key.converter.apicurio.registry.as-confluent": "true",
        "key.converter.apicurio.registry.auto-register": "true",
        "value.converter": "io.apicurio.registry.utils.converter.AvroConverter",
        "value.converter.apicurio.registry.converter.serializer": "io.apicurio.registry.serde.avro.AvroKafkaSerializer",
        "value.converter.apicurio.registry.url": "<your-service-registry-core-api-url>",
        "value.converter.apicurio.auth.service.url": "https://identity.api.openshift.com/auth",
        "value.converter.apicurio.auth.realm": "rhoas",
        "value.converter.apicurio.auth.client.id": "<registry-sa-client-id>",
        "value.converter.apicurio.auth.client.secret": "<registry-sa-client-secret>",
        "value.converter.apicurio.registry.as-confluent": "true",
        "value.converter.apicurio.registry.auto-register": "true"
```

> The compatibility mode allows you to use other providers tooling to deserialize and reuse the schemas in the Apicurio service registry.

This also includes the information required for the serializer to authenticate with the service registry using a service account.

### Create the topics in Red Hat OpenShift Streams for Apache Kafka

You will need to manually create the required topics that will be used by Debezium.

1. Create the following topics in your Kafka cluster:

    | Topic Name | Partitions    | Retention Time | Retention Size         |      |
    | ------------------------------------------------------------ | ---- | --------------------- | ----------------- | ---- |
    | avro | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.addresses | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.customers | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.geom | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.orders | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.products | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | avro.inventory.products_on_hand | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | debezium-cluster-configs | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | debezium-cluster-offsets | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | debezium-cluster-status | 1    | 604800000 ms (7 days) | Unlimited         |      |
    | schema-changes.inventory | 1    | 604800000 ms (7 days) | Unlimited |      |

You should end with a table of topics like this:

![topics-openshift-streams-debezium.png](topics-openshift-streams-debezium.png)

**NOTE:** It is also necessary that the topics that start with `debezium-cluster-` are configured with the `compact` policy. If you don't set this property correctly, you will see errors in the kafka connect log and the connector won't be able to start.

![compact policy](cleanup-policy-debezium.png)

### Configure database history

In a separate[ database history Kafka topic](https://debezium.io/documentation/reference/stable/connectors/mysql.html#mysql-schema-history-topic), the Debezium connector for MySQL records all DDL statements along with the position in the binlog where each DDL statement appeared. Because of that, the connector needs access to the target Kafka cluster, so we need to add the connection details to the connector configuration.

You will need the following lines in your connector configuration to access OpenShift Streams. Add them as you did with the details of the converter.

```json
        "database.history.kafka.topic": "schema-changes.inventory",        
        "database.history.kafka.bootstrap.servers": "<your-boostrap-server>",
        "database.history.producer.security.protocol": "SASL_SSL",
        "database.history.producer.sasl.mechanism": "PLAIN",
        "database.history.producer.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=<kafka-sa-client-id> password=<kafka-sa-client-secret>;",
        "database.history.consumer.security.protocol": "SASL_SSL",
        "database.history.consumer.sasl.mechanism": "PLAIN",
        "database.history.consumer.sasl.jaas.config": "org.apache.kafka.common.security.plain.PlainLoginModule required username=<kafka-sa-client-id> password=<kafka-sa-client-secret>;",
```

As you can see, you will need to configure the producer and consumer authentication independently.

> You can check the final file configuration called `dbz-mysql-openshift-registry-avro.json` under the main folder.

### Create the connector

Now that the configuration for the connector is ready, let's add it to the Kafka Connect cluster to start the task that begins capturing the changes of the database. We will use the kcctl a command-line client for Kafka Connect that allows you to register and examine connectors, delete them, restart them among other features.

1. Configure kcctl context:

    ```sh
    kcctl config set-context --cluster http://localhost:8083 local
    ```
    
1. Register the connector using kcctl

    ```bash
    kcctl apply -f dbz-mysql-openshift-registry-avro.json
    ```

### Check the service registry

Access the *Red Hat OpenShift Service Regtistry* console, and you should be able to find all the schema artifacts.

![registry-debezium-artifacts.png](registry-debezium-artifacts.png)

### Check the data

We will use kcat CLI utility to query the information from the OpenShift Streams Kafka cluster. 

1. Set the environment variables in your terminal session:

   ```sh
   export BOOTSTRAP_SERVER=<replace-with-bootstrap-server>
   export CLIENT_ID=<replace-with-kafka-sa-client-id>
   export CLIENT_SECRET=<replace-with-kafka-sa-client-secret>
   ```

2. Check connectivity by querying the cluster metadata:

   ```sh
   kcat -b $BOOTSTRAP_SERVER \
   -X sasl.mechanisms=PLAIN \
   -X security.protocol=SASL_SSL \
   -X sasl.username="$CLIENT_ID" \
   -X sasl.password="$CLIENT_SECRET" -L
   ```

   You should get an output similar to the following:

   ```sh
   Metadata for all topics (from broker -1: sasl_ssl://kafkaesque-c-isn-bhfjlsl-g-dana.bf2.kafka.rhcloud.com:443/bootstrap):
    3 brokers:
     broker 0 at broker-0-kafkaesque-c-isn-bhfjlsl-g-dana.bf2.kafka.rhcloud.com:443 (controller)
     broker 2 at broker-2-kafkaesque-c-isn-bhfjlsl-g-dana.bf2.kafka.rhcloud.com:443
     broker 1 at broker-1-kafkaesque-c-isn-bhfjlsl-g-dana.bf2.kafka.rhcloud.com:443
    11 topics:
     topic "avro.inventory.orders" with 1 partitions:
       partition 0, leader 0, replicas: 0,1,2, isrs: 0,1,2
     topic "avro.inventory.addresses" with 1 partitions:
       partition 0, leader 2, replicas: 2,0,1, isrs: 2,0,1
     topic "debezium-cluster-configs" with 1 partitions:
       partition 0, leader 2, replicas: 2,0,1, isrs: 2,0,1
     topic "debezium-cluster-offsets" with 1 partitions:
       partition 0, leader 0, replicas: 0,1,2, isrs: 0,1,2
     topic "avro.inventory.products" with 1 partitions:
       partition 0, leader 0, replicas: 0,2,1, isrs: 0,2,1
     topic "debezium-cluster-status" with 1 partitions:
       partition 0, leader 0, replicas: 0,2,1, isrs: 0,2,1
     topic "avro" with 1 partitions:
       partition 0, leader 1, replicas: 1,2,0, isrs: 1,2,0
     topic "avro.inventory.geom" with 1 partitions:
       partition 0, leader 0, replicas: 0,2,1, isrs: 0,2,1
     topic "schema-changes.inventory" with 1 partitions:
       partition 0, leader 2, replicas: 2,0,1, isrs: 2,0,1
     topic "avro.inventory.products_on_hand" with 1 partitions:
       partition 0, leader 0, replicas: 0,2,1, isrs: 0,2,1
     topic "avro.inventory.customers" with 1 partitions:
       partition 0, leader 1, replicas: 1,2,0, isrs: 1,2,0
   ```

   

3. Now check the records on the `customers` topic:

   ```sh
   kcat -b $BOOTSTRAP_SERVER \
   -X sasl.mechanisms=PLAIN \
   -X security.protocol=SASL_SSL \
   -X sasl.username="$CLIENT_ID" \
   -X sasl.password="$CLIENT_SECRET" \
   -t avro.inventory.customers -C -e
   ```

   You should see the 4 scramble records in the terminal:

   ```sh
   �
   Sally
        Thomas*sally.thomas@acme.com01.5.4.Final-redhat-00001
   mysqavro����trueinventorycustomers mysql-bin.000003�r����_
   �
    George
          Bailey$gbailey@foobar.com01.5.4.Final-redhat-00001
   mysqavro����trueinventorycustomers mysql-bin.000003�r����_
   �
    Edward
          Walkered@walker.com01.5.4.Final-redhat-00001
   mysqavro����trueinventorycustomers mysql-bin.000003�r����_
   AnneKretchmar$annek@noanswer.org01.5.4.Final-redhat-00001
   mysqavro����lastinventorycustomers mysql-bin.000003�r����_
   % Reached end of topic avro.inventory.customers [0] at offset 4
   ```

   > This is because we are using Avro for the serialization. The kcat utility is expecting Strings and hence, can not convert correctly. We will fix this in the following step.

4. Now that we can see the records, we can then connect with the service registry so kcat can query the used schema and deserialize correctly the missing information:

   ```sh
   kcat -b $BOOTSTRAP_SERVER \
   -X sasl.mechanisms=PLAIN \
   -X security.protocol=SASL_SSL \
   -X sasl.username="$CLIENT_ID" \
   -X sasl.password="$CLIENT_SECRET" \
   -t avro.inventory.customers -C -e \
   -s avro -r https://<registry-sa-client-id>:<registry-sa-client-secret>@<registry-compatibility-api-url> | jq
   ```

   > OpenShift Service Registry also supports basic authentication, that's why we are using the credentials in the format: https://username:password@URL

You will be able to see the record in a nice formatted json structure:

```json
...
{
  "before": null,
  "after": {
    "Value": {
      "id": 1004,
      "first_name": "Anne",
      "last_name": "Kretchmar",
      "email": "annek@noanswer.org"
    }
  },
  "source": {
    "version": "1.5.4.Final-redhat-00001",
    "connector": "mysql",
    "name": "avro",
    "ts_ms": 1642452727355,
    "snapshot": {
      "string": "last"
    },
    "db": "inventory",
    "sequence": null,
    "table": {
      "string": "customers"
    },
    "server_id": 0,
    "gtid": null,
    "file": "mysql-bin.000003",
    "pos": 154,
    "row": 0,
    "thread": null,
    "query": null
  },
  "op": "r",
  "ts_ms": {
    "long": 1642452727355
  },
  "transaction": null
}
```

Congratulations! You were able to send Avro serialized records from MySQL to OpenShift Streams using OpenShift Service Registry.

## Summary

Although Debezium makes it easy to capture database changes and record them in Kafka, one of the mo

re critical decisions you have to make is *how* you will serialize those change events in Kafka. Debezium allows you to select key and value *converters* to choose from different types of options. The *Red Hat OpenShift Service Registry* will enable you to store externalized schema versions to minimize the payload to propagate.

