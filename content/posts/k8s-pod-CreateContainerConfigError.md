---
title: "Troubleshooting and Resolving a Pod Stuck in 'CreateContainerConfigError' in Kubernetes"
date: 2023-01-30T22:38:04+01:00
draft: false
tags: ["kubernetes", "troubleshooting"]
---
The other day I was making changes to my helm charts and, after deploying my application, I noticed that one of my pods was stuck in a `CreateContainerConfigError` state. This is a pretty tricky error because it doesn't give you any details on what the underlying issue could be.

## What is the CreateContainerConfigError?
To understand this, let's look at what happens at deployment time to give you an idea of the flow and what could go wrong at each step. 

When you deploy a pod, the first step is to pull the image from the registry and then create the container. If the **image is not found**, then Kubernetes will return an **ErrImagePull** error. If the image is found, then it will proceed to create the container. 

If the **container creation fails**, then it will return a **CreateContainerError** error. If the container creation succeeds, then Kubernetes will then start the container. 

If the **container start fails**, then it will return a **CreateContainerConfigError** error.

In other words, the error happens when the container is transitioning from a Pending state to a Running state. It is at this point that the deployment configuration will be validated to make sure that the container can be started. If the configuration is invalid, then it will return a **CreateContainerConfigError** error.

## How to Troubleshoot the CreateContainerConfigError

> Disclaimer: There can be many reasons why the container configuration is invalid and it will depende on your specific configuration. I will only be covering the one that I have encountered. If you have encountered a different cause, please leave a comment below.

Because the error happens during the validation of the configuration, a good starting point is to double-check the following:
- is the ConfigMap missing? Is it properly configured?
- is a Secret missing? Is it properly configured?
- is the PersistentVolume missing? Is it properly configured?
- is the Pod being created correctly? Are there any empty or invalid fields?

Now that we understand what the error is, and what we should be looking at, let's look at how to troubleshoot it and narrow down the problem.

### Check the Pod Status

The first thing I did was get the pod that I had the error and that I wanted to drill into.

You can do this by running `kubectl get pods -n <namespace>`.

```bash
~ kubectl get pods -n my-service                                                                   
NAME                                READY   STATUS                       RESTARTS       AGE
my-service-00000000078c9fff-dssbk   0/2     CreateContainerConfigError   1 (10s ago)    28s
my-service-00000000bcddf7d-xfsmk    2/2     Running                      25 (42h ago)   16d
```

### Check the Events

Next, we are interested to see all the events on the pod. 

You can do this by running `kubectl describe pod <pod-name> -n <namespace>` and look at the bottom at the **Events**. This will give you a lot of information about the pod, including the events that have happened to it similar to the following, which has been redacted to remove sensitive information.

```bash
~ kubectl describe pod my-service-00000000078c9fff-dssbk -n my-service 

Name:             my-service-00000000078c9fff-dssbk
Namespace:        my-service
Priority:         0
Service Account:  default
Node:             <node-details>
Start Time:       Wed, 25 Jan 2023 15:36:14 +0100
Labels:           app.kubernetes.io/instance=my-service
                  app.kubernetes.io/name=my-service
                  pod-template-hash=00000000
Annotations:      <annotations>
Status:           Pending
IP:               
IPs:
  IP:           
Controlled By:  ReplicaSet/
Containers:
  my-service:
    Container ID:
    Image:          <image-name>
    Image ID:
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       CreateContainerConfigError
    Ready:          False
    Restart Count:  0
    Limits:
      cpu:     100m
      memory:  128Mi
    Requests:
      cpu:      100m
      memory:   128Mi
    Liveness:   http-get http://:http/ delay=15s timeout=60s period=60s #success=1 #failure=3
    Readiness:  http-get http://:http/ delay=15s timeout=60s period=60s #success=1 #failure=3
    Environment:
    (...)
      AzureWebJobsStorage:                                                  <set to the key 'AzureWebJobsStorage' in secret 'my-service'>                                     Optional: false
      AzureAccessKey:                                                       <set to the key 'AzureAccessKey' in secret 'my-service'>                                          Optional: false
      AzureTopicEndpoint:                                                   <set to the key 'AzureTopicEndpoint' in secret 'my-service'>                                      Optional: false
      ClientId:                                                             <set to the key 'ClientId' in secret 'my-service'>                                                Optional: false
    (...)
    State:          Waiting
    (...)
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  94s                default-scheduler  Successfully assigned my-service/my-service-00000000078c9fff-dssbk to <node-name>
  Normal   Pulled     94s                kubelet            Successfully pulled image "image" in 165.014261ms
  Warning  Failed     77s (x4 over 94s)  kubelet            Error: couldn't find key ClientId in Secret my-service/my-service
(...)
```
The Events section shows a list of all the events that have occurred in the process of creating the pod. 

And here we find the issue. The pod is actually missing a secret, the ClientId in my case, that it needs to start. And that is why the pod is in:
```bash
    State:          Waiting
      Reason:       CreateContainerConfigError
```
If you want to double-check that the secret is missing, you can run `kubectl get secrets -n <namespace>` and check if the secret is not there.

Or you can output it in a JSON format and check that the key is missing by running the following command:

```bash
kubectl get secret my-service -n my-service -o json | jq '.data | map_values(@base64d)'
```

## How to Resolve the CreateContainerConfigError

In my case, because I store my infrastructure and configuration (including the kubernetes secrets) in Terraform, I just needed to add the secret to the Terraform configuration, apply it and because the deployment had already timed out, re-run the deployment. But it would've picked it up automatically if I had applied it a bit sooner.

Now that the pod has its necessary configuration and is valid if we run `kubectl get pods -n <namespace>` again, we can see that the pod is now in a Running state.

```bash
kubectl get pods -n my-service                                                                   
NAME                                READY   STATUS                       RESTARTS       AGE
my-service-00000000078c9fff-dssbk   2/2     Running                       1 (10s ago)    28s
my-service-00000000bcddf7d-xfsmk    2/2     Terminating                  25 (42h ago)   16d
```
And there you have it. You have successfully resolved the CreateContainerConfigError.

This was an easy one, let me know what you encountered in the comments below and how you fixed it.

_Happy Coding and I hope this helps someone!_
