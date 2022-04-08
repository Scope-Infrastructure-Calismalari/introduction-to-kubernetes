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

[Return to the Kubernetes Components page.](KubernetesComponents.md)

# Control Plane Node Components

The Control Plane is what controls the cluster and makes it function. It consists of multiple components that can run on a single master node or be split across multiple nodes and replicated to ensure high availability. The container orchestration layer that exposes the API and interfaces to define, deploy, and manage the lifecycle of containers.

The Kubernetes cp runs various server and manager processes for the cluster. Among the components of the master node are the kube-apiserver, the kube-scheduler, and the etcd database. As the software has matured, new components have been created to handle dedicated needs, such as the cloud-controller-manager; it handles tasks once handled by the kube-controller-manager to interact with other tools, such as Rancher or DigitalOcean for third-party cluster management and reporting. 

There are several add-ons which have become essential to a typical production cluster, such as DNS services. Others are third-party solutions where Kubernetes has not yet developed a local component, such as cluster-level logging and resource monitoring.

As a concept, the various pods responsible for ensuring the current state of the cluster matches the desired state are called the control plane.

When building a cluster using kubeadm, the kubelet process is managed by systemd. Once running, it will start every pod found in /etc/kubernetes/manifests/.

<img src=".\images\p2_kubernetes_components.jpg"/>

## kube-apiserver

The Kubernetes API server is the central component used by all other components and by clients, such as kubectl. It provides a CRUD (Create, Read, Update, Delete) interface for querying and modifying the cluster state over a RESTful API. It stores that state in etcd.

In addition to providing a consistent way of storing objects in etcd, it also performs validation of those objects, so clients can’t store improperly configured objects 

The API server is a component of the Kubernetes control plane that exposes the Kubernetes API. The API server is the front end for the Kubernetes control plane. You and the other Control Plane components communicate with Kubernetes Cluster via API server.

The kube-apiserver is central to the operation of the Kubernetes cluster. All calls, both internal and external traffic, are handled via this agent. All actions are accepted and validated by this agent, and it is the only connection to the etcd database. It validates and configures data for API objects, and services REST operations. As a result, it acts as a cp process for the entire cluster, and acts as a frontend of the cluster's shared state.

Starting as a beta feature in v1.18, the Konnectivity service provides the ability to separate user-initiated traffic from server-initiated traffic. Until these features are developed, most network plugins commingle the traffic, which has performance, capacity, and security ramifications.

## kube-scheduler

Scheduler wait for newly created pods through the API server’s watch mechanism and assign a node to each new pod that doesn’t already have the node set.

All the Scheduler does is update the pod definition through the API server. The API server then notifies the Kubelet (through the watch mechanism) that the pod has been scheduled. As soon as the Kubelet on the target node sees the pod has been scheduled to its node, it creates and runs the pod’s containers.

Schedules the applications. Assigns a worker node to each deployable component of the application. Factors taken into account for scheduling decisions include: individual and collective resource requirements, hardware/software/policy constraints, affinity and anti-affinity specifications, data locality, inter-workload interference, and deadlines.

The kube-scheduler uses an algorithm to determine which node will host a Pod of containers. The scheduler will try to view available resources (such as volumes) to bind, and then try and retry to deploy the Pod based on availability and success. The scheduler uses pod-count by default, but complex configuration is often done if cluster-wide metrics are collected.

There are several ways you can affect the algorithm, or a custom scheduler could be used instead. You can also bind a Pod to a particular node, though the Pod may remain in a pending state due to other settings. A Pod can also be assigned bind to a particular node in the pod spec, though the Pod may remain in a pending state if the node or other declared resource is unavailable.

One of the first settings referenced is if the Pod can be deployed within the current quota restrictions. If so, then the taints and tolerations, and labels of the Pods are used along with the metadata of the nodes to determine the proper placement. Some is done as an admission controller in the kube-apiserver, the rest is done by the chosen scheduler.

## etcd

Consistent and highly-available key value data store that persistently stores the cluster configuration.

The state of the cluster, networking, and other persistent information is kept in an etcd database, or, more accurately, a b+tree key-value store. Rather than finding and changing an entry, values are always appended to the end. Previous copies of the data are then marked for future removal by a compaction process. It works with curl and other HTTP libraries, and provides reliable watch queries.

Simultaneous requests to update a value all travel via the kube-apiserver, which then passes along the request to etcd in a series. The first request would update the database. The second request would no longer have the same version number, in which case the kube-apiserver would reply with an error 409 to the requester. There is no logic past that response on the server side, meaning the client needs to expect this and act upon the denial to update.

There is a Leader database along with possible followers, or non-voting Learners who are in the process of joining the cluster.  They communicate with each other on an ongoing basis to determine which will be the Leader, and determine another in the event of failure. While very fast and potentially durable, there have been some hiccups with new tools, such as kubeadm, and features like whole cluster upgrades. The kubeadm cluster creation tool allows easy deployment of a multi-master cluster with stacked etcd or an external database cluster.

While most Kubernetes objects are designed to be decoupled, a transient microservice which can be terminated without much concern etcd is the exception. As it is, the persistent state of the entire cluster must be protected and secured. Before upgrades or maintenance, you should plan on backing up etcd. The etcdctl command allows for snapshot save and snapshot restore.

## kube-controller-manager

Performs cluster-level functions, such as replicating components, keeping track of worker nodes, handling node failures, and so on.

The kube-controller-manager is a core control loop daemon which interacts with the kube-apiserver to determine the state of the cluster. If the state does not match, the manager will contact the necessary controller to match the desired state. There are several operators in use, such as endpoints, namespace, and replication. The full list has expanded as Kubernetes has matured. 

## Controllers running inside Controller Manager

The single Controller Manager process currently combines a multitude of controllers performing various reconciliation tasks.

The list of these controllers includes the:

- Replication Manager (a controller for ReplicationController resources)
- ReplicaSet, DaemonSet, and Job controllers
- Deployment controller
- StatefulSet controller
- Node controller
- Service controller
- Endpoints controller
- Namespace controller
- PersistentVolume controller
- Others

## cloud-controller-manager

A Kubernetes control plane component that embeds cloud-specific control logic. The cloud controller manager lets you link your cluster into your cloud provider's API, and separates out the components that interact with that cloud platform from components that only interact with your cluster.

Remaining in beta since v1.11, the cloud-controller-manager (ccm) interacts with agents outside of the cloud. It handles tasks once handled by kube-controller-manager. This allows faster changes without altering the core Kubernetes control process. Each kubelet must use the --cloud-provider-external settings passed to the binary. You can also develop your own ccm, which can be deployed as a daemonset as an in-tree deployment or as a free-standing out-of-tree installation. The cloud-controller-manager is an optional agent which takes a few steps to enable.

## kube-dns

Depending on which network plugin has been chosen, there may be various pods to control network traffic. To handle DNS queries, Kubernetes service discovery, and other functions, the CoreDNS server has replaced kube-dns. Using chains of plugins, one of many provided or custom written, the server is easily extensible.
