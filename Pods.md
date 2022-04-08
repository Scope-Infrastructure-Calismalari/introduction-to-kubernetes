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

# Pods

The whole point of Kubernetes is to orchestrate the lifecycle of a container. We do not interact with particular containers. Instead, the smallest unit we can work with is a Pod. Some would say a pod of whales or peas-in-a-pod. Due to shared resources, the design of a Pod typically follows a one-process-per-container architecture.

Containers in a Pod are started in parallel. As a result, there is no way to determine which container becomes available first inside a pod. The use of InitContainers can order startup, to some extent. To support a single process running in a container, you may need logging, a proxy, or special adapter. These tasks are often handled by other containers in the same pod.

There is only one IP address per Pod, for almost every network plugin. If there is more than one container in a pod, they must share the IP. To communicate with each other, they can either use IPC, the loopback interface, or a shared filesystem.

While Pods are often deployed with one application container in each, a common reason to have multiple containers in a Pod is for logging. You may find the term sidecar for a container dedicated to performing a helper task, like handling logs and responding to requests, as the primary application container may not have this ability. The term sidecar, like ambassador and adapter, does not have a special setting, but refers to the concept of what secondary pods are included to do.

Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.

A Pod is a group of one or more containers, with shared storage and network resources, and a specification for how to run the containers.

CRI interaction with Docker images:

<img src=".\images\CRI-interaction-with-Docker-images.png"/>

In Kubernetes, instead of deploying containers individually, you always deploy and operate on a pod of containers. It’s common for pods to contain only a single container. 

    When a pod contains multiple containers, all of them are always run on a single worker node—it never spans multiple worker nodes. 

<img src=".\images\p3_pod_node_relation.jpg"/>

    In Kubernetes, pods live and die, not individual containers.

    All containers of a pod: 
     - share the same set of Linux namespaces instead of each container having its own set.
     - share the same hostname and network interfaces.
     - share the same IP address and port space.   
     - run under the same IPC namespace and can communicate through IPC.

Pods are logical hosts and behave much like physical hosts or VMs in the non-container world. Processes running in the same pod are like processes running on the same physical or virtual machine, except that each process is encapsulated in a container.

A pod is also the basic unit of scaling. Kubernetes can’t horizontally scale individual containers; instead, it scales whole pods.

Pods in a Kubernetes cluster are used in two main ways:

* <b>Pods that run a single container:</b> The "one-container-per-Pod" model is the most common Kubernetes use case

* <b>Pods that run multiple containers that need to work together:</b> A Pod can encapsulate an application composed of multiple co-located containers that are tightly coupled and need to share resources. These co-located containers form a single cohesive unit of service. For example, one container serving data stored in a shared volume to the public, while a separate sidecar container refreshes or updates those files. The Pod wraps these containers, storage resources, and an ephemeral network identity together as a single unit. The main reason to put multiple containers into a single pod is when the application consists of one main process and one or more complementary processes.

Deciding when to use multiple containers in a pod:
* Do they need to be run together or can they run on different hosts?
* Do they represent a single whole or are they independent components?
* Must they be scaled together or individually?

A container shouldn’t run multiple processes. A pod shouldn’t contain multiple containers if they don’t need to run on the same machine.

<img src=".\images\p3_multicontainer_in_a_pod_misusage.jpg"/>

Usually you don't need to create Pods directly, even singleton Pods. Instead, create them using workload resources such as Deployment or Job.

## Pod Life Cycle Phases

Because Kubernetes is a state engine with asynchronous control loops, it’s possible that the status of the Pod doesn’t show a Running status right away when listing the Pods. It usually takes a couple of seconds to retrieve the image and start the container. Upon Pod creation, the object goes through several life cycle phases.

<img src=".\images\Pod-Life-cycle-Phases.png"/>

Understanding the implications of each phase is important as it gives you an idea about the operational status of a Pod.

- **Pending**: The Pod has been accepted by the Kubernetes system, but one or more of the container images has not been created.
- **Running**: At least one container is still running, or is in the process of starting or restarting.
- **Succeeded**: All containers in the Pod terminated successfully.
- **Failed**: Containers in the Pod terminated, as least one failed with an error.
- **Unknown**: The state of Pod could not be obtained.

## Creating Pods

Pods and other objects can be created in several ways. They can be created by using a generator, which. historically, has changed with each release. 

The run command is the central entry point for creating Pods imperatively. 

```shell
$ kubectl run newpod --image=nginx --generator=run-pod/v1
```

They can be created and deleted using properly formatted JSON or YAML files:

```shell
$ kubectl create -f newpod.yaml

$ kubectl delete -f newpod.yaml
```

Pods are usually created by posting a JSON or YAML manifest to the Kubernetes REST API endpoint. 

The pod definition consists following main parts:
* apiVersion: Kubernetes API version used in the YAML
* kind: the type of resource the YAML is describing
* metadata: Includes the name, namespace, labels, and other information about the pod.
* spec: Contains the actual description of the pod’s contents, such as the pod’s containers, volumes, and other data.
* <i>status: contains the current information about the running pod, such as what condition the pod is in, the description and status of each container, and the pod’s internal IP and other basic info.</i>

    The status part contains read-only runtime data that shows the state of the resource at a given moment. When creating a new pod, you never need to provide the status part.

The following is an example of a Pod which consists of a container running the image nginx:1.14.2.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

Other objects will be created by operators/watch-loops to ensure the specifications and current status are the same. 

## Containers

While Kubernetes orchestration does not allow direct manipulation on a container level, we can manage the resources containers are allowed to consume. 

In the resources section of the PodSpec you can pass parameters which will be passed to the container runtime on the scheduled node: 

```yaml
resources: 
  limits: 
    cpu: "1" 
    memory: "4Gi" 
  requests: 
    cpu: "0.5" 
    memory: "500Mi"
```

Another way to manage resource usage of the containers is by creating a ResourceQuota object, which allows hard and soft limits to be set in a namespace. The quotas allow management of more resources than just CPU and memory and allows limiting several objects. 

A beta feature in v1.12 uses the scopeSelector field in the quota spec to run a pod at a specific priority if it has the appropriate priorityClassName in its pod spec.

## Rendering Pod Details

The rendered table produced by the get command provides high-level information about a Pod. But what if you needed to have a deeper look at the details? The describe command can help:

```shell
$ kubectl describe pods hazelcast
```

The terminal output contains the metadata information of a Pod, the containers it runs and the event log, such as failures when the Pod was scheduled. You can expect the output to be very lengthy.

There’s a way to be more specific about the information you want to render. You can combine the **describe** command with a Unix **grep** command. Say you wanted to identify the image for running in the container:

```shell
$ kubectl describe pods hazelcast | grep Image:
```

## Accessing Logs of a Pod

As application developers, we know very well what to expect in the log files produced by the application we implemented. Runtime failures may occur when operating an application in a container. The logs command downloads the log output of a container. The following output indicates that the Hazelcast server started up successfully:

```shell
$ kubectl logs hazelcast
```

It’s very likely that more log entries will be produced as soon as the container receives traffic from end users. **You can stream the logs with the command line option -f**. This option is helpful if you want to see logs in real time.

Kubernetes tries to restart a container under certain conditions, such as if the image cannot be resolved on the first try. **Upon a container restart, you’ll not have access to the logs of the previous container anymore; the logs command only renders the logs for the current container. However, you can still get back to the logs of the previous container by adding the -p command line option.** You may want to use the option to identify the root cause that triggered a container restart.

## Running Commands in a Container

Part of the testing may be to execute commands within the Pod. What commands are available depend on what was included in the base environment when the image was created. In keeping to a decoupled and lean design, it's possible that there is no shell, or that the Bourne shell is available instead of bash. After testing, you may want to revisit the build and add resources necessary for testing and production.

Use the -it options for an interactive shell instead of the command running without interaction or access.
If you have more than one container, declare which container:

```shell
$ kubectl exec -i​t <Pod-Name> 
```

There are situations that require you to log into the container and explore the file system. Maybe you want to inspect the configuration of your application or debug the current state of your application. You can use the exec command to open a shell in the container to explore it interactively, as follows:

```shell
$ kubectl exec -it hazelcast -- /bin/sh
```

Notice that you do not have to provide the resource type. This command only works for a Pod. **The two dashes (--) separate the exec command and its options from the command you want to run inside of the container.**

It’s also possible to just execute a single command inside of a container. Say you wanted to render the environment variables available to containers without having to be logged in. Just remove the interactive flag -it and provide the relevant command after the two dashes:

```shell
$ kubectl exec hazelcast -- env
```

## Observability

Applications running in containers do not operate under the premise of “fire and forget.” Once the container has been started, you’ll want to know if the application is ready for consumption and is still working as expected in an hour, a week, or a month.

## Testing

With the decoupled and transient nature and great flexibility, there are many possible combinations of deployments. Each deployment would have its own method for testing. No matter which technology is implemented, the goal is the end user getting what is expected. Building a test suite for your newly deployed application will help speed up the development process and limit issues with the Kubernetes integration.

In addition to overall access, building tools which ensure the distributed application functions properly, especially in a transient environment, is a good idea.

While custom-built tools may be best at testing a deployment, there are some built-in kubectl arguments to begin the process. The first one is describe and the next would be logs.

You can see the details, conditions, volumes and events for an object with describe. The Events at the end of the output can be helpful during testing, as it presents a chronological and node view of cluster actions and associated messages. A simple nginx pod may show the following output:

```shell
$ kubectl describe pod test1
```

A next step in testing may be to look at the output of containers within a pod. Not all applications will generate logs, so it could be difficult  to know if the lack of output is due to error or configuration.

```shell
$ kubectl logs test1
```

A different pod may be configured to show lots of log output, such as the etcd pod:

```shell
$ kubectl -n kube-system logs etcd-master    ##<-- Use Tab to complete for your etcd pod
```

## Declaring Environment Variables

Applications need to expose a way to make their runtime behavior configurable. For example, you may want to inject the URL to an external web service or declare the username for a database connection. Environment variables are a common option to provide this runtime configuration.

Defining environment variables in a Pod YAML manifest is relatively easy. Add or enhance the section env of a container. **Every environment variable consists of a key-value pair, represented by the attributes name and value. Kubernetes does not enforce or sanitize typical naming conventions for environment variable keys. It’s recommended to follow the standard of using upper-case letters and the underscore character (_) to separate words.**

To illustrate a set of environment variables, have a look at example below. The code snippet describes a Pod that runs a Java-based application using the Spring Boot framework.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: spring-boot-app
spec:
  containers:
  - image: bmuschko/spring-boot-app:1.5.3
    name: spring-boot-app
    env:
    - name: SPRING_PROFILES_ACTIVE
      value: prod
    - name: VERSION
      value: '1.5.3'
```

## Defining a Command with Arguments

Many container images already define an **ENTRYPOINT** or **CMD** instruction. The command assigned to the instruction is automatically executed as part of the container startup process. For example, the Hazelcast image we used earlier defines the instruction CMD ["/opt/hazelcast/start-hazelcast.sh"].

**In a Pod definition, you can either redefine the image ENTRYPOINT and CMD instructions or assign a command to execute for the container if hasn’t been specified by the image.** You can provide this information with the help of the command and args attributes for a container. The command attribute overrides the image’s ENTRYPOINT instruction. The args attribute replaces the CMD instruction of an image.

Imagine you wanted to provide a command to an image that doesn’t provide one yet. As usual there are two different approaches, imperatively and declaratively. 

We’ll generate the YAML manifest with the help of the run command. The Pod should use the busybox image and execute a shell command that renders the current date every 10 seconds in an infinite loop:

```shell
$ kubectl run mypod --image=busybox -o yaml --dry-run=client --restart=Never > pod.yaml -- /bin/sh -c "while true; do date; sleep 10; done"
```

You could have achieved the same by a combination of the command and args attributes if you were to hand-craft the YAML manifest.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - command: ["/bin/sh"]
    args: ["-c", "while true; do date; sleep 10; done"]
    image: busybox
    name: mypod
  restartPolicy: Never
```

You can quickly verify if the declared command actually does its job. First, we create the Pod instance, then we tail the logs:

```shell
$ kubectl create -f pod.yaml
```

## Removing Pods

Sooner or later you’ll want to delete a Pod. You made a configuration mistake and want to start the question from scratch:

```shell
$ kubectl delete pod hazelcast
```

**Keep in mind that Kubernetes tries to delete a Pod gracefully.** This means that the Pod will try to finish active requests to the Pod to avoid unnecessary disruption to the end user. **A graceful deletion operation can take anywhere from 5-30 seconds.**

```shell
kubectl delete po pod-name

kubectl delete po -l label=value

kubectl delete po --all

kubectl delete ns custom_namespace
```

An alternative way to delete a Pod is to point the delete command to the YAML manifest you used to create it. The behavior is the same:

```shell
$ kubectl delete -f pod.yaml
```
