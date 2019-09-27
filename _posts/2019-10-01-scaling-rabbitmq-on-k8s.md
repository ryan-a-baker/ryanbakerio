---
layout: post
title: Scaling Pods based on RabbitMQ Queue Depth
hidden: true
---

# Introduction

Kubernetes has a wonderful feature called [horizontal pod autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) that allows scaling of pods in your deployment based on the load that the service is experiencing.  Out of the box, it has the ability to watch CPU and Memory consumption and scale according to those metrics.  What happen if you have a service that isn't bound my CPU or Memory though?

At the heart of lot of applications are AMQP services which are leveraged to pass messages back and forth between services.  Wouldn't it make more sense to scale the services based on the depth of the queue that the service was consuming instead of basing it of CPU or Memory?  This makes a lot more sense, especially if you consider that the consumer of messages may not be CPU or Memory bound at all.

After quite a bit of fiddling, I was able to get this working with a combination of Prometheus, the Prometheus Adapter, and some sample RabbitMQ services I created.  My hope is that sharing this with the world will help prevent people with having as many issues as I had to get it working.
