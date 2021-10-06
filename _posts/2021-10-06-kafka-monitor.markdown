---
layout: post
title: Continuous Testing Kafka with Xinfra Monitor in Kubernetes
description: How to continuously testing a kafka cluster using xinfra monitor
summary: How to continuously testing a kafka cluster using xinfra monitor
tags: kafka monitor
---


Monitors kafka as part of continuously cheking how the cluster performs. The challenge with this is to create script to produce and consume
with some ludicrous amount of works. Instead of creating stuff in house, a mate refering a fancy tool called Kafka Monitor.

Kafka monitor, or now known as [Xinfra Monitor](https://github.com/linkedin/kafka-monitor) is an open source software created by folks in Linkedin. 
This tools provide the produce, consume and offset test and measurement.<br>
The fun part here, is Jolokia. So, Xinfra Monitor (Kafka Monitor), expose mbeans through Jolokia, so it can be accessed easily by any monitoring tools like Prometheus (plus Jolokia Exporter) or paid one like Datadog.

# Kafka Xinfra Monitor Deployment
In this case I would use kubernetes cluster to host the xinfra monitor, and also use strimzi as kafka and zookeeper operator. For the sake of simplicity, I will host everything on the kubernetes, minikube -in this example.

Prerequisites :
- Minikube | [ref](https://minikube.sigs.k8s.io/docs/start/)
- Kafka and Zookeeper | [ref](https://strimzi.io/docs/operators/in-development/quickstart.html#proc-kafka-cluster-str)

Nice to have locally :
* jq
* curl

Base assumption :

* kafka cluster endpoint : `my-cluster-kafka-bootstrap`, Port : `9092`
* zookeeper cluster endpoint : `my-cluster-zookeeper-client`, Port : `2181`
* Run xinfra monitor in single cluster monitoring mode


## Deploy the Kafka Monitor / Xinfra Monitor

### Initial steps :
* Clone source code 
    
    `git clone git@github.com:linkedin/kafka-monitor.git`
* Checkout to latest release tag. e.g 
    
    `git checkout 2.5.10`


### Modify the Dockerfile
If you can see. There is Dockerfile [here](https://github.com/linkedin/kafka-monitor/blob/2.5.10/docker/Dockerfile). 

```Dockerfile
FROM anapsix/alpine-java

MAINTAINER coffeepac@gmail.com

WORKDIR /opt/kafka-monitor
ADD build/ build/
ADD bin/xinfra-monitor-start.sh bin/xinfra-monitor-start.sh
ADD bin/kmf-run-class.sh bin/kmf-run-class.sh
ADD config/xinfra-monitor.properties config/xinfra-monitor.properties
ADD config/log4j2.properties config/log4j2.properties
ADD docker/kafka-monitor-docker-entry.sh kafka-monitor-docker-entry.sh
ADD webapp/ webapp/

CMD ["/opt/kafka-monitor/kafka-monitor-docker-entry.sh"]
```

>**Side note** : in the Dockerfile above, you will find out that the container will use the properties which can be configured in `config/xinfra-monitor.properties`. With this config, you can set the configuration in granular way, also possible for you to monitor multiple kafka cluster - which I don't really need in this case. Much better way to do this approach, is to convert the config file to kubernetes configmap.


In my case, I will not use the default Dockerfile approach, since I only want to monitor a single Kafka Cluster. So I modified the Dockerfile to this : 

```Dockerfile
FROM anapsix/alpine-java

WORKDIR /opt/kafka-monitor
ADD build/ build/
ADD bin/single-cluster-monitor.sh bin/single-cluster-monitor.sh
ADD bin/kmf-run-class.sh bin/kmf-run-class.sh
ADD config/log4j2.properties config/log4j2.properties

ENTRYPOINT ["/opt/kafka-monitor/bin/single-cluster-monitor.sh"]
```


### Build Docker image

* Build the docker image using existing makefile (yea you can use a vanilla docker build command too)

  ```Bash
  # Go to the docker folder - inside the kafka-monitor repo
  cd docker
  PREFIX=robeevanjava/kafka-monitor TAG=v2.5.10 make container
  ```

  You can change the `PREFIX` environment variable with your docker hub username or even to quay.io. Change the tag to any version you want.

* Push the docker image to docker registry

    ```Bash
    docker push robeevanjava/kafka-monitor:v2.5.10
    ```


### Deploying the Kubernetes Objects

I have create the kubernetes manifest here: [xinfra-monitor-manifest.yaml](https://gist.githubusercontent.com/robertusnegoro/b2a029811ee63725212dabaab38b4fa4/raw/479c86d04291c044feb92d50e78cf73fa80268a1/xinfra-monitor-k8s.yaml)

If you want to use exactly that manifest mentioned above, just do 

```Bash
kubectl apply -f https://git.io/JwLDr -n kafka-monitor
```

If you need to change the kafka and zookeeper cluster, please change the argument lines here :

```yaml
      containers:
      - name: kafka-monitor
        image: robeevanjava/kafka-monitor:v2.5.10
        imagePullPolicy: Always
        env: []
        args:
        - --topic
        - kafkamonitor-topic-test
        - --broker-list
        - my-cluster-kafka-bootstrap.kafka.svc.cluster.local:9092
        - --zookeeper
        - my-cluster-zookeeper-client.kafka.svc.cluster.local:2181
```

_Change the topic, broker-list and zookeeper parameter accordingly._

In the manifest I only expose 1 port on the kubernetes service, and that is for Jolokia [ref](https://jolokia.org/documentation.html), a software that exposing mbeans to http. So it enable us to query the mbeans metric through http request, with json returning value.

To find the available metric, try to do these steps :

* Do port-forwarding to jolokia service port

    ```Bash
    kubectl port-forward svc/kafka-monitor 8778:8778 -n kafka-monitor
     ```
* Make request to jolokia port

    ```Bash
    # Get list of available metric
    curl "http://localhost:8778/jolokia/list"
    # Json pretty print 
     curl "http://localhost:8778/jolokia/list" | jq .
    # Read to a sample metric
    curl "http://localhost:8778/jolokia/read/kafka.consumer:client-id=kmf-consumer,type=consumer-metrics"
    ```

Now you have kafka monitor run on your kubernetes cluster. 


## Optional Steps

Some of the organization uses prometheus as "default" observability toolings. We can use [jolokia-exporter](https://github.com/Scalify/jolokia_exporter) 

>You can clone the repo and build your very own Jolokia docker image and push it to your favourite container registry, but I will use the prebuilt one :)

I have kubernetes manifest [here](https://gist.githubusercontent.com/robertusnegoro/68afab1983e8cbeeb8a1b936f2362821/raw/f5c28949096d25aa2521c4b442229408f1071fe9/jolokia-exporter.yaml)

So to apply it, run 

```Bash
kubectl apply -f https://git.io/JwLim -n kafka-monitor
```

Then you can just do either [ServiceMonitor](https://github.com/prometheus-operator/prometheus-operator/blob/master/Documentation/user-guides/getting-started.md#include-servicemonitors) or write a prometheus scrape config like explained in the example [here](https://github.com/Scalify/jolokia_exporter#readme) 


>PS : If the git.io URL is broken, please refer to each linked gist file above :D