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

# Resource Boundaries

A ResourceQuota defines the computing resources (e.g., CPU, RAM, and disk space) available to a namespace to prevent unbounded consumption by Pods running it. You can also limit the number of resource types that are allowed to be created. Accordingly, Pods have to work within those resource boundaries by declaring their minimum and maximum resource expectations. The Kubernetes scheduler will enforce those boundaries upon object creation.

## Understanding Resource Boundaries

**Namespaces do not enforce any quotas for computing resources like CPU, memory, or disk space, nor do they limit the number of Kubernetes objects that can be created.** As a result, Kubernetes objects can consume unlimited resources until the maximum available capacity is reached. In a cloud environment, resources are provisioned on demand as long as you pay the bill. I think we can agree that that approach doesn’t scale well.

Kubernetes measures CPU resources in millicores and memory resources in bytes. That’s why you might see resources defined as 600m or 100Mib. For a deep dive on those resource units, it’s worth cross-referencing the section “Resource units in Kubernetes” in the official documentation.

https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-units-in-kubernetes

## Creating a ResourceQuota

The Kubernetes primitive ResourceQuota establishes the usable, maximum amount of resources per namespace. Once put in place, the Kubernetes scheduler will take care of enforcing those rules. 

The following list should give you an idea of the rules that can be defined:
- Setting an upper limit for the number of objects that can be created for a specific type (e.g., a maximum of 3 Pods).
- Limiting the total sum of compute resources (e.g., 3 GiB of RAM).
- Expecting a Quality of Service (QoS) class for a Pod (e.g., BestEffort to indicate that the Pod must not make any memory or CPU limits or requests).

Creating a ResourceQuota object is usually a task a Kubernetes administrator would take on, but it’s relatively easy to define and create such an object. First, create the namespace the quota should apply to:

```shell
$ kubectl create namespace team-awesome

namespace/team-awesome created

$ kubectl get namespace

NAME           STATUS   AGE
team-awesome   Active   23s
```

Next, define the ResourceQuota in YAML. To demonstrate the functionality of a ResourceQuota, add given constraints to the namespace.
- Limit the number of Pods to 2.
- Define the minimum resources requested by a Pod to 1 CPU and 1024m of RAM.
- Define the maximum resources used by a Pod to 4 CPUs and 4096m of RAM.

Defining hard resource limits with ResourceQuota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: awesome-quota
spec:
  hard:
    pods: 2
    requests.cpu: "1"
    requests.memory: 1024m
    limits.cpu: "4"
    limits.memory: 4096m
```

You’re ready to create a ResourceQuota for the namespace. After it’s created, the object provides a convenient table overview for comparing used resources with the hard limits set by the ResourceQuota spec via the describe command:

```shell
$ kubectl create -f awesome-quota.yaml --namespace=team-awesome

resourcequota/awesome-quota created

$ kubectl describe resourcequota awesome-quota --namespace=team-awesome

Name:            awesome-quota
Namespace:       team-awesome
Resource         Used  Hard
--------         ----  ----
limits.cpu       0     4
limits.memory    0     4096m
pods             0     2
requests.cpu     0     1
requests.memory  0     1024m
```

## Exploring ResourceQuota Enforcement

With the quota rules in place for the namespace team-awesome, we’ll want to see its enforcement in action. We’ll start by creating more than the maximum number of Pods, which is two. To test this, we can create Pods with any definition we like. Say, for example, we use a bare-bones definition that runs the image nginx:1.18.0 in the container.

A Pod without resource requirements:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx:1.18.0
    name: nginx
```

From that YAML definition, let’s create a Pod and see what happens. In fact, Kubernetes will reject the creation of the object with the following error message:

```shell
$ kubectl create -f nginx-pod.yaml --namespace=team-awesome

Error from server (Forbidden): error when creating "nginx-pod.yaml": pods "nginx" is forbidden: failed quota: awesome-quota: must specify limits.cpu,limits.memory,requests.cpu,requests.memory
```

Because we defined minimum and maximum resource requirements for objects in the namespace, we’ll have to ensure that the YAML manifest actually defines them. Modify the initial definition by updating the instruction under resources.

A Pod with resource requirements:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginx:1.18.0
    name: nginx
    resources:
      requests:
        cpu: "0.5"
        memory: "512m"
      limits:
        cpu: "1"
        memory: "1024m"
```

We should be able to create two uniquely named Pods with that manifest, as the combined resource requirements still fit with the boundaries defined in the ResourceQuota:

```shell
$ kubectl create -f nginx-pod1.yaml --namespace=team-awesome

pod/nginx1 created

$ kubectl create -f nginx-pod2.yaml --namespace=team-awesome

pod/nginx2 created

$ kubectl describe resourcequota awesome-quota --namespace=team-awesome

Name:            awesome-quota
Namespace:       team-awesome
Resource         Used   Hard
--------         ----   ----
limits.cpu       2      4
limits.memory    2048m  4096m
pods             2      2
requests.cpu     1      1
requests.memory  1024m  1024m
```

You  may  be  able  to  imagine  what  would  happen  if  we  tried  to  create  another  Pod  with the definition of nginx1 and nginx2. It will fail for two reasons. For one, we’re not allowed to create a third Pod in the namespace, as the maximum number is set to two. Moreover, we’d exceed the alotted maximum for requests.cpu and requests.memory. The following error message provides us with this information:

```shell
$ kubectl create -f nginx-pod3.yaml --namespace=team-awesome

Error from server (Forbidden): error when creating "nginx-pod3.yaml": pods "nginx3" is forbidden: exceeded quota: awesome-quota, requested: pods=1,requests.cpu=500m,requests.memory=512m, used: pods=2,requests.cpu=1, requests.memory=1024m, limited: pods=2,requests.cpu=1,requests.memory=1024m
```

