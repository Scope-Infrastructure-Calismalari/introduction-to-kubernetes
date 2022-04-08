# Introduction to Kubernetes

Editors: **Kaan Keskin, Sezen Erdem**

Date: November 2021

Available at: https://github.com/kaan-keskin/introduction-to-kubernetes

**Resources:**

> - Kubernetes Documentation - https://kubernetes.io/docs/home/
> - Kubernetes in Action - Marko Lukša - Manning Publications
> - Kubernetes Fundamentals (LFS258) - Timothy Serewicz - The Linux Foundation
> - Kubernetes for Developers (LFD259) - Timothy Serewicz - The Linux Foundation
> - Kubernetes Lecture Notes - California Institute of Technology
> - Certified Kubernetes Application Developer (CKAD) Study Guide - Benjamin Muschko - O'Reilly Media
> - Getting Started with Kubernetes - Sander van Vugt - Addison-Wesley Professional

**LEGAL NOTICE: This document is created for educational purposes, and it can not be used for any commercial intentions. If you find this document useful in any means please support the original authors for ethical reasons.** 

[Return to the README page.](README.md)

# Autoscaling

In the autoscaling group we find the Horizontal Pod Autoscalers (HPA). This is a stable resource. HPAs automatically scale Replication Controllers, ReplicaSets, or Deployments based on a target of 50% CPU usage by default. The usage is checked by the kubelet every 30 seconds, and retrieved by the Metrics Server API call every minute. HPA checks with the Metrics Server every 30 seconds. Should a Pod be added or removed, HPA waits 180 seconds before further action. 

Other metrics can be used and queried via REST. The autoscaler does not collect the metrics, it only makes a request for the aggregated information and increases or decreases the number of replicas to match the configuration. 

The Cluster Autoscaler (CA) adds or removes nodes to the cluster, based on the inability to deploy a Pod or having nodes with low utilization for at least 10 minutes. This allows dynamic requests of resources from the cloud provider and minimizes expenses for unused nodes. If you are using CA, nodes should be added and removed through cluster-autoscaler- commands. Scale-up and down of nodes is checked every 10 seconds, but decisions are made on a node every 10 minutes. Should a scale-down fail, the group will be rechecked in 3 minutes, with the failing node being eligible in five minutes. The total time to allocate a new node is largely dependent on the cloud provider. 

Another project still under development is the Vertical Pod Autoscaler. This component will adjust the amount of CPU and memory requested by Pods.

## Autoscaling a Deployment

Scaling a Deployment based on the expected load requires manual supervision via monitoring or manual intervention by changing the number of replicas. Kubernetes offers primitives for taking on this job in an automated fashion: so-called autoscalers. 

On the level of a Deployment, we can differentiate two types of autoscalers:
- **Horizontal Pod Autoscaler (HPA)**: Scales the number of Pod replicas based on CPU and memory thresholds.
- **Vertical Pod Autoscaler (VPA)**: Scales the CPU and memory allocation for existing Pods based on historic metrics.

Both autoscalers use the metrics server. Refer to the instructions on installing and enabling metrics support. 

The HPA is a standard feature of Kubernetes, whereas the VPA has to the supported by your cloud provider as an add-on or needs to be installed manually.

## Horizontal Pod Autoscaler

Figure below shows the use of an HPA that will scale up the number of replicas if an average of 70% CPU utilization is reached across all available Pods controlled by the Deployment.

<img src=".\images\autoscaling-deployment-horizontally.png"/>

A Deployment can be autoscaled using the autoscale deployment command. Provide the name and the thresholds you’d like the autoscaler to act upon. In the following example, we’re specifying a minimum of 2 replicas at any given time, a maximum number of 8 replicas the HPA can scale up to, and the CPU utilization threshold of 70%. Listing the HPAs in the namespace reflects those numbers. 

You can setup an autoscaler (HPA) for your Deployment and choose the minimum and maximum number of Pods you want to run based on the CPU utilization of your existing Pods. You can use the primitive name horizontalpodautoscalers for the command; however, I prefer the short-form notation hpa:

```shell
$ kubectl autoscale deployment my-deploy --cpu-percent=70 --min=2 --max=8

horizontalpodautoscaler.autoscaling/my-deploy autoscaled

$ kubectl get hpa

NAME        REFERENCE             TARGETS        MINPODS  MAXPODS REPLICAS  AGE
my-deploy   Deployment/my-deploy  <unknown>/70%  2        8       2         37s

```

The current status of the HPA shows the upper CPU threshold limit but renders <unknown> for the current CPU consumption. That’s usually the case if the metrics server is not running, is misconfigured, or if the Pod template of the Deployment doesn’t define any resource requirements. Check the events of the HPA using the command kubectl describe hpa my-deploy.

What you should usually see is a percentage number to indicate the current CPU utilization, as shown in the following terminal output. None of the Pods consumes CPU resources at the time of querying the information:

```shell
$ kubectl describe hpa my-deploy

Name:                                                  my-deploy
...
Reference:                                             Deployment/my-deploy
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (0) / 70%
Min replicas:                                          3
Max replicas:                                          8
Deployment pods:                                       3 current / 3 desired
...
```

The API spec autoscaling/v1 only supports autoscaling capabilities based on CPU utilization. The new API spec version autoscaling/v2beta2 provides a more generic approach to defining metric thresholds. For example, you can specify a scaling threshold for memory consumption. Given that the specification is still in its beta state, refer to the Kubernetes documentation for more information.

