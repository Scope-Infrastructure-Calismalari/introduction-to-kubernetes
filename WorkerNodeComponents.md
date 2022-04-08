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

# Worker Node Components

All nodes run the kubelet and kube-proxy, as well as the container engine, such as Docker or cri-o, among several options. Other management daemons are deployed to watch these agents or provide services not yet included with Kubernetes.

The kubelet interacts with the underlying container engine also installed on all the nodes, and makes sure that the containers that need to run are actually running. The kubelet agent is the heavy lifter for changes and configuration on worker nodes. It accepts the API calls for Pod specifications (a PodSpec is a JSON or YAML file that describes a Pod). It will work to configure the local node until the specification has been met.

Should a Pod require access to storage, Secrets or ConfigMaps, the kubelet will ensure access or creation. It also sends back status to the kube-apiserver for eventual persistence.

The kube-proxy is in charge of managing the network connectivity to the containers. It does so through the use of iptables entries. It also has the userspace mode, in which it monitors Services and Endpoints using a random port to proxy traffic via ipvs. Use of ipvs can be enabled, with the expectation it will become the default, replacing iptables. A network plugin pod, such as calico-node, may be found depending on the plugin in use.

Each node could run in a different engine. It is likely that Kubernetes will support additional container runtime engines.

Supervisord is a lightweight process monitor used in traditional Linux environments to monitor and notify about other processes. In non-systemd cluster, this daemon can be used to monitor both the kubelet and docker processes. It will try to restart them if they fail, and log events. While not part of a typical installation, some may add this monitor for added reporting.

Kubernetes does not have cluster-wide logging yet. Instead, another CNCF project is used, called Fluentd. When implemented, it provides a unified logging layer for the cluster, which filters, buffers, and routes messages.

Cluster-wide metrics is another area with limited functionality. The metrics-server SIG provides basic node and pod CPU and memory utilization. For more metrics, many use the Prometheus project.

<img src=".\images\p2_kubernetes_components.jpg"/>

## kubelet

The Kubelet is the component responsible for everything running on a worker node. Its initial job is to register the node it’s running on by creating a Node resource in the API server. Then it needs to continuously monitor the API server for Pods that have been scheduled to the node, and start the pod’s containers. It does this by telling the configured container runtime (which is Docker, CoreOS’ rkt, or something else) to run a container from a specific container image. The Kubelet then constantly monitors running containers and reports their status, events, and resource consumption to the API server.

An agent that runs on each node in the cluster. It talks to the API server and manages containers on its node.

The kubelet systemd process is the heavy lifter for changes and configuration on worker nodes. It accepts the API calls for Pod specifications (a PodSpec is a JSON or YAML file that describes a pod). It will work to configure the local node until the specification has been met.

Should a Pod require access to storage, Secrets or ConfigMaps, the kubelet will ensure access or creation. It also sends back status to the kube-apiserver for eventual persistence. 

- Uses PodSpec 
- Mounts volumes to Pod 
- Downloads secrets 
- Passes request to local container engine 
- Reports status of Pods and node to cluster.

Kubelet calls other components such as the Topology Manager, which uses hints from other components to configure topology-aware resource NUMA assignments such as for CPU and hardware accelerators. As an alpha feature, it is not enabled by default.

## kube-proxy

kube-proxy is a network proxy that runs on each node in the cluster. It maintains network rules on nodes. These network rules allow network communication to your Pods from network sessions inside or outside of your cluster.

## Container Run Time

The container runtime is the software that is responsible for running containers.

Kubernetes supports several container runtimes: Docker, containerd, CRI-O, and any implementation of the Kubernetes CRI (Container Runtime Interface).
