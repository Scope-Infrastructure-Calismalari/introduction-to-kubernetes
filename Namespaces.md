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

## Namespace

Namespaces are an API construct to avoid naming collisions and represent a scope for object names. A good use case for namespaces is to isolate the objects by team or responsibility. 

A single cluster should be able to satisfy the needs of multiple users or groups of users. Kubernetes namespaces help different projects, teams, or customers to share a Kubernetes cluster. Kubernetes namespaces also provide a scope for objects names. Using multiple namespaces allows you to split complex systems with numerous components into smaller distinct groups. Resource names only need to be unique within a namespace. Two different namespaces can contain resources of the same name. 

The term namespace is used to reference both the kernel feature and the segregation of API objects by Kubernetes. Both are means to keep resources distinct. 

Every API call includes a namespace, using default if not otherwise declared: 
    
    https://10.128.0.3:6443/api/v1/namespaces/default/pods. 

Namespaces, a Linux kernel feature that segregates system resources, are intended to isolate multiple groups and the resources they have access to work with via quotas. Eventually, access control policies will work on namespace boundaries, as well. One could use labels to group resources for administrative reasons. 

### Listing Namespaces

A Kubernetes cluster starts out with four initial namespaces. You can list them with the following command:

```shell
kubectl get namespaces

NAME              STATUS   AGE
default           Active   1d
kube-node-lease   Active   1d
kube-public       Active   1d
kube-system       Active   1d
```

The default namespace hosts object that haven’t been assigned to an explicit namespace. Namespaces starting with the prefix kube- are not considered end user-namespaces. They have been created by the Kubernetes system. You will not have to interact with them as an application developer.

* <i>**default**</i>:  The default namespace for objects with no other namespace. This is where all the resources are assumed, unless set otherwise.

* <i>**kube-system**</i>: The namespace for objects created by the Kubernetes system. This namespace contains infrastructure pods.

* <i>**kube-public**</i>: This namespace is created automatically and is readable by all users. This namespace is mostly reserved for cluster usage, in case that some resources should be visible and readable publicly throughout the whole cluster. A namespace readable by all, even those not authenticated. General information is often included in this namespace.

* <i>**kube-node-lease**</i>: This namespace holds Lease objects associated with each node. Node leases allow the kubelet to send heartbeats so that the control plane can detect node failure. This is the namespace where worker node lease information is kept.

**Should you want to see all the resources on a system, you must pass the --all-namespaces option to the kubectl command.**

### Creating namespaces

A namespace is a Kubernetes resource like any other, so namespaces can be created by posting a YAML file to the Kubernetes API server.

To create a new namespace, use the **create namespace** command. The following command uses the name code-red:

```shell
$ kubectl create namespace custom-namespace
```

The corresponding representation as a YAML manifest would look as follows:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```

Once the namespace is in place, you can create objects within it. You can do so with the command line option --namespace or its short-form -n.

### Discovering namespaces

```shell
kubectl get namespaces

kubectl get pods --namespace kube-system
```

### Working with Namespaces

Take a look at the following commands:​

```shell
​$ kubectl get namespaces
$ kubectl create namespace linuxcon
$ kubectl describe namespace linuxcon
$ kubectl get namespace/linuxcon -o yaml
$ kubectl delete namespace/linuxcon​
```

The above commands show how to view, create and delete namespaces. Note that the describe subcommand shows several settings, such as Labels, Annotations, resource quotas, and resource limits, which we will discus later in the course.

Once a namespace has been created, you can reference it via YAML when creating a resource: 

```shell
$ cat redis.yaml
```

```yaml
apiVersion: V1
kind: Pod
metadata: 
  name: redis 
  namespace: linuxcon
...
```

### kubectl Context's Namespace

You can also specify the default namespace you want to work on, with:

```shell
$ kubectl config set-context --current --namespace=my-ns

Context "minikube" modified.
```
