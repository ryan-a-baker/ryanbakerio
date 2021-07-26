---
layout: post
title: Using K8S Service Accounts in Kubectl
---

# Introduction

I recently had to create a demo for a continous deployment pipeline via Azure DevOps that was capable of executing the [kpack cli](https://github.com/pivotal/kpack) across kubernetes clusters on all the major public clouds (AKS, GKE, and EKS).  This was slightly annoying because each cloud provider users their own authentication mechanisms, which would have been a pain to handle individually.  

Instead - I decided to create a k8s service account on each cluster and use that so that it was consitent across platforms.  I had tried this several years ago as a "break the glass" scenario for EKS after I broke one of our clusters messing with the aws-iam-authenticator, but never had much luck.  I finally got this working and decided to document it.  I'll also include the commands you can run to create a kubeconfig file via command line to help in a pipeline if you have to do it like I did.

Caution:  This will create an admin service account.  I'd highly suggest scoping it to the action your service account needs, and storing the credentials in a secure way.  You have been warned.

# Steps

The overall steps are:

1. Create a Service Account and map it to cluster-admin
2. Gather the Token, Certificate Authority, and API endpoint from the cluster
3. Create a kubeconfig file using kubectl

Since this was for azure-pipelines - we'll use that as the service account name and cluster role binding.  Feel free to change as needed for your use case.

First, we create the service account:

```
kubectl -n kube-system create serviceaccount azure-pipelines
```

Next, we create the cluster role binding and map it to the cluster-admin

```
kubectl create clusterrolebinding azure-pipelines-cb --clusterrole=cluster-admin --serviceaccount=kube-system:azure-pipelines
```

Next - extract the name of the token that was create for the service account

```
TOKENNAME=`kubectl -n kube-system get serviceaccount/azure-pipelines -o jsonpath='{.secrets[0].name}'`
```

Then - get the token from the service account

```
TOKEN=`kubectl -n kube-system get secret $TOKENNAME -o jsonpath='{.data.token}'| base64 --decode`
```

Then - we'll grab the certificate authority cert so that kubectl can trust the certificate from the cluster

```
CACRT=$(kubectl -n kube-system get secret $TOKENNAME -o jsonpath='{.data.ca\.crt}' | base64 --decode)
```

You will also need the K8S API endpoint.  I won't go through getting that because it's different for each provider.  I'll assume you know how to get it since you've already used kubectl above :)

Now that we have all the information we need, we can create a kubeconfig file.

Set the endpoint for the cluster config.

```
kubectl config set-cluster demo --server=<api endpoint>
```

Now - add the token.

```
kubectl config set-credentials azure-pipelines --token=$TOKEN
```

And the user:

```
kubectl config set-context demo --user=azure-pipelines --cluster=demo
```

Finally - add the certificate authority certificate

```
kubectl config set-cluster demo --embed-certs --certificate-authority <(echo $CACRT)
```

Now your context should be ready to use!

```
kubectl config use-context demo
```




