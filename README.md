# kafka-kerberos-k8s-example

### Requirements

minikube (https://minikube.sigs.k8s.io/docs/start/)
kubectl (https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
helm (https://helm.sh/docs/intro/install/)
docker (https://docs.docker.com/engine/install/)

### Setup environment

```
# start minikube with k8s version 1.27
minikube start --kubernetes-version=v1.27.0

# check minikube status
minikube status

# check k8s connection status
kubectl get nodes

# Clone example repo
git clone https://github.com/abalalaev/kafka-kerberos-k8s-example.git
cd ./kafka-kerberos-k8s-example
```

### Deploy kerberos 

```
# Clone krb5-helm repo containing helm chart and dockerfile for kuberos(kerberos for kubernetes) deployment
git clone https://github.com/theSaarco/krb5-helm.git

# Point your terminal's docker-cli to the Docker Engine inside minikube
eval $(minikube -p minikube docker-env)

# Build kuberos docker image
cd ./krb5-helm/docker/server/
docker build -f Dockerfile -t kuberos:latest ./

# Deploy kuberos helm chart
cd ../../kuberos/
helm -n default install kuberos ./

# Check kuberos-kdc pod status
kubectl get pods -n default
```

### Create kuberos principals

```
# Connect to kerberos kadmin container
kubectl -n default exec -ti kuberos-kuberos-kdc-0 --container kadmin -- /bin/bash

# Run kadmin's an interactive shell
kadmin.local

# Create principals for every kafka broker
addprinc -randkey kafka/kafka-controller-0.kafka-controller-headless.default.svc.cluster.local
addprinc -randkey kafka/kafka-controller-1.kafka-controller-headless.default.svc.cluster.local
addprinc -randkey kafka/kafka-controller-2.kafka-controller-headless.default.svc.cluster.local

# Create principals for kafka admin and test user
addprinc -randkey kafkaAdmin
addprinc -randkey kafkaUser

# Save principal's keys to the /etc/krb5.keytab file
ktadd kafka/kafka-controller-0.kafka-controller-headless.default.svc.cluster.local
ktadd kafka/kafka-controller-1.kafka-controller-headless.default.svc.cluster.local
ktadd kafka/kafka-controller-2.kafka-controller-headless.default.svc.cluster.local
ktadd kafkaAdmin kafkaUser
```

### Create k8s secret with keytab file and configmap with kerberos client config

```
cd ./kafka-kerberos-k8s-example

# Copy created keytab file and krb config to the local fs
kubectl cp default/kuberos-kuberos-kdc-0:/etc/krb5.keytab ./krb5.keytab -c kadmin
kubectl cp default/kuberos-kuberos-kdc-0:/etc/krb5.conf ./krb5.conf -c kadmin

# Create secret for keytab
kubectl create secret generic kafka-krb5-keytab --from-file=./krb5.keytab -n default

# Create configmap for krb5 client config
kubectl create configmap kafka-krb5-config --from-file=./krb5.conf -n default
```

### Deploy kafka

```
# Change dir to example repo root
cd ./kafka-kerberos-k8s-example

# Deploy bitnami kafka helm chart using kafka-values.yml
helm install kafka oci://registry-1.docker.io/bitnamicharts/kafka --values kafka-values.yml --namespace default

# Check kafka's pods status
kubectl get pods -n default
```

### Try to use kafka client with authentication and authorization

```
# Create kafka client pod
kubectl run kafka-client --restart='Never' --image docker.io/bitnami/kafka:3.6.1-debian-12-r12 --namespace default --command -- sleep infinity

# Copy keytab file and kerberos client config to the kafka-client pod's fs
kubectl cp krb5.keytab kafka-client:/bitnami/kafka/config/krb5.keytab --namespace default
kubectl cp krb5.conf kafka-client:/bitnami/kafka/config/krb5.conf --namespace default

# Copy kafka Admin client.properties file to the kafka-client pod's fs
kubectl cp kafkaAdmin.client.properties kafka-client:/tmp/client.properties --namespace default

# Login to the kafka-client pod
kubectl exec --tty -i kafka-client --namespace default -- bash

# Create two test topics (test1 and test2)
export KAFKA_OPTS="-Djava.security.krb5.conf=/bitnami/kafka/config/krb5.conf"; kafka-topics.sh --command-config /tmp/client.properties --bootstrap-server kafka-controller-0.kafka-controller-headless.default.svc.cluster.local:9092 --topic test1 --create

export KAFKA_OPTS="-Djava.security.krb5.conf=/bitnami/kafka/config/krb5.conf"; kafka-topics.sh --command-config /tmp/client.properties --bootstrap-server kafka-controller-0.kafka-controller-headless.default.svc.cluster.local:9092 --topic test1 --create

# Produce some test messages to the both test topics
export KAFKA_OPTS="-Djava.security.krb5.conf=/bitnami/kafka/config/krb5.conf"; kafka-console-producer.sh --producer.config /tmp/client.properties --bootstrap-server kafka-controller-0.kafka-controller-headless.default.svc.cluster.local:9092 --topic test1

export KAFKA_OPTS="-Djava.security.krb5.conf=/bitnami/kafka/config/krb5.conf"; kafka-console-producer.sh --producer.config /tmp/client.properties --bootstrap-server kafka-controller-0.kafka-controller-headless.default.svc.cluster.local:9092 --topic test2

# Provide read permisson for test1 topic to kafkaUser principal 
export KAFKA_OPTS="-Djava.security.krb5.conf=/bitnami/kafka/config/krb5.conf"; kafka-acls.sh --command-config /tmp/client.properties --bootstrap-server kafka-controller-0.kafka-controller-headless.default.svc.cluster.local:9092 --add --allow-principal "User:kafkaUser" --operation Read --group '*' --topic test1

# Logout and copy kafka User client.properties file to the kafka-client pod's fs
kubectl cp kafkaUser.client.properties kafka-client:/tmp/client.properties --namespace default

# Login to the kafka-client pod
kubectl exec --tty -i kafka-client --namespace default -- bash

# Try to consume test1 topic as kafkaUser principal (this should work)
export KAFKA_OPTS="-Djava.security.krb5.conf=/bitnami/kafka/config/krb5.conf"; kafka-console-consumer.sh --consumer.config /tmp/client.properties --bootstrap-server kafka-controller-0.kafka-controller-headless.default.svc.cluster.local:9092 --topic test1 --from-beginning

# Try to consume test2 topic as kafkaUser principal (this should not work because of 'Topic authorization failed')
export KAFKA_OPTS="-Djava.security.krb5.conf=/bitnami/kafka/config/krb5.conf"; kafka-console-consumer.sh --consumer.config /tmp/client.properties --bootstrap-server kafka-controller-0.kafka-controller-headless.default.svc.cluster.local:9092 --topic test2 --from-beginning
```

