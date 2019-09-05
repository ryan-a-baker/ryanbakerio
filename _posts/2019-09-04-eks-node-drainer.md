When I first started using EKS several months back, I was pretty impressed by the platform.  Coming from an on-premise world where you have to manage literally everything from hardware to the Kubernetes infrastructure, EKS provided a breath of fresh air.

However, it didn't take long before the honeymoon phase started to wear off.  Things that I would expect to be in place for a hosted Kubernetes provider just weren't there.  Two of the most major things that stuck out to me were version updates (now available, thanks [CVE-2018-1002105](https://github.com/kubernetes/kubernetes/issues/71411)), and graceful pod draining on any sort of scheduled maintenance such as AMI upgrades or instance resize/scaling.

In other hosted Kubernetes platforms like GKE and AKS, an upgrade initiated on a node will first cordon and gracefully "drain" each node before upgrading/replacing it.  When a drain is initiated on a node, the Kubernetes API performs an "evict" on each pod currently running on the host.  In turn, the evict functionality sends a "SIGTERM" signal to each container in the pod.  In a well built container, this "SIGTERM" will be handled and the container will be gracefully shutdown, performing any preparation steps it needs to do on the way down.

The lack of clean evictions of pods burnt us several times, particularly on stateful applications where abruptly terminating a process could leave the persistent data in an unclean state.  After spending hours recovering from this after planned maintenance, I decided to implement tooling to fill this gap.

# Exploring Options

We looked at several options that we found while googling.  There was a [github issue](https://github.com/awslabs/amazon-eks-ami/issues/66) around this issue, and ways people solved it, but I was unable to find anything that shared a step by step implementation plan and the code to go along with it.

AWS also provides you with [guidance](https://docs.aws.amazon.com/eks/latest/userguide/migrate-stack.html) to use eksctl which will drain nodes enabling blue/green node flips between node pools.  While I would have preferred this, we had business requirements which made this not an option.  We needed a better solution, one which would hook in to the existing workflow and make it seamless for us.

# The Workflow

The workflow is quite simple, once you walk through it.

![EKS Node Drainer Workflow](https://github.com/ryan-a-baker/ryanbakerio/blob/master/img/workflow.png?raw=true){: .center-block :}

First, let's look at how EKS node management works.

The instances in an EKS cluster are typically managed by EC2 Auto-Scaling groups (ASG) which is configured to maintain a fixed number of EC2 instances.  If you were to go in to the AWS console and delete one of the EC2 instances in your cluster, the ASG will automatically launch another node with the configuration specified by the ASG's launch configuration, bringing your node count back to the number of instances specified by the ASG.

ASG’s have a really neat feature called “Lifecycle Hooks” which allow you to perform specific actions on either an instance launch or instance termination.  This lifecycle hook event is then captured by a CloudWatch Event Rule, which can then do many things, including call a Lambda.

So, if we leverage the lifecycle hooks on the EKS clusters auto-scaling group, we can have a CloudWatch rule invoke a Lambda when a node termination happens, which in turn can call the K8S API to drain the node before allowing the termination to continue.
