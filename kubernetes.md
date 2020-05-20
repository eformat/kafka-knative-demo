# Kubernetes Set-up

## Database

Run the database:

```shell
# (To run container as root on OpenShift)
oc adm policy add-scc-to-user anyuid system:serviceaccount:kafka:default

kubectl run --image=quay.io/gunnarmorling/javaland2020-knativedemo-postgres \
    weatherdb \
    --port=5432 \
    --env="POSTGRES_USER=postgresuser" \
    --env="POSTGRES_PASSWORD=postgrespw" \
    --env="POSTGRES_DB=weatherdb"
```

Expose it as service:

```shell
cat <<EOF | kubectl -n kafka apply -f -
apiVersion: v1
kind: Service
metadata:
  name: weatherdb
  labels:
    run: weatherdb
spec:
  type: NodePort
  ports:
   - port: 5432
  selector:
   run: weatherdb
EOF
```

## Strimzi Cluster Operator

```shell
curl -L https://github.com/strimzi/strimzi-kafka-operator/releases/download/0.16.1/strimzi-cluster-operator-0.16.1.yaml \
  | sed 's/namespace: .*/namespace: kafka/' \
  | kubectl apply -f - -n kafka
```

## Apache Kafka

```shell
kubectl -n kafka \
    apply -f https://raw.githubusercontent.com/strimzi/strimzi-kafka-operator/0.16.1/examples/kafka/kafka-persistent-single.yaml \
  && kubectl wait kafka/my-cluster --for=condition=Ready --timeout=300s -n kafka
```

## Kafka Connect

```shell
cat <<EOF | kubectl -n kafka apply -f -
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
  # use-connector-resources configures this KafkaConnect
  # to use KafkaConnector resources to avoid
  # needing to call the Connect REST API directly
    strimzi.io/use-connector-resources: "true"
spec:
  image: quay.io/gunnarmorling/javaland2020-knativedemo-kafkaconnect-debezium
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9093
  tls:
    trustedCertificates:
      - secretName: my-cluster-cluster-ca-cert
        certificate: ca.crt
  config:
    config.storage.replication.factor: 1
    offset.storage.replication.factor: 1
    status.storage.replication.factor: 1
EOF
```

## The Sensors App

```shell
kubectl run --image=docker.io/gunnarmorling/debezium-knative-demo-sensors sensors --env="KAFKA_BOOTSTRAP_SERVERS=my-cluster-kafka-bootstrap.kafka:9092"
```

# Demo

Start a tooling pod:

```shell
kubectl run tooling -it --image=debezium/tooling --restart=Never
```

Examining temperature value topic:

```shell
kafkacat -b my-cluster-kafka-bootstrap:9092 -C -o end -q -u -t temperature-values
```

Selecting existing weather stations:

```shell
[root@tooling /] pgcli postgresql://postgresuser:postgrespw@weatherdb:5432/weatherdb

weatherdb> select * from weather.weatherstations;
```

Registering a Debezium connector:

```shell
cat <<EOF | kubectl -n kafka apply -f -
apiVersion: "kafka.strimzi.io/v1alpha1"
kind: "KafkaConnector"
metadata:
  name: "inventory-connector"
  labels:
    strimzi.io/cluster: my-connect-cluster
spec:
  class: io.debezium.connector.postgresql.PostgresConnector
  tasksMax: 1
  config:
    database.hostname: weatherdb.kafka.svc
    database.port: "5432"
    database.user: "postgresuser"
    database.password: "postgrespw"
    database.server.name: "dbserver1"
    database.dbname : "weatherdb"
    schema.whitelist : "weather"
    decimal.handling.mode : "double"
    key.converter : "org.apache.kafka.connect.json.JsonConverter"
    key.converter.schemas.enable : "false"
    value.converter : "org.apache.kafka.connect.json.JsonConverter"
    value.converter.schemas.enable : "false"
EOF
```

Examining weather station topic:

```shell
kafkacat -b my-cluster-kafka-bootstrap:9092 -C -o beginning -q -u -t dbserver1.weather.weatherstations | jq .
```

Run aggregator app:

```shell
kubectl run --image=docker.io/gunnarmorling/debezium-knative-demo-aggregator aggregator --env="QUARKUS_KAFKA_STREAMS_BOOTSTRAP_SERVERS=my-cluster-kafka-bootstrap.kafka:9092"
```

Examining enriched topic:

```shell
kafkacat -b my-cluster-kafka-bootstrap:9092 -C -o end -q -u -t temperature-values-enriched | jq .
```

Enabling more weather stations:

```sql
update weather.weatherstations set active = true;
```

Stop tooling pod:

```shell
kubectl delete pod tooling
```

# Optional: Running maps app without Knative

```shell
kubectl run --image=docker.io/gunnarmorling/debezium-knative-demo-temperature-map temperature-map \
    --env="MAP_READ_FROM_TOPIC=true" \
    --env="KAFKA_BOOTSTRAP_SERVERS=my-cluster-kafka-bootstrap.kafka:9092"
```

```shell
cat <<EOF | kubectl -n kafka apply -f -
apiVersion: v1
kind: Service
metadata:
  name: temperature-map
  labels:
    run: temperature-map
spec:
  type: NodePort
  ports:
   - port: 8080
  selector:
   run: temperature-map
EOF
```

```shell
oc expose svc temperature-map
```