---
layout: post
title: EKS Node Drainer
subtitle:  Or... How I got Mongo DB to stop having bad days!
---

When I first started using EKS several months back, I was pretty impressed by the platform.  Coming from an on-premise world where you have to manage literally everything from hardware to the Kubernetes infrastructure, EKS provided a breath of fresh air.

However, it didn't take long before the honeymoon phase started to wear off.  Things that I would expect to be in place for a hosted Kubernetes provider just weren't there.  Two of the most major things that stuck out to me were version updates (now available, thanks [CVE-2018-1002105](https://github.com/kubernetes/kubernetes/issues/71411)), and graceful pod draining on any sort of scheduled maintenance such as AMI upgrades or instance resize/scaling.

In other hosted Kubernetes platforms like GKE and AKS, an upgrade initiated on a node will first cordon and gracefully "drain" each node before upgrading/replacing it.  When a drain is initiated on a node, the Kubernetes API performs an "evict" on each pod currently running on the host.  In turn, the evict functionality sends a "SIGTERM" signal to each container in the pod.  In a well built container, this "SIGTERM" will be handled and the container will be gracefully shutdown, performing any preparation steps it needs to do on the way down.

The lack of clean evictions of pods burnt us several times, particularly on stateful applications where abruptly terminating a process could leave the persistent data in an unclean state.  After spending hours recovering from this after planned maintenance, I decided to implement tooling to fill this gap.

# Exploring Options

We looked at several options that we found while googling.  There was a [github issue](https://github.com/awslabs/amazon-eks-ami/issues/66) around this issue, and ways people solved it, but I was unable to find anything that shared a step by step implementation plan and the code to go along with it.

AWS also provides you with [guidance](https://docs.aws.amazon.com/eks/latest/userguide/migrate-stack.html) to use eksctl which will drain nodes enabling blue/green node flips between node pools.  While I would have preferred this, we had business requirements which made this not an option.  We needed a better solution, one which would hook in to the existing workflow and make it seamless for us.

As a result, I decided to build a solution, and for the first time in my career, *truly open-source a project!*

# The Workflow

The workflow is quite simple, once you walk through it.

![EKS Node Drainer Workflow](https://github.com/ryan-a-baker/ryanbakerio/blob/master/img/workflow.png?raw=true){: .center-block :}

First, let's look at how EKS node management works.

The instances in an EKS cluster are typically managed by EC2 Auto-Scaling groups (ASG) which is configured to maintain a fixed number of EC2 instances.  If you were to go in to the AWS console and delete one of the EC2 instances in your cluster, the ASG will automatically launch another node with the configuration specified by the ASG's launch configuration, bringing your node count back to the number of instances specified by the ASG.

ASG’s have a really neat feature called “Lifecycle Hooks” which allow you to perform specific actions on either an instance launch or instance termination.  This lifecycle hook event is then captured by a CloudWatch Event Rule, which can then do many things, including call a Lambda.

So, if we leverage the lifecycle hooks on the EKS clusters auto-scaling group, we can have a CloudWatch rule invoke a Lambda when a node termination happens, which in turn can call the K8S API to drain the node before allowing the termination to continue.

Let's take a look at each of these components in depth to learn how they work.  My project includes a cloudformation that will deploy most of the artifacts, so you don't have to worry about deploying them if you wish to use this, but it's important to know how it all works together.

## Auto-Scaling Group Lifecycle hook

As mentioned, the heart of EKS's fleet management is managed by an [EC2 auto-scaling group](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html).  This auto-scaling group ties together the configuration that EKS instances should be deployed with ([EC2 Launch Configuration](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchConfiguration.html)) with the desired scaling parameters to create a logical grouping of EC2 instances.  The auto-scaling group is what defines the number of nodes that are in your EKS cluster, adjusting the desired number of instances on the auto-scaling group will either launch or terminate instances to match your new desired count.

Often times, before nodes are launched or terminated, some sort of action needs to be taken in order to prepare an application for the addition/removal of an instance.  To facilitate this, Amazon provides an [auto-scaling group lifecycle hook](https://docs.aws.amazon.com/autoscaling/ec2/userguide/lifecycle-hooks.html), which can be used to perform any needed action when a lifecycle event occurs.  Since our desire is to drain all pods off a node before it's terminated, we'll be able to take advantage of this functionality as our initiation point for the workflow, so lets take a look at how to set this up.

The auto-scaling group for the EKS cluster is deployed as part of the [cluster configuration](https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-01-
09/amazon-eks-nodegroup.yaml) that Amazon provides in their [quick-start guide](https://s3.amazonaws.com/aws-quickstart/quickstart-amazon-eks/doc/amazon-eks-architecture.pdf).  Because of this, this is the only part of this deployment that you will have to manually do.  Ideally, it would be best to update the Cloudformation you used to launch your cluster, but given that Amzon has released many versions of this template, it would be difficult to document every permutation, but doing it manually will work fine.  Just make sure if you run an update via Cloudformation after this is added that you ensure the lifecycle hook persists as it could be removed since it is added out-of-band.