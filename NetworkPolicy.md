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

# Network Policy

Intra-Pod communication or communication between two containers of the same Pod is completely unrestricted in Kubernetes. Network policies instate rules to control the network traffic either from or to a Pod. You can think of network policies as firewall rules for Pods. It’s best practice to start with a “deny all traffic” rule to minimize the attack vector. From there, you can open access as needed. Learning about the intricacies of network policies requires a bit of hands-on practice, as it is not directly apparent if the rules work as expected.

## Understanding Network Policies

**Within a Kubernetes cluster, any Pod can talk to any other Pod without restrictions using its IP address or DNS name, even across namespaces.** Not only does unrestricted inter-Pod communication pose a potential security risk, it also makes it harder to understand the mental communication model of your architecture.

For example, there’s no good reason to allow a backend application running in a Pod to directly talk to the frontend application running in another Pod. The communication should be directed from the frontend Pod to the backend Pod. A network policy defines the rules that control traffic from and to a Pod.

<img src=".\images\network-policies-traffic-pod.png"/>

Label selection plays a crucial role in defining which Pods a network policy applies to. Furthermore, a network policy defines the direction of the traffic, to allow or disallow. Incoming traffic is called ingress, and outgoing traffic is called egress. For ingress and egress, you can whitelist the sources of traffic like Pods, IP addresses, or ports.

A network policy defines a couple of important attributes, which together forms its set of rules.

Configuration elements of a network policy:

| Attribute | Description |
| :--: | :--: |
| podSelector | Selects the Pods in the namespace to apply the network policy to. |
| policyTypes | Defines the type of traffic (i.e., ingress and/or egress) the network policy applies to. |
| ingress | Lists the rules for incoming traffic. Each rule can define from and ports sections. |
| egress | Lists the rules for outgoing traffic. Each rule can define to and ports sections. |

## Creating Network Policies

The creation of network policies is best explained by example. Let’s say you’re dealing with the following scenario: you’re running a Pod that exposes an API to other consumers. For example, it’s a Pod that handles the processing of payments for other applications. The company you’re working for is in the process of migrating applications from a legacy payment processor to a new one. Therefore, you’ll only want to allow access from the applications that are capable of properly communicating with it. Right now, you have two consumers—a grocery store and a coffee shop—each running their application in a separate Pod. The coffee shop is ready to consume the API of payment processor, but the grocery store isn’t. Figure below shows the Pods and their assigned labels.

Limiting traffic to and from a Pod:

<img src=".\images\limiting-traffic-pod.png"/>

You cannot create a new network policy with the imperative create command. Instead, you will have to use the declarative approach. The YAML manifest in example below, stored in the file networkpolicy-api-allow.yaml, shows a network policy for the scenario described previously.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
spec:
  podSelector:
    matchLabels:
      app: payment-processor
      role: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: coffee-shop
```

Before creating the network policy, you’ll stand up the Pod that runs the payment processor:

```shell
$ kubectl run payment-processor --image=nginx --restart=Never -l app=payment-processor,role=api --port 80

pod/payment-processor created

$ kubectl get pods -o wide

NAME              READY STATUS  RESTARTS AGE   IP        NODE     NOMINATED NODE  READINESS GATES
payment-processor 1/1   Running 0        6m43s 10.0.0.51 minikube <none>          <none>

$ kubectl create -f networkpolicy-api-allow.yaml

networkpolicy.networking.k8s.io/api-allow created
```

**Without a network policy controller, network policies won’t have any effect. You need to configure a network overlay solution that provides this controller.** If you’re testing network policies on minikube, you’ll have to go through some extra steps to install and enable the network provider Cilium. Without adhering to the proper prerequisites, network policies won’t have any effect. 

To verify the correct behavior of the network policy, you’ll emulate the grocery store Pod and the coffeshop Pod. As you can see in the following console output, traffic from the grocery store Pod is blocked:

```shell
$ kubectl run grocery-store --rm -it --image=busybox --restart=Never -l app=grocery-store,role=backend -- /bin/sh

# wget --spider --timeout=1 10.0.0.51

Connecting to 10.0.0.51 (10.0.0.51:80)
wget: download timed out

# exit

pod "grocery-store" deleted
```

Accessing the payment processor from the coffeeshop Pod works perfectly, as the Pod selector matches the label app=coffeeshop:

```shell
$ kubectl run coffeeshop --rm -it --image=busybox --restart=Never -l app=coffee-shop,role=backend -- /bin/sh

# wget --spider --timeout=1 10.0.0.51

Connecting to 10.0.0.51 (10.0.0.51:80)
remote file exists

# exit

pod "coffeshop" deleted
```

## Listing Network Policies

Listing network policies works the same as any other Kubernetes primitive. Use the get command in combination with the resource type networkpolicy, or its short-form, netpol. For the previous network policy, you see a table that renders the name and Pod selector:

```shell
$ kubectl get networkpolicy

NAME         POD-SELECTOR                     AGE
api-allow    app=payment-processor,role=api   83m
```

It’s unfortunate that the output of the command doesn’t give away a lot of information about the rules. To retrieve more information, you have to dig into the details.

## Rendering Network Policy Details

You can inspect the details of a network policy using the describe command. The output renders all the important information: Pod selector, and ingress and egress rules:

```shell
$ kubectl describe networkpolicy api-allow

Name:         api-allow
Namespace:    default
Created on:   2020-09-26 18:02:57 -0600 MDT
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=payment-processor,role=api
  Allowing ingress traffic:
    To Port: <any> (traffic allowed to all ports)
    From:
      PodSelector: app=coffeeshop
  Not affecting egress traffic
  Policy Types: Ingress
```

The network policy details don’t draw a clear picture of the Pods that have been selected based on its rules. It would be extremely useful to be presented with a visual representation. The product Weave Cloud can provide such a visualization to make troubleshooting network policies easier.

https://www.weave.works/docs/cloud/latest/overview/

<img src=".\images\weave-cloud-scope-troubleshooting.png"/>

## Isolating All Pods in a Namespace

The safest approach to writing a new network policy is to define it in a way that disallows all ingress and egress traffic. With those constraints in place, you can define more detailed rules and loosen restrictions gradually. 

Disallowing all traffic with the default policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

The curly braces for spec.podSelector mean “apply to all Pods in the namespace.” The attribute spec.policyTypes defines the types of traffic the rule should apply to.

We can easily verify the correct behavior. Say, we’re dealing with a Pod serving up frontend logic and another Pod that provides the backend functionality. The backend functionality is a basic NGINX web server exposing its endpoint on port 80. First, we’ll create the backend Pod and connect to it from the frontend Pod running the busybox image. We should have no problem connecting to the backend Pod:

```shell
$ kubectl run backend --image=nginx --restart=Never --port=80

pod/backend created

$ kubectl get pods backend -o wide

NAME      READY   STATUS    RESTARTS   AGE   IP          NODE       NOMINATED NODE   READINESS GATES
backend   1/1     Running   0          16s   10.0.0.61   minikube   <none>          <none>

$ kubectl run frontend --rm -it --image=busybox --restart=Never -- /bin/sh

# wget --spider --timeout=1 10.0.0.61

Connecting to 10.0.0.61 (10.0.0.61:80)
remote file exists

# exit

pod "frontend" deleted
```

Now, we’ll go through the same procedure but with the “deny all” network policy put in place. Ingress access to the backend Pod will be blocked:

```shell
$ kubectl create -f networkpolicy-deny-all.yaml

networkpolicy.networking.k8s.io/default-deny-all created

$ kubectl run frontend --rm -it --image=busybox --restart=Never -- /bin/sh

# wget --spider --timeout=1 10.0.0.61

Connecting to 10.0.0.61 (10.0.0.61:80)
wget: download timed out

# exit

pod "frontend" deleted
```

## Restricting Access to Ports

If not specified by a network policy, all ports are accessible. There are good reasons why you may want to restrict access on the port level as well. Say you’re running an application in a Pod that only exposes port 8080 to the outside. While convenient during development, it widens the attack vector on any other port that’s not relevant to the application. Port rules can be specified for ingress and egress as part of a network policy. The definition of a network policy in Example 7-4 allows access on port 8080.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: port-allow
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

