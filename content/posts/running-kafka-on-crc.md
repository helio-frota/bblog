---
title: "Running Kafka on CRC and Ubuntu 20.04"
date: 2021-01-11T10:41:42-03:00
draft: false
---

> Note: CRC version 4.6.6

> This post is based on [this blog post](https://medium.com/@osamaahmedtahir17/configure-and-run-kafka-on-local-openshift-cluster-through-code-ready-container-windows-part-2-7c4078a12dfa), 
but for Ubuntu 20.04 and with the minimal steps/content required to run as possible.

```
crc start -m 20000 -c 6
eval $(crc oc-env)
```

Login as kubeadmin and create a new project:

```
oc new-project nodejs-kafka
```

Install strimzi kafka-operator (this post is using version 0.20.1)

```
oc apply -f 'https://strimzi.io/install/latest?namespace=nodejs-kafka'
```

Create a file kafka.yaml with the following content:

```yml
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
namespace: nodejs-kafka
metadata:
  name: nodejs-kafka-cluster
spec:
  kafka:
    replicas: 1
    listeners:
      plain: {}
      tls: {}
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
    storage:
      type: ephemeral
  zookeeper:
    replicas: 1
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

```
oc apply -f kafka.yaml
```

Verify the running services

```
oc get services
NAME                                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
nodejs-kafka-cluster-kafka-bootstrap    ClusterIP   172.25.112.113   <none>        9091/TCP,9092/TCP,9093/TCP   67s
nodejs-kafka-cluster-kafka-brokers      ClusterIP   None             <none>        9091/TCP,9092/TCP,9093/TCP   67s
nodejs-kafka-cluster-zookeeper-client   ClusterIP   172.25.25.141    <none>        2181/TCP                     118s
nodejs-kafka-cluster-zookeeper-nodes    ClusterIP   None             <none>        2181/TCP,2888/TCP,3888/TCP   118s
```

Test with kafka-producer and kafka-consumer

Open a terminal and run
```
oc run kafka-producer -ti --image=strimzi/kafka:0.20.1-kafka-2.6.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --broker-list nodejs-kafka-cluster-kafka-bootstrap:9092 --topic foobar-topic
```

Open another terminal and run
```
oc run kafka-consumer -ti --image=strimzi/kafka:0.20.1-kafka-2.6.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server nodejs-kafka-cluster-kafka-bootstrap:9092 --topic foobar-topic --from-beginning
```

