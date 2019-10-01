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

## Overview

Let's take a look at all the components that go in to this:

![Workflow](https://github.com/ryan-a-baker/ryanbakerio/blob/master/_posts/scaling-rabbit-images/workflow.png?raw=true){: .center-block :}
