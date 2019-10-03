---
layout: post
title: Scaling Pods based on RabbitMQ Queue Depth
hidden: true
---

# Introduction

Kubernetes has a wonderful feature called [horizontal pod autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) that allows scaling of pods in your deployment based on the load that the service is experiencing.  Out of the box, it has the ability to watch CPU and Memory consumption and scale according to those metrics.  What happens if you have a service that isn't bound my CPU or Memory though?

For example, at the heart of lot of applications are AMQP services (such as RabbitMQ) which are leveraged to pass messages back and forth between services.  Wouldn't it make more sense to scale the services based on the depth of the queue that the service is consuming instead of basing it of CPU or Memory?  This makes a lot more sense, especially if you consider that the consumer of messages may not be CPU or memory bound at all. Basing your HPA off those metrics alone could actually cause more harm than good.

Luckily, Kubernetes allows you to create [custom metrics](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-custom-metrics) which you can base HPA's on.  There are several blog posts about this (especially around http request rates), but it appears that it has moved so fast that even blog posts that are six months old are no longer working.  Google also has some [examples](https://cloud.google.com/kubernetes-engine/docs/tutorials/custom-metrics-autoscaling), but their examples leverage StackDriver.  What if you are on-premise or multi-cloud and can't be bound to Google services?

I've run across too many use cases where scaling pods based on queue depth would be incredibly useful, so I decided focus in and get this working and share it with the world.  After quite a bit of fiddling, I was able to get it working with a combination of [Prometheus](https://prometheus.io/), the [Prometheus Adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter), the [RabbitMQ Exporter](https://github.com/kbudde/rabbitmq_exporter), and some sample [RabbitMQ services](https://github.com/ryan-a-baker/k8s-scaling-demo/tree/master/RabbitMQ-Samples) I created.

# Deployment

It'll probably be easiest to deploy the demo first, then walk through each of the components so you can see them live if you wish.

Everything you need to deploy this demo can be located in the [k8s-scaling-demo repo](https://github.com/ryan-a-baker/k8s-scaling-demo).  There are a few requirements you will need in order to do this demo though:

1.  A Kubernetes Cluster with kubectl setup([minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) will be fine)
2.  Helm deployed to the K8S cluster and a local helm client (Instructions [here](https://helm.sh/docs/using_helm/))
3.  Support for HPA version v2beta2 (`kubectl get apiservices | grep "autoscaling"`)

To make this as easy as possible, I have included a [deploy.sh script](https://github.com/ryan-a-baker/k8s-scaling-demo/blob/master/deploy.sh), which will deploy all the helm charts that are needed, as well as deploying the sample RabbitMQ app and inject 10k messages in to the "task_queue".

To get started, clone the repo the repo locally, then run the deploy script

```
./deploy.sh
```

This will deploy the RabbitMQ, Prometheus, Prometheus Adapter, and a sample RabbitMQ python script.

In order to demonstrate the scaling, the sample python application has two components, a publisher and a worker, which were based off the [RabbitMQ tutorials](https://www.rabbitmq.com/getstarted.html). The publisher will inject the number of requested of messages in to the the RabbitMQ server with a random number of periods (between 2 and 10) in the message.  The worker will then pull the messages off the queue, and sleep one second for each of the periods that in the message.  This was done to simulate handling an event that is not CPU or Memory bound (such as making an API call, performing a SQL query, etc).

Initially, the deployment script will deploy the application with only 1 worker Pod.  Allowed to run, this would take well over 16 hours to clear all the messages in the queue.

Once it's deployed, you can work through the workflow below, which will walk you through each component and how it's configured to make automatic scaling work.  It will also walk you through setting up the HPA to see the pods scale to help empty to the queue.

# Workflow

Let's take a look at all the components that go in to this demo and the workflow:

![Workflow](https://github.com/ryan-a-baker/ryanbakerio/blob/master/_posts/scaling-rabbit-images/workflow.png?raw=true){: .center-block :}

## RabbitMQ Server and the RabbitMQ Exporter

At the heart of this demo is the RabbitMQ service, which is a typical deployment based on the [RabbitMQ helm chart]([bitnami](https://github.com/helm/charts/tree/master/stable/rabbitmq) from Bitnami.  There is a unique aspect to this deployment though, and that is the addition of the RabbitMQ Exporter.  This runs as the "metrics" container inside of the RabbitMQ pod, and it's purpose is to export metrics from RabbitMQ via HTTP in a format that Prometheus can "scrape" to collect data.

With the helm chart for RabbitMQ, it's very easy to enable this as part of your [values file](https://github.com/ryan-a-baker/k8s-scaling-demo/blob/master/charts/rabbitmq/values.yaml#L18-L20)

```
metrics:
  enabled: true
  port: 9090
```

Let's take a look at the RabbitMQ management page by using port-fowarding to access the RabbitMQ management and metrics exporter page:

```
kubectl port-forward -n rabbitmq-scaling-demo svc/rabbitmq-server-scaling-demo 15672:15672 9090:9090
```

With a browser, you can now access the management page at http://127.0.0.1:5672.  The username is `admin-demo` and the password is `dynamicscale123!`

When you initially log you, in you won't see any much, as we haven't created any queues or published any messages.  Let's go ahead and do that now by running the following:

```
kubectl run publish -it --rm --image=theryanbaker/rabbitmq-scaling-demo --restart=Never publish 50
```

This will run a pod on our Kubernetes cluster that injects 50 messages in to the "task_queue".  Now, you can click on the "Queues" tab and you'll see that there was a queue named "task_queue" created, and it should have some messages in it.  

![Message Example](https://github.com/ryan-a-baker/ryanbakerio/blob/master/_posts/scaling-rabbit-images/rabbitmq-manager.png?raw=true){: .center-block :}

The number may now be less than 50 that we published because the worker pod that we deployed earlier is consuming the messages.  You'll also actively be able to see the number falling as the worker pulls messages off the queue.  We'll cover that more in a bit.

Let's also take a look at the RabbitMQ Exporter, which can be viewed at http://127.0.0.1:9090/metrics.  A whole bunch of metrics will be returned which  are all exposed in a format that Prometheus knows how to scrape and understand.  If you do a search on the page for "task_queue", you can see all the metrics related to the queue we created and published messages to above.

The metric we care about looks like the following:

```
# HELP rabbitmq_queue_messages Sum of ready and unacknowledged messages (queue depth).
# TYPE rabbitmq_queue_messages gauge
rabbitmq_queue_messages{durable="true",policy="",queue="task_queue",vhost="/"} 40
```

This is the total number of unprocessed messages in our queue "task_queue".  This will be the metric that we'll use to scale pods on.

Take a look around, there are a ton of awesome metrics that the exporter exposes!

# Prometheus

If you aren't familiar with Prometheus, it's a monitoring system based on a time series database that has become the standard tool to use to monitor both Kubernetes, as well as the workloads running inside of it.  While discussing all the features of Prometheus is outside the scope of this blog post (maybe in the future?), I highly encourage you to check it out if you haven't already.  Coupling this with [Grafana](https://grafana.com/) can give you some insanely powerful visibility in the health and status of your Kubernetes cluster.

For the purpose of this blog post, we're going to leverage Prometheus as the collector of the metrics from the RabbitMQ exporter and in turn, the endpoint that the Prometheus Adapter will use to expose the metrics via the Kubernetes API.

It's already been deployed as part of our deployment, so let's take a look at it by using port forwarding again:

```
kubectl port-forward -n rabbitmq-scaling-demo service/prometheus-scaling-demo-server 8080:80
```

Once that's running, let's take a look at the console at http://localhost:8080/graph.

The landing page won't have much on it, as we have to query for the metric we are interested in, so lets do that.  In the query box, enter the following query in order to see the data it has collected:

```
rabbitmq_queue_messages{kubernetes_name="rabbitmq-server-scaling-demo",kubernetes_namespace="rabbitmq-scaling-demo",queue="task_queue"}
```

Once that's entered, execute the query and the go to the graph page to view the historical number of messages in the queue.  You should see something like this:

![Prometheus Graph](https://github.com/ryan-a-baker/ryanbakerio/blob/master/_posts/scaling-rabbit-images/prometheus_graph.png?raw=true){: .center-block :}

You can see from the graph that I've published 50 messages a couple of times, and the slowly the worker node processed those messages out of the queue.

But how did Prometheus know to collect that data?  That's where the magic comes in.  With Kubernetes, Prometheus can query Kubernetes for resources (such as pods and services) that it should try to "scrape" metrics from.  This is done by adding annotations on to the resource.  Let's take a look at the RabbitMQ-server service:

```
➜  ~ kubectl get service rabbitmq-server-scaling-demo -n rabbitmq-scaling-demo -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    prometheus.io/port: "9090"
    prometheus.io/scrape: "true"
....
```

As you can see, there are annotations that tell Prometheus that this resource should be scraped, and which port to scrape it on.  From there, Prometheus handles the rest.  Magic right?
