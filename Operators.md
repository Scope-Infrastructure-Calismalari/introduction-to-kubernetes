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

# Operators

An important concept for orchestration is the use of operators, otherwise known as controllers or watch-loops. Various operators ship with Kubernetes, and you can create your own, as well. A simplified view of an operator is an agent, or Informer, and a downstream store. Using a DeltaFIFO queue, the source and downstream are compared. A loop process receives an obj or object, which is an array of deltas from the FIFO queue. As long as the delta is not of the type Deleted, the logic of the operator is used to create or modify some object until it matches the specification.

The Informer which uses the API server as a source requests the state of an object via an API call. The data is cached to minimize API server transactions. A similar agent is the SharedInformer; objects are often used by multiple other objects. It creates a shared cache of the state for multiple requests.

A Workqueue uses a key to hand out tasks to various workers. The standard Go work queues of rate limiting, delayed, and time queue are typically used. 

The endpoints, namespace, and serviceaccounts operators each manage the eponymous resources for Pods. Deployments manage replicaSets, which manage Pods running the same podSpec, or replicas.

## Service Operator

With every object and agent decoupled we need a flexible and scalable agent which connects resources together and will reconnect, should something die and a replacement is spawned. A service is an operator which listens to the endpoint operator to provide a persistent IP for Pods. Pods have ephemeral IP addresses chosen from a pool. 

Then the service operator sends messages via the kube-apiserver which forwards settings to kube-proxy on every node, as well as the network plugin such as calico-kube-controllers.

A service also handles access policies for inbound requests, useful for resource control, as well as for security. 

- Connect Pods together 
- Expose Pods to Internet 
- Decouple settings 
- Define Pod access policy.

## Overview

A Kubernetes operator is a method of packaging, deploying, and managing a Kubernetes application. A Kubernetes application is both deployed on Kubernetes and managed using the Kubernetes API (application programming interface) and kubectl tooling.

**A Kubernetes operator is an application-specific controller that extends the functionality of the Kubernetes API to create, configure, and manage instances of complex applications on behalf of a Kubernetes user.**

**It builds upon the basic Kubernetes resource and controller concepts, but includes domain or application-specific knowledge to automate the entire life cycle of the software it manages.**

In Kubernetes, controllers of the control plane implement control loops that repeatedly compare the desired state of the cluster to its actual state. If the cluster's actual state doesn’t match the desired state, then the controller takes action to fix the problem. 

**An operator is a custom Kubernetes controller that uses custom resources (CR) to manage applications and their components. High-level configuration and settings are provided by the user within a CR. The Kubernetes operator translates the high-level directives into the low level actions, based on best practices embedded within the operator’s logic.**

A custom resource is the API extension mechanism in Kubernetes. A custom resource definition (CRD) defines a CR and lists out all of the configuration available to users of the operator. 

The Kubernetes operator watches a CR type and takes application-specific actions to make the current state match the desired state in that resource.

Kubernetes operators introduce new object types through custom resource definitions. Custom resource definitions can be handled by the Kubernetes API just like built-in objects, including interaction via kubectl and inclusion in role-based access control (RBAC) policies.

A Kubernetes operator continues to monitor its application as it runs, and can back up data, recover from failures, and upgrade the application over time, automatically. 

The actions a Kubernetes operator performs can include almost anything: scaling a complex app, application version upgrades, or even managing kernel modules for nodes in a computational cluster with specialized hardware.

## How operators manage Kubernetes applications

Kubernetes can manage and scale stateless applications, such as web apps, mobile backends, and API services, without requiring any additional knowledge about how these applications operate. The built-in features of Kubernetes are designed to easily handle these tasks.

However, stateful applications, like databases and monitoring systems, require additional domain-specific knowledge that Kubernetes doesn’t have. It needs this knowledge in order to scale, upgrade, and reconfigure these applications.

Kubernetes operators encode this specific domain knowledge into Kubernetes extensions so that it can manage and automate an application’s life cycle. 

By removing difficult manual application management tasks, Kubernetes operators make these processes scalable, repeatable, and standardized.

For application developers, operators make it easier to deploy and run the foundation services on which their apps depend. 

For infrastructure engineers and vendors, operators provide a consistent way to distribute software on Kubernetes clusters and reduce support burdens by identifying and correcting application problems. 

Operators allow you to write code to automate a task, beyond the basic automation features provided in Kubernetes. For teams following a DevOps or site reliability engineering (SRE) approach, operators were developed to put SRE practices into Kubernetes. 

The function of the operator pattern is to capture the intentions of how a human operator would manage a service. A human operator needs to have a complete understanding of how an app or service should work, how to deploy it, and how to fix any problems that may occur.

The site reliability engineer or operations team typically writes the software to manage an application, but an operator is designed to take human operational knowledge and encode it into software to manage and deploy Kubernetes workloads while eliminating manual tasks. 

Operators are best built by those that are experts in the business logic of installing, running, and upgrading a particular application.

The creation of an operator often starts with automating the application’s installation and self-service provisioning, and follows with more complex automation capabilities.

There is also a Kubernetes operator software development kit (SDK) that can help you develop your own operator. The SDK provides the tools to build, test, and package operators with a choice of creating operators using Helm charts, Ansible Playbooks or Golang

## Operator Framework

The Operator Framework is an open source project that provides developer and runtime Kubernetes tools, enabling you to accelerate the development of an operator.

The Operator Framework includes:

- **Operator SDK**: Enables developers to build operators based on their expertise without requiring knowledge of Kubernetes API complexities.
- **Operator Lifecycle Management**: Oversees installation, updates, and management of the lifecycle of all of the operators running across a Kubernetes cluster.
- **Operator Metering**: Enables usage reporting for operators that provide specialized services.

## Operators in Kubernetes

> Kubernetes is designed for automation. Out of the box, you get lots of built-in automation from the core of Kubernetes. You can use Kubernetes to automate deploying and running workloads, and you can automate how Kubernetes does that.
Kubernetes’ controllers concept lets you extend the cluster’s behaviour without modifying the code of Kubernetes itself. Operators are clients of the Kubernetes API that act as controllers for a Custom Resource.

## An Example Operator

Some of the things that you can use an operator to automate include:
- deploying an application on demand
- taking and restoring backups of that application’s state
- handling upgrades of the application code alongside related changes such as database schemas or extra configuration settings
- publishing a Service to applications that don’t support Kubernetes APIs to discover them
- simulating failure in all or part of your cluster to test its resilience
- choosing a leader for a distributed application without an internal member election process

What might an Operator look like in more detail? Here’s an example in more detail:
1.	A custom resource named SampleDB, that you can configure into the cluster.
2.	A Deployment that makes sure a Pod is running that contains the controller part of the operator.
3.	A container image of the operator code.
4.	Controller code that queries the control plane to find out what SampleDB resources are configured.
5.	The core of the Operator is code to tell the API server how to make reality match the configured resources.
If you add a new SampleDB, the operator sets up PersistentVolumeClaims to provide durable database storage, a StatefulSet to run SampleDB and a Job to handle initial configuration.
If you delete it, the Operator takes a snapshot, then makes sure that the StatefulSet and Volumes are also removed.
6.	The operator also manages regular database backups. For each SampleDB resource, the operator determines when to create a Pod that can connect to the database and take backups. These Pods would rely on a ConfigMap and / or a Secret that has database connection details and credentials.
7.	Because the Operator aims to provide robust automation for the resource it manages, there would be additional supporting code. For this example, code checks to see if the database is running an old version and, if so, creates Job objects that upgrade it for you.

## Deploying Operators

The most common way to deploy an Operator is to add the Custom Resource Definition and its associated Controller to your cluster. The Controller will normally run outside of the control plane, much as you would run any containerized application. For example, you can run the controller in your cluster as a Deployment.

`https://operatorhub.io/getting-started`

