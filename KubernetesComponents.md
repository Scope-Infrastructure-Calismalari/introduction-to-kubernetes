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

# Kubernetes Components

## Cluster & Node

A <b>Kubernetes cluster</b> is a set of node machines (virtual or physical) for running containerized applications. 

    If you’re running Kubernetes, you’re running a cluster.

A <b>Node</b> is a machine (physical or virtual) on which Kubernetes is installed. A Node is a worker machine, and that is where containers will be launched by Kubernetes.

<img src=".\images\p2_cluster_single_node.jpg"/>

If the Node on which your application is running fails, our application goes down. so we need to have more than one Nodes. A Cluster is a set of Nodes grouped together, If one Node fails, our application is still accessible from the other Nodes. Moreover, having multiple Nodes helps in sharing load as well. 

<img src=".\images\p2_cluster_multi_node.jpg"/>

In a Cluster, we need a master who is responsible for managing the Cluster, store the information about members of the Cluster, monitor the status of Nodes and move the workload of the failed Node to another Worker Node. 

The Master is another Node with Kubernetes installed in it and is configured as a Master. The Master watches over the Nodes in the Cluster and is responsible for the actual orchestration of containers on the Worker Nodes. Master is not required to be a single node. So the management part is called as Control Plane. 

The control plane is responsible for maintaining the desired state of the cluster, such as which applications are running and which container images they use. Nodes actually run the applications and workloads.

<img src=".\images\p2_kubernetes_components.jpg"/>

## Kubernetes Architecture

To quickly demistify Kubernetes, let's have a look at the Kubernetes Architecture graphic, which shows a high-level architecture diagram of the system components. Not all components are shown. Every node running a container would have kubelet and kube-proxy, for example.

<img src=".\images\kubernetes_architecture.png"/>

Kubernetes has the following main components:
- Control Plane and worker nodes 
- Operators 
- Services 
- Pods of containers 
- Namespaces and quotas 
- Network and policies 
- Storage

In its simplest form, Kubernetes is made of control plane nodes (aka cp nodes) and worker nodes, once called minions. The cp runs an API server, a scheduler, various controllers and a storage system to keep the state of the cluster, container settings, and the networking configuration.

Kubernetes exposes an API via the API server. You can communicate with the API using a local client called kubectl or you can write your own client and use curl commands. The kube-scheduler is forwarded the pod spec for running containers coming to the API and finds a suitable node to run those containers. Each node in the cluster runs two processes: a kubelet, which is often a systemd process, not a container, and kube-proxy. The kubelet receives requests to run the containers, manages any resources necessary and works with the container engine to manage them on the local node. The local container engine could be Docker, cri-o, containerd, or some other.

The kube-proxy creates and manages networking rules to expose the container on the network to other containers or the outside world.
Using an API-based communication scheme allows for non-Linux worker nodes and containers. Support for Windows Server 2019 was graduated to Stable with the 1.14 release. Only Linux nodes can be cp of the cluster at this time.

## Terminology

We have learned that Kubernetes is an orchestration system to deploy and manage containers. Containers are not managed individually; instead, they are part of a larger object called a Pod. A Pod consists of one or more containers which share an IP address, access to storage and namespace. Typically, one container in a Pod runs an application, while other containers support the primary application.

Kubernetes uses namespaces to keep objects distinct from each other, for resource control and multi-tenant considerations. Some objects are cluster-scoped, others are scoped to one namespace at a time. As the namespace is a segregation of resources, pods would need to leverage services to communicate.

Orchestration is managed through a series of watch-loops, also called controllers or operators. Each controller interrogates the kube-apiserver for a particular object state, then modifying the object until the declared state matches the current state. These controllers are compiled into the kube-controller-manager, but others can be added using custom resource definitions. The default and feature-filled operator for containers is a Deployment. A Deployment does not directly work with pods. Instead it manages ReplicaSets. The ReplicaSet is an operator which will create or terminate pods according to a podSpec. The podSpec is sent to the kubelet, which then interacts with the container engine to download and make available the required resources, then spawn or terminate containers until the status matches the spec.

<img src=".\images\deployment-replicaset-pod.png"/>

The service operator requests existing IP addresses and information from the endpoint operator, and will manage the network connectivity based on labels. A service is used to communicate between pods, namespaces, and outside the cluster. There are also Jobs and CronJobs to handle single or recurring tasks, among other default operators.

To easily manage thousands of Pods across hundreds of nodes could be difficult. To make management easier, we can use labels, arbitrary strings which become part of the object metadata. These can then be used when checking or changing the state of objects without having to know individual names or UIDs. Nodes can have taints to discourage Pod assignments, unless the Pod has a toleration in its metadata.

There is also space in metadata for annotations which remain with the object but is not used as a selector. This information could be used by the containers, by third-party agents or other tools.

While using lots of smaller, commodity hardware could allow every user to have their very own cluster, often multiple users and teams share access to one or more clusters. This is referred to as multi-tenancy. Some form of isolation is necessary in a multi-tenant cluster, using a combination of the following, which we introduce here but will learn more about in the future:

- **namespace**: A segregation of resources, upon which resource quotas and permissions can be applied. Kubernetes objects may be created in a namespace or cluster-scoped. Users can be limited by the object verbs allowed per namespace. Also the LimitRange admission controller constrains resource usage in that namespace. Two objects cannot have the same Name: value in the same namespace. 
- **context**: A combination of user, cluster name and namespace. A convenient way to switch between combinations of permissions and restrictions. For example you may have a development cluster and a production cluster, or may be part of both the operations and architecture namespaces. This information is referenced from ~/.kube/config. 
- **Resource Limits**: A way to limit the amount of resources consumed by a pod, or to request a minimum amount of resources reserved, but not necessarily consumed, by a pod. Limits can also be set per-namespaces, which have priority over those in the PodSpec. 
- **Pod Security Policies**: A policy to limit the ability of pods to elevate permissions or modify the node upon which they are scheduled. This wide-ranging limitation may prevent a pod from operating properly. The use of PSPs may be replaced by Open Policy Agent in the future. 
- **Network Policies**: The ability to have an inside-the-cluster firewall. Ingress and Egress traffic can be limited according to namespaces and labels as well as typical network traffic characteristics.

## Kubernetes Components

When you install Kubernetes on a system, you're actually installing the following components. 

Kubernetes cluster is split into two parts:

> - [Control Plane Node Components](ControlPlaneNodeComponents.md): 
>   - kube-apiserver
>   - etcd
>   - kube-controller-manager
>   - cluster-controller-manager
>   - kube-scheduler
>   - kube-dns
>   - Container Runtime

> - [Worker Node Components](WorkerNodeComponents.md)    
>   - kubelet 
>   - kube-proxy
>   - Container Runtime

> - Add-on Components   
>   - The Kubernetes DNS server
>   - The Dashboard
>   - An Ingress controller
>   - Heapster
>   - The Container Network Interface network plugin

> - Kubernetes Command Line Interface
>   - kubectl   

<img src=".\images\Kubernetes_Architecture_Simplified.jpg"/>

Kubernetes system components communicate only with the API server. They don’t talk to each other directly. The API server is the only component that communicates with etcd. None of the other components communicate with etcd directly, but instead modify the cluster state by talking to the API server.

The components on the worker nodes all need to run on the same node, the components of the Control Plane can easily be split across multiple servers. There can be more than one instance of each Control Plane component running to ensure high availability. While multiple instances of etcd and API server can be active at the same time and do perform their jobs in parallel, only a single instance of the Scheduler and the Controller Manager may be active at a given time—with the others in standby mode.
