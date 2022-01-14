# Red Hat Integration - Debezium Container Image

## Change Data Capture Container Image

To build the image download from customer portal or developer site the Debezium connectors and the apicurio converters

Run the following command to build the image:

```sh
docker build -t quay.io/redhatintegration/rhi-cdc-connect:2021-Q3 -f ./.docker/Dockerfile .
```