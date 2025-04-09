# JBOD volumes removal

Run the init script to setup your environment.

```sh
$ source init.sh 
namespace/test created
deployment.apps/strimzi-cluster-operator condition met
Done
```

Create a test Kafka cluster and topic.

```sh
$ kubectl create -f sessions/002/install.yaml
kafkanodepool.kafka.strimzi.io/controller created
kafkanodepool.kafka.strimzi.io/broker created
kafka.kafka.strimzi.io/my-cluster created
kafkatopic.kafka.strimzi.io/my-topic created
```

Wait for the cluster to be up and running.

```sh
$ kubectl get po
NAME                                          READY   STATUS    RESTARTS   AGE
my-cluster-broker-3                           1/1     Running   0          3m54s
my-cluster-broker-4                           1/1     Running   0          3m53s
my-cluster-broker-5                           1/1     Running   0          3m53s
my-cluster-controller-0                       1/1     Running   0          3m52s
my-cluster-controller-1                       1/1     Running   0          3m52s
my-cluster-controller-2                       1/1     Running   0          3m52s
my-cluster-cruise-control-7d9dbd9875-xfhc9    1/1     Running   0          27s
my-cluster-entity-operator-794b6fbcb6-gmlzf   2/2     Running   0          49s
strimzi-cluster-operator-6596f469c9-wwhfg     1/1     Running   0          15m
```

Check how partitions are assigned to broker volumes.

```sh
$ kubectl-kafka bin/kafka-log-dirs.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe \
  --broker-list 3,4,5 --topic-list my-topic | tail -n1 | yq .brokers[].logDirs
# broker 4, disks: 2,0,1
[{"partitions": 
    [{"partition": "my-topic-0", "size": 0, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-2/kafka-log4"},
{"partitions": 
    [{"partition": "my-topic-2", "size": 0, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-0/kafka-log4"},
{"partitions": 
    [{"partition": "my-topic-1", "size": 0, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-1/kafka-log4"}]

# broker 5, disks: 1,0,2
[{"partitions": 
    [{"partition": "my-topic-2", "size": 0, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-1/kafka-log5"},
{"partitions": 
    [{"partition": "my-topic-1", "size": 0, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-0/kafka-log5"},
{"partitions": 
    [{"partition": "my-topic-0", "size": 0, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-2/kafka-log5"}]

# broker 3, disks: 0,2,1
[{"partitions": 
    [{"partition": "my-topic-2", "size": 0, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-0/kafka-log3"},
{"partitions": 
    [{"partition": "my-topic-1", "size": 0, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-2/kafka-log3"},
{"partitions": 
    [{"partition": "my-topic-0", "size": 0, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-1/kafka-log3"}]
```

Move all partitions from volumes 1 and 2 to volume 0 on all brokers by creating a KafkaRebalance with auto-approval.

```sh
$ echo -e 'apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaRebalance
metadata:
  name: my-rebalance
  labels:
    strimzi.io/cluster: my-cluster
  annotations:
    strimzi.io/rebalance-auto-approval: "true"    
spec:
  mode: remove-disks
  moveReplicasOffVolumes:
    - brokerId: 3
      volumeIds:
        - 1
        - 2
    - brokerId: 4
      volumeIds:
        - 1
        - 2
    - brokerId: 5
      volumeIds:
        - 1
        - 2' | kubectl create -f -
kafkarebalance.kafka.strimzi.io/my-rebalance created

$ kubectl get kr -w
NAME           CLUSTER      TEMPLATE   STATUS
my-rebalance   my-cluster              Rebalancing
my-rebalance   my-cluster              Ready
```

When the rebalance is ready, check again how partitions are assigned to broker volumes.

```sh
$ kubectl-kafka bin/kafka-log-dirs.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --describe \
  --broker-list 3,4,5 --topic-list my-topic | tail -n1 | yq .brokers[].logDirs
# broker 4, disks: 2,0,1
[{"partitions": [], 
    "error": null, "logDir": "/var/lib/kafka/data-2/kafka-log4"},
{"partitions": 
    [{"partition": "my-topic-0", "size": 3561155, "offsetLag": 0, "isFuture": false},
    {"partition": "my-topic-1", "size": 3663704, "offsetLag": 0, "isFuture": false}, 
    {"partition": "my-topic-2", "size": 3773327, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-0/kafka-log4"},
{"partitions": [], "error": null, "logDir": "/var/lib/kafka/data-1/kafka-log4"}]

# broker 5, disks: 1,0,2
[{"partitions": [], "error": null, "logDir": "/var/lib/kafka/data-1/kafka-log5"},
{"partitions": 
    [{"partition": "my-topic-0", "size": 3561155, "offsetLag": 0, "isFuture": false},
    {"partition": "my-topic-1", "size": 3663704, "offsetLag": 0, "isFuture": false},
    {"partition": "my-topic-2", "size": 3773327, "offsetLag": 0, "isFuture": false}], 
    "error": null, "logDir": "/var/lib/kafka/data-0/kafka-log5"},
{"partitions": [], "error": null, "logDir": "/var/lib/kafka/data-2/kafka-log5"}]

# broker 3, disks: 0,2,1
[{"partitions": 
    [{"partition": "my-topic-0", "size": 3561155, "offsetLag": 0, "isFuture": false},
    {"partition": "my-topic-1", "size": 3663704, "offsetLag": 0, "isFuture": false},
    {"partition": "my-topic-2", "size": 3773327, "offsetLag": 0, "isFuture": false}],
    "error": null, "logDir": "/var/lib/kafka/data-0/kafka-log3"},
{"partitions": [], "error": null, "logDir": "/var/lib/kafka/data-2/kafka-log3"},
{"partitions": [], "error": null, "logDir": "/var/lib/kafka/data-1/kafka-log3"}]
```

Remove volumes 1 and 2 from the broker noed pool and all related PVCs.

> [!WARNING]
> There is currently no check on removed disks, so make sure that rebalance completed and there are no partitions left.

```sh
$ kubectl patch knp broker --type=json \
  -p='[{"op":"remove","path":"/spec/storage/volumes/1"}],[{"op":"remove","path":"/spec/storage/volumes/2"}]'
kafkanodepool.kafka.strimzi.io/broker patched

$ kubectl delete pvc \
  $(kubectl get pvc -l strimzi.io/name=my-cluster-kafka,strimzi.io/pool-name=broker | grep -e "data-1\|data-2" | awk '{print $1}')
persistentvolumeclaim "data-1-my-cluster-broker-3" deleted
persistentvolumeclaim "data-1-my-cluster-broker-4" deleted
persistentvolumeclaim "data-1-my-cluster-broker-5" deleted
persistentvolumeclaim "data-2-my-cluster-broker-3" deleted
persistentvolumeclaim "data-2-my-cluster-broker-4" deleted
persistentvolumeclaim "data-2-my-cluster-broker-5" deleted
```
