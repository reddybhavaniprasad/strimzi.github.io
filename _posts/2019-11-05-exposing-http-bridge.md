---
layout: post
title:  "Exposing your Apache Kafka cluster through the HTTP bridge"
date: 2019-11-05
author: paolo_patierno
---

In the previous blog [post](https://strimzi.io/2019/07/19/http-bridge-intro.html) about HTTP Bridge, we introduced the new Strimzi HTTP bridge component.
Using the bridge, it is possible to interact with an Apache Kafka cluster through the HTTP/1.1 protocol instead of the native Kafka protocol.
We already covered how simple it is to deploy the bridge through the new `KafkaBridge` custom resource and via the Strimzi Cluster Operator.
We also covered all the REST endpoints that the bridge provides as an API for consumers and producers.
In this blog post we are going to describe how to expose the bridge outside of the Kubernetes (or OpenShift) cluster, where it is running alongside the Kafka cluster.
We will also show real examples using `curl` commands.

<!--more-->

Before starting with the bridge, the main prerequisite is having a Kubernetes cluster and an Apache Kafka cluster already deployed on top of it using the Strimzi Cluster Operator; to do so you can find information in the official [documentation](https://strimzi.io/docs/quickstart/master/).

# Deploy the HTTP bridge

Deploying the bridge on Kubernetes is really easy using the new `KafkaBridge` custom resource provided by the Strimzi Cluster Operator.
For the purpose of this post, you can just use the resource provided as an example in Strimzi.

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: KafkaBridge
metadata:
  name: my-bridge
spec:
  replicas: 1
  bootstrapServers: my-cluster-kafka-bootstrap:9092
  http:
    port: 8080
```

This declaration assumes that the Kafka cluster deployed by the Strimzi Cluster Operator is named `my-cluster` otherwise you have to change the `bootstrapServers` accordingly.
All the configuration parameters for the consumer and producer parts of the bridge are the default ones defined by the Apache Kafka documentation.

Just run the following command in order to apply this custom resource allowing the Strimzi Cluster Operator to deploy the HTTP bridge for you.

```shell
kubectl apply -f examples/kafka-bridge/kafka-bridge.yaml
```

# Create a Kubernetes Ingress

An `Ingress` is a Kubernetes resource for allowing external access via HTTP/HTTPS to internal services, like the HTTP bridge.
Even if you can create an `Ingress` resource, it is possible that it might not work out of the box.
In order to have an Ingress working properly, an Ingress controller needs to be running in the cluster.
There are many different Ingress controllers and Kubernetes already provides two of them:

* [NGINX controller](https://github.com/kubernetes/ingress-nginx)
* [GCE controller](https://github.com/kubernetes/ingress-gce) for the Google Cloud Platform

> If your cluster is running using Minikube, an NGINX ingress controller is provided as an addon that you can enable with the command `minikube addons enable ingress` before starting the cluster; it's in charge of deploying an NGINX reverse proxy pod.

Basically, the external traffic reaches the NGINX reverse proxy which routes the traffic itself based on the rules specified by the `Ingress` resource.
In this case, you need to create an `Ingress` resource defining the rule for routing the HTTP traffic to the Strimzi Kafka Bridge.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-bridge-ingress
spec:
  rules:
  - host: my-bridge.io
    http:
      paths:
      - path: /
        backend:
          serviceName: my-bridge-bridge-service
          servicePort: 8080
```

When the above resource is created, the Strimzi Kafka Bridge is reachable through the `my-bridge.io` host, so you can interact with it using different HTTP methods at the address `http://my-bridge.io:80/<endpoint>` where `<endpoint>` is one of the REST endpoints exposed by the bridge for sending and receiving messages, subscribing to topics and so on.

> If your cluster is running using Minikube, you can use the `nip.io` service to reach the brigde. The bridge address will be like `<minikubeip>.nip.io` where you can get the `<minikubeip>` by running the command `minikube ip` first.

In order to verify that the Ingress is working properly, try to hit the `/healthy` endpoint of the bridge with the following `curl` command.

```shell
curl -v GET http://my-bridge.io/healthy
```

If the bridge is reachable through the Ingress, it will return an HTTP response with status code `200 OK` but an empty body.
Following an example of the output.

```shell
> GET /healthy HTTP/1.1
> Host: my-bridge.io:80
> User-Agent: curl/7.61.1
> Accept: */*
> 
< HTTP/1.1 200 OK
< content-length: 0
```

# Producing messages

Assuming that the Kafka brokers have topic auto creation enabled, we can start immediately to send messages through the `/topic/{topicname}` endpoint exposed by the HTTP bridge.

The HTTP request payload is always a JSON but the message values can be JSON or binary (encoded in base64 because you are sending binary data in a JSON payload so encoding in a string format is needed).

```shell
curl -X POST \
  http://my-bridge.io/topics/my-topic \
  -H 'content-type: application/vnd.kafka.json.v2+json' \
  -d '{
    "records": [
        {
            "key": "key-1",
            "value": "value-1"
        },
        {
            "key": "key-2",
            "value": "value-2"
        }
    ]
}'
```

After writing the messages into the topic, the bridge replies with an HTTP status code `200 OK` and a JSON paylod describing in which partition and at which offset the messages are written.
In this case, the auto-created topic has just one partition, so the response will look something like this:

```json
{ 
   "offsets":[ 
      { 
         "partition":0,
         "offset":0
      },
      { 
         "partition":0,
         "offset":1
      }
   ]
}
```

# Consuming messages

Consuming messages is not so simple as producing because there are several steps to do which involve different endpoints.
First of all, you create a consumer through the `/consumers/{groupid}` endpoint by sending an HTTP POST with a body containing some of the supported configuration parameters, the name of the consumer and the data format (JSON or binary).
In the following snippet, we are going to create a consumer named `my-consumer` that will join the consumer group `my-group`.

```shell
curl -X POST http://my-bridge.io/consumers/my-group \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "name": "my-consumer",
    "format": "json",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": false
  }'
```

After creating a corresponding native Kafka consumer connected to the Kafka cluster, the bridge replies with an HTTP status code `200 OK` and a JSON payload containing the URI that the HTTP client has to use in order to interact with such a consumer for subscribing and receiving messages from topics.

```json
{ 
   "instance_id":"my-consumer",
   "base_uri":"http://my-bridge.io:80/consumers/my-group/instances/my-consumer"
}
```

# Subscribing to the topic

The most common way for a Kafka consumer to get messages from a topic is to subscribe to that topic as part of a consumer group and have partitions assigned automatically.
Using the HTTP bridge, it's possible through an HTTP POST to the `/consumers/{groupid}/instances/{name}/subscription` endpoint providing in a JSON formatted payload the list of topics to subscribe to or a topics pattern.
With the following snippet, the consumer is subscribing to the `my-topic` topic.

```shell
curl -X POST http://my-bridge.io/consumers/my-group/instances/my-consumer/subscription \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "topics": [
        "my-topic"
    ]
}'
```

In this case, the bridge just replies with an HTTP status code `200 OK` with an empty body.

# Consuming messages

The action of consuming messages from a Kafka topic is done with a "poll" operation when we talk about native Kafka consumer.
Typically, a Kafka application has a "poll" loop where the poll operation is called every cycle for getting new messages.
The HTTP bridge provides the same action through the `/consumers/{groupid}/instances/{name}/records` endpoint.
Doing an HTTP GET against the above endpoint, actually does a "poll" for getting the messages from the already subscribed topics.
The first "poll" operation after the subscription doesn't always return records because it just starts the join operation of the consumer to the group and the rebalancing in order to get partitions assigned; doing the next poll actually can return messages if there are any in the topic.

```shell
curl -X GET http://my-bridge.io/consumers/my-group/instances/my-consumer/records \
  -H 'accept: application/vnd.kafka.json.v2+json'
```
When messages are available, the bridge replies with an HTTP status code `200 OK` and the JSON body contains the messages.

```json
[ 
   { 
      "topic":"my-topic",
      "key":"key-1",
      "value":"value-1",
      "partition":0,
      "offset":0
   },
   { 
      "topic":"my-topic",
      "key":"key-2",
      "value":"value-2",
      "partition":0,
      "offset":1
   }
]
```

## Committing offsets

The consumer was created disabling the `enable.auto.commit` configuration parameter.
It means that the consumer has to commit the messages offsets by itself without leveraging the periodic auto-commit feature provided by the native Kafka consumer running on the bridge.
The HTTP bridge provides the `/consumers/{groupid}/instances/{name}/offsets` endpoint to do so.
The consumer has to send a HTTP POST on this endpoint providing a colleaction of topics, partitions and related offsets to commit in JSON format inside the payload.

```
curl -X POST http://my-bridge.io/consumers/my-group/instances/my-consumer/offsets \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "offsets": [
        {
            "topic": "my-topic",
            "partition": 0,
            "offset": 2
        }
    ]
}'
```

> Other than committing a specific offset, the consumer can also send the HTTP POST request on the same endpoint but with an empty payload.
Because no request body is submitted, offsets are committed for all the records that have been received by the consumer.

When the offsets are committed, the bridge replies with an HTTP status code `204 NO CONTENT`.

# Deleting a consumer

After creating and using a consumer, when it is not needed anymore, it is important to delete a consumer for freeing resources on the bridge.
It is possible through an HTTP DELETE on the `/consumers/{groupid}/instances/{name}` endpoint.

```shell
curl -X DELETE http://my-bridge.io/consumers/my-group/instances/my-consumer
```

When the consumer is deleted, the bridge replies with an HTTP status code `204 No Content`.
If the client application doesn't do that, the bridge is able to delete "stale" consumers which doesn't send HTTP request for long time (the timeout is configurable through the `http.timeoutSeconds` parameter in the bridge configuration).
Anyway, by default, this feature is disabled so the client application has to take care to delete HTTP consumers properly.

# Exposing on OpenShift using Route

If you are running an OpenShift cluster (for example using Minishift locally), it is possible to expose the Strimzi HTTP bridge using a `Route`.
An OpenShift Route actually works as a Kubernetes Ingress adding features like TLS re-encryption, TLS passthrough, generated pattern-based hostnames and more but it is OpenShift specific.
What you need in this case is to create a `Route` resource as shown in the following snippet.

```yaml
apiVersion: v1
kind: Route
metadata:
  name: my-bridge-route
spec:
  to:
    kind: Service
    name: my-bridge-bridge-service
  port:
    targetPort: rest-api
```

The above Route declaration assumes that your bridge instance is named `my-bridge` so that the corresponding `Service`, to which the Route will route traffic, is named `my-bridge-bridge-service`.
With the above declaration, the hostname for reaching the bridge will be autogenerated by OpenShift.
In order to check its name you can run the `oc get routes` command.

If you want to specify one, you can do so by setting the `spec.host` field, so that the snippet will be something like the following.

```yaml
apiVersion: v1
kind: Route
metadata:
  name: my-bridge-route
spec:
  host: my-bridge.io
  to:
    kind: Service
    name: my-bridge-bridge-service
  port:
    targetPort: rest-api
```

Finally, instead of creating a `Route` resource applying the above YAML declaration, it is possible to expose the Strimzi HTTP bridge service through the `expose` subcommand of the `oc` tool by running the following command which will create the Route for you.

```shell
oc expose service my-bridge-bridge-service
```

# Conclusion

Using a command line tool such as "curl", a UI such as "Postman" or an HTTP based application developed in your preferred programming language, it is really simple to interact with an Apache Kafka cluster using the HTTP/1.1 protocol thanks to the Strimzi Kafka bridge.
This blog post has shown how it is possible with a few simple operations for both producing and consuming messages.
Of course, the bridge provides more operations than the basic ones as for example seeking the position inside a topic partition, sending messages to a specific partition, committing offsets and more.

If you liked this blog post, star us on [GitHub](https://github.com/strimzi/strimzi-kafka-operator) and follow us on [Twitter](https://twitter.com/strimziio) to make sure you don't miss any of our future blog posts!
