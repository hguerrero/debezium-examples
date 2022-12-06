# Red Hat Build of Debezium Container Image

## Change Data Capture Container Image

To build the image download from customer portal or developer site the Debezium connectors and the Apicurio Service Registry Kafka Connect converters

Run the following command to build the image:

```sh
docker build -t quay.io/redhatintegration/rhi-cdc-connect:<RELEASE> -f ./.docker/Dockerfile .
```