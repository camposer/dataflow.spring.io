---
path: 'installation/kubernetes/kubectl'
title: 'kubectl'
description: 'Installation using kubectl'
---

# Deploying with `kubectl`

To install with `kubectl`, you need to get the Kubernetes configuration
files.

They have the required metadata set for service discovery needed by the different applications and services deployed.
To check out the code, enter the following commands:

```bash
git clone https://github.com/spring-cloud/spring-cloud-dataflow
cd spring-cloud-dataflow
git checkout %github-tag%
```

The latest tag is `v%dataflow-version%`.

## Choose a Message Broker

For deployed applications to communicate with each other, you need to select a message broker.
The sample deployment and service YAML files provide configurations for RabbitMQ and Kafka.
You need to configure only one message broker.

- RabbitMQ

  Run the following command to start the RabbitMQ service:

  ```
  kubectl create -f src/kubernetes/rabbitmq/
  ```

  You can use `kubectl get all -l app=rabbitmq` to verify that the deployment, pod, and service resources are running.
  Use `kubectl delete all -l app=rabbitmq` to clean up afterwards.

- Kafka

  Run the following command to start the Kafka service:

  ```
  kubectl create -f src/kubernetes/kafka/
  ```

  You can use `kubectl get all -l app=kafka-broker` to verify that the deployment, pod, and service resources are running.
  Use `kubectl delete all -l app=kafka-broker` to clean up afterwards.

## Deploy Services, Skipper, and Data Flow

You must deploy a number of services and the Data Flow server. The
following subsections describe how to do so:

1.  [Deploy MySQL](#deploy-mysql)

2.  [Deploy Prometheus and Grafana](#deploy-prometheus-and-grafana)

3.  [Create Data Flow Role Bindings and Service Account](#create-data-flow-role-bindings-and-service-account)

4.  [Deploy Skipper](#deploy-skipper)

5.  [Deploy the Data Flow Server](#deploy-data-flow-server)

### Deploy MySQL

We use MySQL for this guide, but you could use a Postgres or H2 database
instead. We include JDBC drivers for all three of these databases. To
use a database other than MySQL, you must adjust the database URL and
driver class name settings.

<!--TIP-->

**Password Management**

You can modify the password in the `src/kubernetes/mysql/mysql-deployment.yaml` files if you prefer to be more secure.
If you do modify the password, you must also provide it as base64-encoded string in the `src/kubernetes/mysql/mysql-secrets.yaml` file.

<!--END_TIP-->

Run the following command to start the MySQL service:

```
kubectl create -f src/kubernetes/mysql/
```

You can use `kubectl get all -l app=mysql` to verify that the
deployment, pod, and service resources are running. You can use
`kubectl delete all,pvc,secrets -l app=mysql` to clean up afterwards.

### Deploy Prometheus and Grafana

The [Prometheus RSocket](https://github.com/micrometer-metrics/prometheus-rsocket-proxy) allows to establish a persistent bidirectional `RSocket` connections between all Stream and Task applications and one or more `Prometheus RSocket Proxy` instances.
Prometheus is configured to scrape each proxy instance.
Proxies in turn use the connection to pull metrics from each application.
The scraped metrics are viewable through Grafana dashboards.
Out of the box, Grafana dashboard comes pre-configured with a Prometheus data-source connection along with Data Flow specific dashboards to monitor the streaming and/or task applications composed in a data pipeline.

<!--TIP-->

**Memory Resources**

To run Prometheus and Grafana, you need at least 2GB to 3GB of Memory.
If you use Minikube and you want Prometheus and Grafana running in it, you need to be sure to allocate enough resources. The instructions above point to `minikube start --cpus=4 --memory=8192`, but to account for these two components, you need at least 10GB or more of memory.

<!--END_TIP-->

Run the following commands to create the cluster role, binding, and
service account:

```
kubectl create -f src/kubernetes/prometheus/prometheus-clusterroles.yaml
kubectl create -f src/kubernetes/prometheus/prometheus-clusterrolebinding.yaml
kubectl create -f src/kubernetes/prometheus/prometheus-serviceaccount.yaml
```

Run the following commands to deploy Prometheus RSocket Proxy:

```
kubectl create -f src/kubernetes/prometheus-proxy/
```

You can use `kubectl get all -l app=prometheus-proxy` to verify that the deployment, pod, and service resources are running. You can use `kubectl delete all,cm,svc -l app=prometheus-proxy` to clean up afterwards. To cleanup roles, bindings, and the service account for Prometheus proxy, run the following command:

```
kubectl delete clusterrole,clusterrolebinding,sa -l app=prometheus-proxy
```

Run the following commands to deploy Prometheus:

```
kubectl create -f src/kubernetes/prometheus/prometheus-configmap.yaml
kubectl create -f src/kubernetes/prometheus/prometheus-deployment.yaml
kubectl create -f src/kubernetes/prometheus/prometheus-service.yaml
```

You can use `kubectl get all -l app=prometheus` to verify that the
deployment, pod, and service resources are running. You can use
`kubectl delete all,cm,svc -l app=prometheus` to clean up afterwards. To
cleanup roles, bindings, and the service account for Prometheus, run the
following command:

```
kubectl delete clusterrole,clusterrolebinding,sa -l app=prometheus
```

Run the following command to deploy Grafana:

```
kubectl create -f src/kubernetes/grafana/
```

You can use `kubectl get all -l app=grafana` to verify that the deployment, pod, and service resources are running. You can use `kubectl delete all,cm,svc,secrets -l app=grafana` to clean up afterwards.

<!--TIP-->

You should replace the `url` attribute value shown in the following example in `src/kubernetes/server/server-config.yaml` to reflect the address and port Grafana is running on.
On Minikube, you can obtain the value by running the command `minikube service --url grafana`.
This configuration is needed for Grafana links to be accessible when accessing the dashboard from a web browser.

```yml
grafana-info:
  url: 'https://grafana:3000'
```

<!--END_TIP-->

The default Grafana dashboard credentials are a username of `admin` and a password of `password`. You can change these defaults by modifying the `src/kubernetes/grafana/grafana-secret.yaml` file.

In the event that you would not like to deploy Prometheus and Grafana for metrics and monitoring, you should remove the following section of `src/kubernetes/server/server-config.yaml`:

```yaml
applicationProperties:
  stream:
    management:
      metrics:
        export:
          prometheus:
            enabled: true
            rsocket:
              enabled: true
              host: prometheus-proxy
              port: 7001
  task:
    management:
      metrics:
        export:
          prometheus:
            enabled: true
            rsocket:
              enabled: true
              host: prometheus-proxy
              port: 7001
grafana-info:
  url: 'https://grafana:3000'
```

### Create Data Flow Role Bindings and Service Account

To create Role Bindings and Service account, run the following commands:

```
kubectl create -f src/kubernetes/server/server-roles.yaml
kubectl create -f src/kubernetes/server/server-rolebinding.yaml
kubectl create -f src/kubernetes/server/service-account.yaml
```

You can use `kubectl get roles` and `kubectl get sa` to list the
available roles and service accounts.

To cleanup roles, bindings and the service account, use the following
commands:

```
kubectl delete role scdf-role
kubectl delete rolebinding scdf-rb
kubectl delete serviceaccount scdf-sa
```

### Deploy Skipper

Data Flow delegates the streams lifecycle management to Skipper. You
need to deploy [Skipper](https://cloud.spring.io/spring-cloud-skipper/)
to enable the stream management features.

The deployment is defined in the
`src/kubernetes/skipper/skipper-deployment.yaml` file. To control what
version of Skipper gets deployed, you can modify the tag used for the
Docker image in the container specification, as the following example
shows:

```yml
spec:
  containers:
    - name: skipper
      image: springcloud/spring-cloud-skipper-server:%skipper-version% #
```

- You may change the version as you like.

<!--TIP-->

**Multiple platform support**

Skipper includes the concept of
[platforms](https://docs.spring.io/spring-cloud-skipper/docs/current/reference/htmlsingle/#using-platforms),
so it is important to define the "`accounts`" based on the project
preferences.

<!--END_TIP-->

To use RabbitMQ as the messaging middleware, run the following command:

```
kubectl create -f src/kubernetes/skipper/skipper-config-rabbit.yaml
```

To use Apache Kafka as the messaging middleware, run the following
command:

```
kubectl create -f src/kubernetes/skipper/skipper-config-kafka.yaml
```

Additionally, to use the [Apache Kafka Streams
Binder](https://docs.spring.io/spring-cloud-stream/docs/current/reference/htmlsingle/#_apache_kafka_streams_binder),
update the `environmentVariables` attribute to include the Kafka Streams
Binder configuraton in
`src/kubernetes/skipper/skipper-config-kafka.yaml` as follows:

```yaml
environmentVariables: 'SPRING_CLOUD_STREAM_KAFKA_BINDER_BROKERS=${KAFKA_SERVICE_HOST}:${KAFKA_SERVICE_PORT},SPRING_CLOUD_STREAM_KAFKA_BINDER_ZK_NODES=${KAFKA_ZK_SERVICE_HOST}:${KAFKA_ZK_SERVICE_PORT}, SPRING_CLOUD_STREAM_KAFKA_STREAMS_BINDER_BROKERS=${KAFKA_SERVICE_HOST}:${KAFKA_SERVICE_PORT},SPRING_CLOUD_STREAM_KAFKA_STREAMS_BINDER_ZK_NODES=${KAFKA_ZK_SERVICE_HOST}:${KAFKA_ZK_SERVICE_PORT}'
```

Run the following commands to start Skipper as the companion server for
Spring Cloud Data Flow:

```
kubectl create -f src/kubernetes/skipper/skipper-deployment.yaml
kubectl create -f src/kubernetes/skipper/skipper-svc.yaml
```

You can use `kubectl get all -l app=skipper` to verify that the
deployment, pod, and service resources are running. You can use
`kubectl delete all,cm -l app=skipper` to clean up afterwards.

### Deploy Data Flow Server

The deployment is defined in the
`src/kubernetes/server/server-deployment.yaml` file. To control which
version of Spring Cloud Data Flow server gets deployed, modify the tag
used for the Docker image in the container specification, as follows:

```yaml
spec:
  containers:
    - name: scdf-server
      image: springcloud/spring-cloud-dataflow-server:%dataflow-version%
```

You must specify the version of Spring Cloud Data Flow server that you want to deploy.

1. Change the version as you like. This document is based on the
   `%dataflow-version%` release. You can use the docker `latest` tag for
   `BUILD-SNAPSHOT` releases.

2. The Skipper service should be running and the `SPRING_CLOUD_SKIPPER_CLIENT_SERVER_URI` property in `src/kubernetes/server/server-deployment.yaml` should point to it.

The Data Flow Server uses the Fabric8 Java client library to connect to the Kubernetes cluster.
There are [several ways to configure the client](https://github.com/fabric8io/kubernetes-client#configuring-the-client) to connect the cluster.
We use environment variables to set the values needed when deploying the Data Flow server to Kubernetes. We also use
the [Spring Cloud Kubernetes library](https://github.com/spring-cloud/spring-cloud-kubernetes) to access the Kubernetes
[`ConfigMap`](https://kubernetes.io/docs/user-guide/configmap/) and
[`Secrets`](https://kubernetes.io/docs/user-guide/secrets/) settings.

The `ConfigMap` settings for RabbitMQ are specified in the `src/kubernetes/skipper/skipper-config-rabbit.yaml` file and for Kafka in
the `src/kubernetes/skipper/skipper-config-kafka.yaml` file.

MySQL secrets are located in the `src/kubernetes/mysql/mysql-secrets.yaml` file.
If you modified the password for MySQL, you should change it in the `src/kubernetes/mysql/mysql-secrets.yaml` file.
Any secrets have to be provided in base64 encoding.

To create the configuration map with the default settings, run the following
command:

```
kubectl create -f src/kubernetes/server/server-config.yaml
```

Now you need to create the server deployment, by running the following
commands:

```
kubectl create -f src/kubernetes/server/server-svc.yaml
kubectl create -f src/kubernetes/server/server-deployment.yaml
```

You can use `kubectl get all -l app=scdf-server` to verify that the
deployment, pod, and service resources are running. You can use
`kubectl delete all,cm -l app=scdf-server` to clean up afterwards.

You can use the `kubectl get svc scdf-server` command to locate the
`EXTERNAL_IP` address assigned to `scdf-server`. You can use that
address later to connect from the shell. The following example (with
output) shows how to do so:

```
kubectl get svc scdf-server
NAME         CLUSTER-IP       EXTERNAL-IP       PORT(S)    AGE
scdf-server  10.103.246.82    130.211.203.246   80/TCP     4m
```

In this case, the URL you need to use is `https://130.211.203.246`.

If you use Minikube, you do not have an external load balancer, and the
`EXTERNAL_IP` shows as `<pending>`. You need to use the `NodePort`
assigned for the `scdf-server` service. You can use the following
command to look up the URL to use:

```
minikube service --url scdf-server
https://192.168.99.100:31991
```
