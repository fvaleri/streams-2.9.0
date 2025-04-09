# Cluster auto-rebalancing

Run the init script to setup your environment.

```sh
$ source init.sh 
namespace/test created
deployment.apps/strimzi-cluster-operator condition met
Done
```

Create a test Kafka cluster with Cruise Control and topic.

```sh
$ kubectl create -f sessions/001/install.yaml
kafkanodepool.kafka.strimzi.io/controller created
kafkanodepool.kafka.strimzi.io/broker created
kafka.kafka.strimzi.io/my-cluster created
kafkatopic.kafka.strimzi.io/my-topic created
```

Wait for the cluster to be up and running.

```sh
$ kubectl get po
NAME                                         READY   STATUS    RESTARTS   AGE
my-cluster-broker-3                          1/1     Running   0          86s
my-cluster-broker-4                          1/1     Running   0          86s
my-cluster-broker-5                          1/1     Running   0          86s
my-cluster-controller-0                      1/1     Running   0          85s
my-cluster-controller-1                      1/1     Running   0          85s
my-cluster-controller-2                      1/1     Running   0          85s
my-cluster-cruise-control-ffd8965bd-5g8vz    1/1     Running   0          33s
my-cluster-entity-operator-b9877bf66-xq5wd   2/2     Running   0          55s
strimzi-cluster-operator-6596f469c9-2qb45    1/1     Running   0          2m11s
```

From the Kafka resource we can see that the `autoReabalance` feature is enabled.

> [!IMPORTANT]
> If a template is not specified, default Cruise Control rebalancing configuration is used.

```sh
$ kubectl get k my-cluster -o yaml | yq .spec.cruiseControl
autoRebalance:
  - mode: add-brokers
  - mode: remove-brokers
```

Check how partitions are distributed between brokers.

```sh
$ kubectl-kafka bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --topic my-topic
Topic: my-topic	TopicId: tPlZGjijRB-peP7vSOv3vg	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2,segment.bytes=1073741824,retention.ms=7200000
	Topic: my-topic	Partition: 0	Leader: 5	Replicas: 5,3,4	Isr: 5,3,4	Elr: 	LastKnownElr: 
	Topic: my-topic	Partition: 1	Leader: 3	Replicas: 3,4,5	Isr: 3,4,5	Elr: 	LastKnownElr: 
	Topic: my-topic	Partition: 2	Leader: 4	Replicas: 4,5,3	Isr: 4,5,3	Elr: 	LastKnownElr: 
```

Scale up the Kafka cluster by adding two new brokers.
When the new brokers are ready, Cruise Control is restarted to take into account the new cluster capacity.

```sh
$ kubectl patch knp broker --type merge -p '
    spec:
      replicas: 5'
kafkanodepool.kafka.strimzi.io/broker patched
```

Watch the auto-generated KafkaRebalance execution from command line.

```sh
$ kubectl get kr -w
NAME                                      CLUSTER      TEMPLATE   STATUS
my-cluster-auto-rebalancing-add-brokers   my-cluster              PendingProposal
my-cluster-auto-rebalancing-add-brokers   my-cluster              ProposalReady
my-cluster-auto-rebalancing-add-brokers   my-cluster              Rebalancing
my-cluster-auto-rebalancing-add-brokers   my-cluster              Ready
```

Check again how partitions are distributed between brokers.

```sh
$ kubectl-kafka bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe --topic my-topic
Topic: my-topic	TopicId: tPlZGjijRB-peP7vSOv3vg	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=2,segment.bytes=1073741824,retention.ms=7200000
	Topic: my-topic	Partition: 0	Leader: 5	Replicas: 5,6,7	Isr: 5,6,7	Elr: 	LastKnownElr: 
	Topic: my-topic	Partition: 1	Leader: 7	Replicas: 7,6,5	Isr: 5,6,7	Elr: 	LastKnownElr: 
	Topic: my-topic	Partition: 2	Leader: 7	Replicas: 7,5,6	Isr: 5,6,7	Elr: 	LastKnownElr: 
```
