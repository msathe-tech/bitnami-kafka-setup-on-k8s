# Kafka on Kubernetes (on Public Cloud or NSX-T)
We will setup Kafka on Kubernetes using Bitnami Helm charts.
This Kafka will be accessible from outside of the Kubernetes cluster.
We will use Helm charts but will need to modify the charts to make it work for this use case.

## Oreview
We need to setup two separate components - Zookeeper and Kafka.
Kafka uses Zookeeper.

## Prerequisites
1. Kubernetes cluster
2. kubectl set with required Kubernetes cluster
3. For the sake of simplicity keep the cluster to a single node. This is because we will setup Kafka with a single POD. Or try to setup a cluster with nodes < pods in case you intend to scale up the Kafka or Zookeeper. This is necessary to ensure no node remains unused. Will explain later why this is important.
4. Helm setup in the Kubernetes cluster. Take a look at this video to setup Helm on Kubernetes - https://www.youtube.com/watch?v=irvBlE1la_w
5. Take a look at reference YAMLs included in this git.

## Steps
The steps here are a quick guide to setup Kafka on Kuberetes that can be accessed by apps outside of the Kubernetes cluster.

1. `git clone https://github.com/bitnami/charts`
2. Edit kafka-values.yaml and change the value of "clusterDomain"
3. Copy kafka-values.yaml to {your working dir}/charts/bitnami/kafka/values.yaml
4. Copy kafka-requirements.yaml to {your working dir}/charts/bitnami/kafka/requirements.yaml
5. Copy zookeeper-values.yaml to {your working dir}/charts/bitnami/zookeeper/values.yaml
6. `cd {your working dir}/charts/bitnami`
7. ```helm install --name bitnami-kafka-zookeeper ./zookeeper```
8. ```kubectl get services``` Get the external IP of the Zookeeper service and add A-record to your DNS. You will need this DNS name later.
9. ```helm install --name bitnami-kafka ./kafka```
10. ```export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=kafka,app.kubernetes.io/instance=bitnami-kafka,app.kubernetes.io/component=kafka" -o jsonpath="{.items[0].metadata.name}")``` Get the pod name of the kafka pod.
11. ```kubectl describe pod/$POD_NAME```
12. Pick up value of ```KAFKA_CFG_ADVERTISED_LISTENERS```. Replace ```$(MY_POD_NAME)``` with value of $POD_NAME you retrieved ealier. This becomes Broker Host.
13. Pick up value of ```KAFKA_PORT_NUMBER```.
14. Construct a Broker URL using 12 and 13. It should looks something similar to bitnami-kafka-0.bitnami-kafka-headless.default.svc.<your base domain>:9092
15. Construct a Broker URL for every Kafka pod.
16. Go to you IaaS cloud and setup DNS A-record for every Broker Host and map it to external IP of the Kafka service.

At the end of these steps you should have a Kafka setup that can be accessed from web.

### If you have number of nodes > number of pods
In this case you will have Kubernetes cluster nodes that won't have a Kafka or Zookeeper pod on it.
In such a case you have to remove each such node from the backend VMs of the Load Balancer.
This is because the LB will simply route the requests to all VMs and your app will fail whenever request lands on a VM that doesn't have a required pod.
First run ```kubectl get nodes``` to get names of all nodes in the cluster.
Run ```kubectl describe <pod for each Kafka and Zookeeper node>``` to retrive node of the POD. Any node of the cluster is not in the list should be removed as a LB backend.
Remember, there are two separate LBs, one for Kafka and other for Zookeeper. So you have to check nodes for each one of them separately. It is possible that a node may have a Zookeeper pod but not a Kafka pod, and vice versa.

# Run the app
java -jar target/wordcount-0.0.1-SNAPSHOT.jar \
--spring.cloud.stream.kafka.binder.brokers=bitnami-kafka-0.bitnami-kafka-headless.default.svc.kafka.cloudpandit.online:9092

## Cleanup
	helm delete --purge bitnami-kafka
	helm delete --purge bitnami-kafka-zookeeper


### Kafka helm install output
To create a topic run the following command:

    export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=kafka,app.kubernetes.io/instance=bitnami-kafka,app.kubernetes.io/component=kafka" -o jsonpath="{.items[0].metadata.name}")
    kubectl --namespace default exec -it $POD_NAME -- kafka-topics.sh --create --zookeeper bitnami-kafka-zookeeper:2181 --replication-factor 1 --partitions 1 --topic test

To list all the topics run the following command:

    export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=kafka,app.kubernetes.io/instance=bitnami-kafka,app.kubernetes.io/component=kafka" -o jsonpath="{.items[0].metadata.name}")
    kubectl --namespace default exec -it $POD_NAME -- kafka-topics.sh --list --zookeeper bitnami-kafka-zookeeper:2181

To start a kafka producer run the following command:

    export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=kafka,app.kubernetes.io/instance=bitnami-kafka,app.kubernetes.io/component=kafka" -o jsonpath="{.items[0].metadata.name}")
    kubectl --namespace default exec -it $POD_NAME -- kafka-console-producer.sh --broker-list localhost:9092 --topic test

To start a kafka consumer run the following command:

    export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=kafka,app.kubernetes.io/instance=bitnami-kafka,app.kubernetes.io/component=kafka" -o jsonpath="{.items[0].metadata.name}")
    kubectl --namespace default exec -it $POD_NAME -- kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning

To connect to your Kafka server from outside the cluster execute the following commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w bitnami-kafka'

    export SERVICE_IP=$(kubectl get svc --namespace default bitnami-kafka --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
    echo "Kafka Broker Endpoint: $SERVICE_IP:9092"

    PRODUCER:
        kafka-console-producer.sh --broker-list 127.0.0.1:9092 --topic test
    CONSUMER:
        kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic test --from-beginning

Madhavs-MacBook-Pro:bitnami msathe$ helm install --name bitnami-kafka-zookeeper ./zookeeper
NAME:   bitnami-kafka-zookeeper
LAST DEPLOYED: Fri Aug 23 14:34:15 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1beta2/StatefulSet
NAME                     DESIRED  CURRENT  AGE
bitnami-kafka-zookeeper  1        1        0s

==> v1/Pod(related)
NAME                       READY  STATUS   RESTARTS  AGE
bitnami-kafka-zookeeper-0  0/1    Pending  0         0s

==> v1/Service
NAME                              TYPE          CLUSTER-IP   EXTERNAL-IP  PORT(S)                                       AGE
bitnami-kafka-zookeeper-headless  ClusterIP     None         <none>       2181/TCP,2888/TCP,3888/TCP                    0s
bitnami-kafka-zookeeper           LoadBalancer  10.12.1.193  <pending>    2181:31251/TCP,2888:32586/TCP,3888:31208/TCP  0s


NOTES:

-------------------------------------------------------------------------------
 WARNING

    By specifying "serviceType=LoadBalancer" and not specifying "auth.enabled=true"
    you have most likely exposed the ZooKeeper service externally without any
    authentication mechanism.

    For security reasons, we strongly suggest that you switch to "ClusterIP" or
    "NodePort". As alternative, you can also specify a valid password on the
    "auth.clientPassword" parameter.

-------------------------------------------------------------------------------

** Please be patient while the chart is being deployed **

ZooKeeper can be accessed via port 2181 on the following DNS name from within your cluster:

    bitnami-kafka-zookeeper.default.svc.cluster.local

To connect to your ZooKeeper server run the following commands:

    export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=zookeeper,app.kubernetes.io/instance=bitnami-kafka-zookeeper,app.kubernetes.io/component=zookeeper" -o jsonpath="{.items[0].metadata.name}")
    kubectl exec -it $POD_NAME -- zkCli.sh

To connect to your ZooKeeper server from outside the cluster execute the following commands:

  NOTE: It may take a few minutes for the LoadBalancer IP to be available.
        Watch the status with: 'kubectl get svc --namespace default -w bitnami-kafka-zookeeper'

    export SERVICE_IP=$(kubectl get svc --namespace default bitnami-kafka-zookeeper --template "{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}")
    zkCli.sh $SERVICE_IP:2181
