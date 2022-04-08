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

# Understanding Kubernetes

## Definition of Kubernetes

Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem. Kubernetes services, support, and tools are widely available.

Running a container on a laptop is relatively simple. But, connecting containers across multiple hosts, scaling them, deploying applications without downtime, and service discovery among several aspects, can be difficult.

Kubernetes addresses those challenges from the start with a set of primitives and a powerful open and extensible API. The ability to add new objects and controllers allows easy customization for various production needs.

According to the kubernetes.io website, Kubernetes is:

    "an open-source system for automating deployment, scaling, and management of containerized applications"

A key aspect of **Kubernetes** is that it builds on 15 years of experience at **Google** in a project called **borg**.

Google's infrastructure started reaching high scale before virtual machines became pervasive in the datacenter, and containers provided a fine-grained solution for packing clusters efficiently. Efficiency in using clusters and managing distributed applications has been at the core of Google challenges. 

The name Kubernetes originates from Greek, meaning helmsman or pilot. Keeping with the maritime theme of Docker containers, Kubernetes is the pilot of a ship of containers. Due to the difficulty in pronouncing the name, many will use a nickname, K8s, as Kubernetes has eight letters between K and S. K8s as an abbreviation results from counting the eight letters between the "K" and the "s". 

Kubernetes can be an integral part of Continuous Integration/Continuous Delivery (CI/CD), as it offers many of the necessary components.

**Continuous Integration**: A consistent way to build and test software. Deploying new packages with code written each day, or every hour, instead of quarterly. Tools like Helm and Jenkins are often part of this with Kubernetes.

**Continuous Delivery**: An automated way to test and deploy software into various environments. Kubernetes handles the lifecycle of containers and connection of infrastructure resources to make rolling updates and rollbacks easy, among other deployment schemes.

There are several options and possible configurations when building a CI/CD pipeline. Tools such as Jenkins, Spinnaker, GitHub, GitLab and Helm, among others, may be part of your particular pipeline.

## History of Kubernetes

Kubernetes was founded by Joe Beda, Brendan Burns, and Craig McLuckie, who were quickly joined by other Google engineers including Brian Grant and Tim Hockin, and was first announced by Google in mid-2014. 

The original codename for Kubernetes within Google was Project 7, a reference to the Star Trek ex-Borg character Seven of Nine. The seven spokes on the wheel of the Kubernetes logo are a reference to that codename. 

What primarily distinguishes Kubernetes from other systems is its heritage. Kubernetes is inspired by Borg - the internal system used by Google to manage its applications (e.g. Gmail, Apps, GCE). Its development and design are heavily influenced by Google's Borg system and many of the top contributors to the project previously worked on Borg. 

With Google pouring the valuable lessons they learned from writing and operating Borg for over 15 years into Kubernetes, this makes Kubernetes a safe choice when having to decide on what system to use to manage containers. While a powerful tool, part of the current growth in Kubernetes is making it easier to work with and handle workloads not found in a Google data center. 

The original Borg project was written entirely in C++, but the rewritten Kubernetes system is implemented in Go.

<img src=".\images\kuberneteslineage.jpg"/>

To learn more about the ideas behind Kubernetes, you can read the "Large-Scale Cluster Management at Google with Borg" paper.

    Large-scale cluster management at Google with Borg

    Abhishek Verma, Luis Pedrosa, Madhukar R. Korupolu, David Oppenheimer, Eric Tune, John Wilkes 

    Proceedings of the European Conference on Computer Systems (EuroSys), ACM, Bordeaux, France (2015)

    https://ai.google/research/pubs/pub43438

Borg has inspired current data center systems, as well as the underlying technologies used in container runtime today. Google contributed cgroups to the Linux kernel in 2007; it limits the resources used by collection of processes. Both cgroups and Linux namespaces are at the heart of containers today, including Docker.

|||
|-|-|
|2003-2004: Birth of the Borg System|Borg was a large-scale internal cluster management system, which ran hundreds of thousands of jobs, from many thousands of different applications, across many clusters, each with up to tens of thousands of machines.|
|2013: From Borg to Omega|Following Borg, Google introduced the Omega cluster management system, a flexible, scalable scheduler for large compute clusters.|
|2014: Google Introduces Kubernetes|Google introduced Kubernetes as an open source version of Borg|
|2015: The year of Kube v1.0 & CNCF|Kubernetes v1.0 gets released. Along with the release, Google partnered with the Linux Foundation to form the Cloud Native Computing Foundation (CNCF). |

<b>Revision History of Kubernetes </b>

<img src=".\images\p1_kubernetes_versions.jpg"/>

### Future of Kubernetes

Nowadays, there is a growing excitement about ‘serverless’ technologies, and Lambda-based architectures, and some would argue that Kubernetes is moving in the opposite direction.

However, Kubernetes has it’s place in ‘increasingly serverless’ world. Kubernetes has the opportunity to be the servers of the serverless world. Tools like Kubeless and Fission providing equivalents to functions-as-a-service but running within Kubernetes. 

Kubernetes will be everywhere in coming years and more exciting technologies like mesh networks, multi-region Clusters, multi-cloud Clusters, serverless reimagined will be built on top of it.

## Components of Kubernetes

Deploying containers and using Kubernetes may require a change in the development and the system administration approach to deploying applications. In a traditional environment, an application (such as a web server) would be a monolithic application placed on a dedicated server. As the web traffic increases, the application would be tuned, and perhaps moved to bigger and bigger hardware. After a couple of years, a lot of customization may have been done in order to meet the current web traffic needs.

Instead of using a large server, Kubernetes approaches the same issue by deploying a large number of small servers, or microservices. The server and client sides of the application are written to expect that there are many possible agents available to respond to a request. It is also important that clients expect the server processes to die and be replaced, leading to a transient server deployment. Instead of a large Apache web server with many httpd daemons responding to page requests, there would be many nginx servers, each responding.

The transient nature of smaller services also allows for decoupling. Each aspect of the traditional application is replaced with a dedicated, but transient, microservice or agent. To join these agents, or their replacements together, we use services. A service ties traffic from one agent to another (for example, a frontend web server to a backend database) and handles new IP or other information, should either one die and be replaced.

Developers new to Kubernetes sometimes assume it is another virtual-machine manager, similar to what they have been using for decades, and continue to develop applications in the same way as prior to using Kubernetes. This is a mistake. The decoupled, transient, microservice architecture is not the same. Most legacy applications will need to be rewritten to optimally run in a cloud. 

<img src=".\images\ArchitectureDifference.png"/>

In the diagram above we see the legacy deployment strategy on the left with a monolithic applications deployed to nodes. On the right, we see an example of the same functionality, on the same hardware, using multiple microservices.

Communication is entirely API call-driven, which allows for flexibility. Cluster configuration information is stored in a JSON format inside of etcd, but is most often written in YAML by the community. Kubernetes agents convert the YAML to JSON prior to persistence to the database. 

Kubernetes is written in Go Language, a portable language which is like a hybridization between C++, Python, and Java. Some claim it incorporates the best (while some claim the worst) parts of each.

## Why do we need Kubernetes and what it can do? 

Development and deployment of applications has changed in recent years. This change is both a consequence of splitting big monolithic apps into smaller microservices and of the changes in the infrastructure that runs those apps.

### Moving from monolithic apps to microservices

The problems in monolithic applications have forced the community to start splitting complex monolithic applications into smaller independently deployable components called <b>microservices</b>. Each microservice runs as an independent process and communicates with other microservices through simple, well-defined interfaces (APIs).

<img src=".\images\p1_monolith_to_distributed.jpg"/>

Microservices allows to horizontally scale the parts that allow scaling out, and scale the parts that don’t, vertically instead of horizontally.

<img src=".\images\p1_traditional_scaling_microservice.jpg"/>

Microservices also have drawbacks. When your system consists of only a small number of deployable components, managing those components is easy. It’s trivial to decide where to deploy each component, because there aren’t that many choices. When the number of those components increases, deployment-related decisions become increasingly difficult because not only does the number of deployment combinations increase, but the number of inter-dependencies between the components increases by an even greater factor.

Microservices perform their work together as a team, so they need to find and talk to each other. When deploying them, someone or something needs to configure all of them properly to enable them to work together as a single system. With increasing numbers of microservices, this becomes tedious and error-prone, especially when you consider what the ops/sysadmin teams need to do when a server fails.

<b>There is no silver bullet.</b>

Solutions follows: 
* Providing a consistent and isolated environment to each application
* Moving to continuous delivery (DevOps)

<img src=".\images\p1_kubernetes_container_evolution.jpg"/>

Kubernetes provides you with a framework to run distributed systems resiliently. It takes care of scaling and failover for your application, provides deployment patterns, and more.

Kubernetes provides you with:

* <b>Service discovery and load balancing:</b> Kubernetes can expose a container using the DNS name or using their own IP address. If traffic to a container is high, Kubernetes is able to load balance and distribute the network traffic so that the deployment is stable.

* <b>Storage orchestration:</b> Kubernetes allows you to automatically mount a storage system of your choice, such as local storages, public cloud providers, and more.

* <b>Automated rollouts and rollbacks:</b> You can describe the desired state for your deployed containers using Kubernetes, and it can change the actual state to the desired state at a controlled rate. For example, you can automate Kubernetes to create new containers for your deployment, remove existing containers and adopt all their resources to the new container.

* <b>Automatic bin packing:</b> You provide Kubernetes with a cluster of nodes that it can use to run containerized tasks. You tell Kubernetes how much CPU and memory (RAM) each container needs. Kubernetes can fit containers onto your nodes to make the best use of your resources.

* <b>Self-healing:</b> Kubernetes restarts containers that fail, replaces containers, kills containers that don't respond to your user-defined health check, and doesn't advertise them to clients until they are ready to serve.

* <b>Secret and configuration management:</b> Kubernetes lets you store and manage sensitive information, such as passwords, OAuth tokens, and SSH keys. You can deploy and update secrets and application configuration without rebuilding your container images, and without exposing secrets in your stack configuration.

Kubernetes enables you to run your software applications on thousands of computer nodes as if all those nodes were a single, enormous computer. It abstracts away the underlying infrastructure and, by doing so, simplifies development, deployment, and management for both development and the operations teams.

<img src=".\images\p1_kubernetes_userview.jpg"/>

Helps dev teams focus on the core application features

Helps ops teams achieve better resource utilization

Deploying applications through Kubernetes is always the same, whether your cluster contains only a couple of nodes or thousands of them. The size of the cluster makes no difference at all. Additional cluster nodes simply represent an additional amount of resources available to deployed apps.

Kubernetes will run your containerized app somewhere in the cluster, provide information to its components on how to find each other, and keep all of them running. Because your application doesn’t care which node it’s running on, Kubernetes can relocate the app at any time, and by mixing and matching apps, achieve far better resource utilization than is possible with manual scheduling.

## Challenges

Containers provide a great way to package, ship, and run applications - that is the Docker motto. 

The developer experience has been boosted tremendously thanks to containers. Containers, and Docker specifically, have empowered developers with ease of building container images, simplicity of sharing images via Docker registries, and providing a powerful user experience to manage containers.

However, managing containers at scale and designing a distributed application based on microservices' principles may be challenging.

You first need a continuous integration pipeline to build your container images, test them, and verify them. A smart first step is deciding on a continuous integration/continuous delivery (CI/CD) pipeline to build, test and verify container images. Tools such as Spinnaker, Jenkins and Helm can be helpful to use, among other possible tools. This will help with the challenges of a dynamic environment. 

Then, you need a cluster of machines acting as your base infrastructure on which to run your containers. You also need a system to launch your containers, and watch over them when things fail and replace as required. Rolling updates and easy rollbacks of containers is an important feature, and eventually tear down the resource when no longer needed.

All of these actions require flexible, scalable, and easy-to-use network and storage.​ As containers are launched on any worker node, the network must join the resource to other containers, while still keeping the traffic secure from others. We also need a storage structure which provides and keeps or recycles storage in a seamless manner.

When Kubernetes answers these concerns, one of the biggest challenges to adoption is the applications themselves, running inside the container. They need to be written, or re-written, to be truly transient. A good question to ponder: If you were to deploy Chaos Monkey, which could terminate any containers at any time, would your customers notice?

## What Kubernetes is not

Kubernetes is not a traditional, all-inclusive PaaS (Platform as a Service) system. 

* Does not limit the types of applications supported. Kubernetes aims to support an extremely diverse variety of workloads, including stateless, stateful, and data-processing workloads. If an application can run in a container, it should run great on Kubernetes.

* Does not deploy source code and does not build your application. 

* Does not provide application-level services, such as middleware (for example, message buses), data-processing frameworks (for example, Spark), databases (for example, MySQL), caches, nor cluster storage systems (for example, Ceph) as built-in services. 

* Does not dictate logging, monitoring, or alerting solutions. 

* Does not provide nor mandate a configuration language/system (for example, Jsonnet). It provides a declarative API that may be targeted by arbitrary forms of declarative specifications.

* Does not provide nor adopt any comprehensive machine configuration, maintenance, management, or self-healing systems.

* Kubernetes is not a mere orchestration system. 

## Running an Application in Kubernetes

To run an application in Kubernetes:
* Package it up into one or more container images
* Push the images to an image registry
* Post a description of your app to the Kubernetes API server

The description includes information about: 
* Container image or images that contain your application components, 
* The relationship between the components, 
* Number of copies (replicas) to be run 
* Whether the components provide a service to either internal or external clients 
* Should be exposed through a single IP address and made discoverable to the other components.

### How it works

<img src=".\images\p1_kubernetes_working_logic.jpg"/>

When the API server processes your app’s description, the Scheduler schedules the specified groups of containers onto the available worker nodes based on computational resources required by each group and the unallocated resources on each node at that moment. The Kubelet on those nodes then instructs the Container Runtime (Docker, for example) to pull the required container images and run the containers.

Once the application is running, Kubernetes continuously makes sure that the deployed state of the application always matches the description you provided. 
    
    For example, if you specify that you always want five instances of a web server running, Kubernetes will always keep exactly five instances running. If one of those instances stops working properly, like when its process crashes or when it stops responding, Kubernetes will restart it automatically.

Similarly, if a whole worker node dies or becomes inaccessible, Kubernetes will select new nodes for all the containers that were running on the node and run them on the newly selected nodes.  

### Scaling the number of copies

While the application is running, you can decide you want to increase or decrease the number of copies, and Kubernetes will spin up additional ones or stop the excess ones, respectively. 

You can even leave the job of deciding the optimal number of copies to Kubernetes. It can automatically keep adjusting the number, based on real-time metrics, such as CPU load, memory consumption, queries per second, or any other metric your app exposes.

## Pros & Cons

Pros:
* Simplify application development
* Achieve better utilization of hardware
* Health checking and self-healing
* Automatic scaling

Cons:
* Can be an overkill for simple applications
* Complex and can reduce productivity
* Transition to Kubernetes can be cumbersome

If done right, investing the time to learn and adopt Kubernetes will often pay off in the future due to better service quality, a higher productivity level and a more motivated workforce.
