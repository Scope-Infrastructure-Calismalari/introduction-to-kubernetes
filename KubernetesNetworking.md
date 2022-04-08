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

# Kubernetes Networking

## Component Review

Now that we have seen some of the components, let's take another look with some of the connections shown. Not all connections are shown in the diagram below. Note that all of the components are communicating with kube-apiserver. Only kube-apiserver communicates with the etcd database.

<img src=".\images\k8s_architectural_review.png"/>

We also see some commands, which we may need to install separately to work with various components. There is an etcdctl command to interrogate the database and calicoctl to view more of how the network is configured. We can see Felix, which is the primary Calico agent on each machine. This agent, or daemon, is responsible for interface monitoring and management, route programming, ACL configuration and state reporting.

BIRD is a dynamic IP routing daemon used by Felix to read routing state and distribute that information to other nodes in the cluster. This allows a client to connect to any node, and eventually be connected to the workload on a container, even if not the node originally contacted.

## Single IP per Pod

A pod represents a group of co-located containers with some associated data volumes. All containers in a pod share the same network namespace.

<img src=".\images\kubernetes-network-pod-2.png"/>

The graphic shows a pod with two containers, A and B, and two data volumes, 1 and 2. Containers A and B share the network namespace of a third container, known as the pause container. The pause container is used to get an IP address, then all the containers in the pod will use its network namespace. Volumes 1 and 2 are shown for completeness.

Single IP, Multiple Volumes:

<img src=".\images\NewPodNetwork.png"/>

The diagram above shows a pod with two containers, MainApp and Logger, and two data volumes, made available under two mount points. Containers MainApp and Logger share the network namespace of a third container, known as the pause container. The pause container is used to get an IP address, then all the containers in the pod will use its network namespace. You won’t see this container from the Kubernetes perspective, but you would by running sudo docker ps.

To communicate with each other, containers within pods can use the loopback interface, write to files on a common filesystem, or via inter-process communication (IPC). There is now a network plugin from HPE Labs which allows multiple IP addresses per pod, but this feature has not grown past this new plugin.

Support for dual-stack, IPv4 and IPv6 continues to increase with each release. For example, in a recent release kube-proxy iptables supports both stacks simultaneously.

Starting as an alpha feature in 1.16 is the ability to use IPv4 and IPv6 for pods and services. In the current version, when creating a service, you need to create the network for each address family separately.

## Container to Outside Path

This graphic shows a node with a single, dual-container pod. A NodePort service connects the Pod to the outside network.

<img src=".\images\container_network.png"/>

Even though there are two containers, they share the same namespace and the same IP address, which would be configured by kubectl working with kube-proxy. The IP address is assigned before the containers are started, and will be inserted into the containers. The container will have an interface like eth0@tun10. This IP is set for the life of the pod.

The endpoint is created at the same time as the service. Note that it uses the pod IP address, but also includes a port. The service connects network traffic from a node high-number port to the endpoint using iptables with ipvs on the way. The kube-controller-manager handles the watch loops to monitor the need for endpoints and services, as well as any updates or deletions.

## Services

We can use a service to connect one pod to another, or to outside of the cluster.

<img src=".\images\service_network.png"/>

This graphic shows a pod with a primary container, App, with an optional sidecar Logger. Also seen is the pause container, which is used by the cluster to reserve the IP address in the namespace prior to starting the other pods. This container is not seen from within Kubernetes, but can be seen using docker and crictl.

This graphic also shows a ClusterIP which is used to connect inside the cluster, not the IP of the cluster. As the graphic shows, this can be used to connect to a NodePort for outside the cluster, an IngressController or proxy, or another ”backend” pod or pods.

## Networking Setup

Getting all the previous components running is a common task for system administrators who are accustomed to configuration management. But, to get a fully functional Kubernetes cluster, the network will need to be set up properly, as well.

A detailed explanation about the Kubernetes networking model can be seen on the Cluster Networking page in the Kubernetes documentation.

If you have experience deploying virtual machines (VMs) based on IaaS solutions, this will sound familiar. The only caveat is that, in Kubernetes, the lowest compute unit is not a container, but what we call a pod. 

A pod is a group of co-located containers that share the same IP address. From a networking perspective, a pod can be seen as a virtual machine of physical hosts. The network needs to assign IP addresses to pods, and needs to provide traffic routes between all pods on any nodes. 

The three main networking challenges to solve in a container orchestration system are:

- Coupled container-to-container communication (solved by the pod concept).

- Pod-to-pod communication.

- External-to-pod communication (solved by the services concept, which we will discuss later).

Kubernetes expects the network configuration to enable pod-to-pod communication to be available; it will not do it for you.

Tim Hockin, one of the lead Kubernetes developers, has created a very useful slide deck to understand the Kubernetes networking: An Illustrated Guide to Kubernetes Networking.

https://speakerdeck.com/thockin/illustrated-guide-to-kubernetes-networking

## CNI Network Configuration File

To provide container networking, Kubernetes is standardizing on the Container Network Interface (CNI) specification. Since v1.6.0, the goal of kubeadm (the Kubernetes cluster bootstrapping tool) has been to use CNI, but you may need to recompile to do so.

CNI is an emerging specification with associated libraries to write plugins that configure container networking and remove allocated resources when the container is deleted. Its aim is to provide a common interface between the various networking solutions and container runtimes. As the CNI specification is language-agnostic, there are many plugins from Amazon ECS, to SR-IOV, to Cloud Foundry, and more.

With CNI, you can write a network configuration file:

```json
{
    "cniVersion": "0.2.0",
    "name": "mynet",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "subnet": "10.22.0.0/16",
        "routes": [
            { "dst": "0.0.0.0/0" }
             ]
     }
}
```

This configuration defines a standard Linux bridge named cni0, which will give out IP addresses in the subnet 10.22.0.0./16. The bridge plugin will configure the network interfaces in the correct namespaces to define the container network properly.

## Pod-to-Pod Communication

While a CNI plugin can be used to configure the network of a pod and provide a single IP per pod, CNI does not help you with pod-to-pod communication across nodes.

The requirement from Kubernetes is the following:

- All pods can communicate with each other across nodes.
- All nodes can communicate with all pods.
- No Network Address Translation (NAT).

Basically, all IPs involved (nodes and pods) are routable without NAT. This can be achieved at the physical network infrastructure if you have access to it (e.g. GKE). Or, this can be achieved with a software defined overlay with solutions like:

- Weave
- Flannel
- Calico
- Romana​

Most network plugins now support the use of Network Policies, which act as an internal firewall, limiting ingress and egress traffic.

See the list of networking add-ons for a more complete list.

https://kubernetes.io/docs/concepts/cluster-administration/addons/

