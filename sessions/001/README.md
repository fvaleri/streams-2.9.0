# Cluster auto-rebalancing

Run the init script to setup your environment.

```sh
$ source init.sh 
namespace/test created
Downloading Strimzi to /tmp/strimzi-0.45.0.yaml
Done
```

Create a test Kafka cluster and topic.

```sh
$ kubectl create -f sessions/001/install.yaml
kafkanodepool.kafka.strimzi.io/controller created
kafkanodepool.kafka.strimzi.io/broker created
kafka.kafka.strimzi.io/my-cluster created
kafkatopic.kafka.strimzi.io/my-topic created

$ kubectl get po
NAME                                         READY   STATUS    RESTARTS   AGE
my-cluster-broker-3                          1/1     Running   0          2m21s
my-cluster-broker-4                          1/1     Running   0          2m21s
my-cluster-broker-5                          1/1     Running   0          2m21s
my-cluster-controller-0                      1/1     Running   0          2m21s
my-cluster-controller-1                      1/1     Running   0          2m21s
my-cluster-controller-2                      1/1     Running   0          2m21s
my-cluster-entity-operator-69859f6d6-bqv2r   2/2     Running   0          108s
strimzi-cluster-operator-6596f469c9-vw5d5    1/1     Running   0          2m57s
```

Send some data and check how partitions are distributed among the brokers.

```sh
$ kubectl-kafka bin/kafka-producer-perf-test.sh --topic my-topic --record-size 100 --num-records 100000 \
  --throughput -1 --producer-props acks=all bootstrap.servers=my-cluster-kafka-bootstrap:9092
100000 records sent, 39952.057531 records/sec (3.81 MB/sec), 1199.20 ms avg latency, 1948.00 ms max latency, 1227 ms 50th, 1892 ms 95th, 1940 ms 99th, 1948 ms 99.9th.

$ kubectl-kafka bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --topic my-topic
Topic: my-topic	TopicId: 2q-H2y2PRoyGktIr6q4Lrg	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2,segment.bytes=1073741824,retention.ms=7200000
	Topic: my-topic	Partition: 0	Leader: 4	Replicas: 4,5,3	Isr: 4,5,3	Elr: 	LastKnownElr: 
	Topic: my-topic	Partition: 1	Leader: 5	Replicas: 5,3,4	Isr: 5,3,4	Elr: 	LastKnownElr: 
	Topic: my-topic	Partition: 2	Leader: 3	Replicas: 3,4,5	Isr: 3,4,5	Elr: 	LastKnownElr: 
```

Deploy Cruise Control with auto-rebalancing.
All broker pods will roll to install the Cruise Control's metrics reporter plugin.

> [!IMPORTANT]
> If a template is not specified, default Cruise Control rebalancing configuration is used.

```sh
$ kubectl patch k my-cluster --type merge -p '
    spec:
      cruiseControl: 
        autoRebalance:
          - mode: add-brokers
          - mode: remove-brokers'
kafka.kafka.strimzi.io/my-cluster patched
```

Wait some time for Cruise Control to build the workload model, and then scale up the Kafka cluster.

```sh
$ kubectl patch knp broker --type merge -p '
    spec:
      replicas: 5'
kafkanodepool.kafka.strimzi.io/broker patched
```

Follow the auto-generated KafkaRebalance execution from command line.

```sh
$ kubectl get kr -w
NAME                                      CLUSTER      TEMPLATE   STATUS
my-cluster-auto-rebalancing-add-brokers   my-cluster              PendingProposal
my-cluster-auto-rebalancing-add-brokers   my-cluster              ProposalReady
my-cluster-auto-rebalancing-add-brokers   my-cluster              Rebalancing
my-cluster-auto-rebalancing-add-brokers   my-cluster              Ready
```

When the KafkaRebalance is ready, check again how partitions are distributed among the brokers.

```sh
$ kubectl-kafka bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --topic my-topic
Topic: my-topic	TopicId: 2q-H2y2PRoyGktIr6q4Lrg	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2,segment.bytes=1073741824,retention.ms=7200000
	Topic: my-topic	Partition: 0	Leader: 4	Replicas: 4,6,7	Isr: 6,7,4	Elr: 	LastKnownElr: 
	Topic: my-topic	Partition: 1	Leader: 5	Replicas: 5,3,4	Isr: 3,4,5	Elr: 	LastKnownElr: 
	Topic: my-topic	Partition: 2	Leader: 3	Replicas: 3,4,5	Isr: 3,4,5	Elr: 	LastKnownElr:
```
