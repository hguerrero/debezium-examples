FROM registry.redhat.io/amq7/amq-streams-kafka-32-rhel8:2.2.0
USER root:root
COPY ./plugins/ /opt/kafka/plugins/
COPY ./apicurio-converter/ /opt/kafka/external_libs/apicurio/
RUN for d in /opt/kafka/plugins/*/; do ln -snf /opt/kafka/external_libs/apicurio/* $d; done
USER 1001