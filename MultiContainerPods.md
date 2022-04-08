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

# Multi-Container Pods

The idea of multiple containers in a Pod goes against the architectural idea of decoupling as much as possible. One could run an entire operating system inside a container, but would lose much of the granular scalability Kubernetes is capable of. But there are certain needs in which a second or third co-located container makes sense. By adding a second container, each container can still be optimized and developed independently, and both can scale and be repurposed to best meet the needs of the workload.

It may not make sense to recreate an entire image to add functionality like a shell or logging agent. Instead, you could add another container to the Pod, which would provide the necessary tools.

Each container in the Pod should be transient and decoupled. If adding another container limits the scalability or transient nature of the original application, then a new build may be warranted.

Every container in a Pod shares a single IP address and namespace. Each container has equal potential access to storage given to the Pod. Kubernetes does not provide any locking, so your configuration or application should be such that containers do not have conflicting writes.

One container could be read only while the other writes. You could configure the containers to write to different directories in the volume, or the application could have built in locking. Without these protections, there would be no way to order containers writing to the storage.

There are three terms often used for multi-container pods: ambassador, adapter, and sidecar. Based off of the documentation, you may think there is some setting for each. There is not. Each term is an expression of what a secondary pod is intended to do. All are just multi-container pods.

Especially to beginners of Kubernetes, how to appropriately design a Pod isn’t necessarily apparent. Upon reading the Kubernetes user documentation and tutorials on the internet, you’ll quickly find out that you can create a Pod that runs multiple containers at the same time. The question often arises, “Should I deploy my microservices stack to a single Pod with multiple containers, or should I create multiple Pods, each running a single microservice?” The short answer is to operate a single microservice per Pod. This modus operandi promotes a decentralized, decoupled, and distributed architecture. Furthermore, it helps with rolling out new versions of a microservice without necessarily interrupting other parts of the system.

So what’s the point of running multiple containers in a Pod then? There are two common use cases. Sometimes, you’ll want to initialize your Pod by executing setup scripts, commands, or any other kind of preconfiguration procedure before the application container should start. This logic runs in a so-called init container. Other times, you’ll want to provide helper functionality that runs alongside the application container to avoid the need to bake the logic into application code. For example, you may want to massage the log output produced by the application. Containers running helper logic are called sidecars.

## Multi-Container Pod Terminology

Real-world scenarios call for running multiple containers inside of a Pod. An init container helps with setting the stage for the main application container by executing initializing logic. Once the initialized logic has been processed, the container will be terminated. The main application container only starts if the init container ran through its functionality successfully.

**Kubernetes enables implementing software engineering best practices like separation of concerns and the single-responsibility principle. Cross-cutting concerns or helper functionality can be run in a so-called sidecar container. A sidecar container lives alongside the main application container within the same Pod and fulfills this exact role.**

The adapter pattern helps with “translating” data produced by the application so that it becomes consumable by third-party services. The ambassador pattern acts as a proxy for the application container when communicating with external services by abstracting the “how.”

### Init Containers

Init containers provide initialization logic concerns to be run before the main application even starts. 

Init containers are always started before the main application containers, which means they have their own lifecycle. To split up the initialization logic, you can even distribute the work into multiple init containers that are run in the order of definition in the manifest. Of course, initialization logic can fail. If an init container produces an error, the whole Pod is restarted, causing all init containers to run again in sequential order. Thus, to prevent any side effects, making init container logic idempotent is a good practice. Figure below shows a Pod with two init containers and the main application.

Sequential and atomic lifecycle of init containers in a Pod

<img src=".\images\sequential-and-atomic-lifecycle-init-containers-pod.png"/>

Not all containers are the same. Standard containers are sent to the container engine at the same time, and may start in any order. LivenessProbes, ReadinessProbes, and StatefulSets can be used to determine the order, but can add complexity. Another option can be an Init container, which must complete before app containers will be started. Should the init container fail, it will be restarted until completion, without the app container running. 

The init container can have a different view of the storage and security settings, which allows utilities and commands to be used, which the application would not be allowed to use.. Init containers can contain code or utilities that are not in an app. It also has an independent security from app containers.

The use of an initContainer allows one or more containers to run only if one or more previous containers run and exit successfully. For example, you could have a checksum verification scan container and a security scan container check the intended containers. Only if both containers pass the checks would the following group of containers be attempted. 

The code below will run the init container until the ls command succeeds; then the database container will start.

```yaml
spec: 
  containers: 
  - name: main-app 
    image: databaseD 
  initContainers:
  - name: wait-database
    image: busybox
    command: ['sh', '-c', 'until ls /db/dir ; do sleep 5; done; '] 
```

A Pod defining an init container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: business-app
spec:
  initContainers:
  - name: configurer
    image: busybox:1.32.0
    command: ['sh', '-c', 'echo Configuring application... && \
              mkdir -p /usr/shared/app && echo -e "{\"dbConfig\": \
              {\"host\":\"localhost\",\"port\":5432,\"dbName\":\"customers\"}}" \
              > /usr/shared/app/config.json']
    volumeMounts:
    - name: configdir
      mountPath: "/usr/shared/app"
  containers:
  - image: bmuschko/nodejs-read-config:1.0.0
    name: web
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: configdir
      mountPath: "/usr/shared/app"
  volumes:
  - name: configdir
    emptyDir: {}
```

When starting the Pod, you’ll see that the status column of the get command provides information on init containers as well. The prefix Init: signifies that an init container is in the process of being executed. The status portion after the colon character shows the number of init containers completed versus the overall number of init containers configured:

```shell
$ kubectl create -f init.yaml

pod/business-app created

$ kubectl get pod business-app

NAME           READY   STATUS    RESTARTS   AGE
business-app   0/1     Init:0/1  0          2s

$ kubectl get pod business-app

NAME           READY   STATUS    RESTARTS   AGE
business-app   1/1     Running   0          8s
```

Errors can occur during the execution of init containers. You can always retrieve the logs of an init container by using the **--container** command-line option (or -c in its short form).

<img src=".\images\pod-targeting-specific-container.png"/>

The following command renders the logs of the configurer init container, which equates to the echo command we configured in the YAML manifest:

```shell
$ kubectl logs business-app -c configurer

Configuring application...
```

### The Sidecar Pattern

The lifecycle of an init container looks as follows: it starts up, runs its logic, then terminates once the work has been done. Init containers are not meant to keep running over a longer period of time. There are scenarios that call for a different usage pattern. For example, you may want to create a Pod that runs multiple containers continuously alongside one another.

Typically, there are two different categories of containers: the container that runs the application and another container that provides helper functionality to the primary application. In the Kubernetes space, the container providing helper functionality is called a sidecar. Among the most commonly used capabilities of a sidecar container are file synchronization, logging, and watcher capabilities. The sidecars are not part of the main traffic or API of the primary application. They usually operate asynchronously and are not involved in the public API.

The idea for a sidecar container is to add some functionality not present in the main container. Rather than bloating code, which may not be necessary in other deployments, adding a container to handle a function such as logging solves the issue, while remaining decoupled and scalable. Prometheus monitoring and Fluentd logging leverage sidecar containers to collect data.

Similar to a sidecar on a motorcycle, it does not provide the main power, but it does help carry stuff. A sidecar is a secondary container which helps or provides a service not found in the primary application. Logging containers are a common sidecar.

To illustrate the behavior of a sidecar, we’ll consider the following use case. The main application container runs a web server—in this case, NGINX. Once started, the web server produces two standard logfiles. The file '/var/log/nginx/access.log' captures requests to the web server’s endpoint. The other file, '/var/log/nginx/error.log', records failures while processing incoming requests.

As part of the Pod’s functionality, we’ll want to implement a monitoring service. The sidecar container polls the file’s error.log periodically and checks if any failures have been discovered. More specifically, the service tries to find failures assigned to the error log level, indicated by [error] in the log file. If an error is found, the monitoring service will react to it. For example, it could send a notification to the administrators of the system. We’ll keep the functionality as simple as possible. The monitoring service will simply render an error message to standard output. The file exchange between the main application container and the sidecar container happens through a Volume.

<img src=".\images\pod-sidecar-pattern.png"/>

The YAML manifest shown below sets up the described scenario. The most tricky portion of the code is the lengthy bash command. The command runs an infinite loop. As part of each iteration, we inspect the contents of the file error.log, grep for an error and potentially act on it. The loop executes every 10 seconds.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: logs-vol
      mountPath: /var/log/nginx
  - name: sidecar
    image: busybox
    command: ["sh","-c","while true; do if [ \"$(cat /var/log/nginx/error.log | grep 'error')\" != \"\" ]; then echo 'Error discovered!'; fi; sleep 10; done"]
    volumeMounts:
    - name: logs-vol
      mountPath: /var/log/nginx
  volumes:
  - name: logs-vol
    emptyDir: {}
```

When starting up the Pod, you’ll notice that the overall number of containers will show 2. After all containers can be started, the Pod signals a Running status:

```shell
$ kubectl create -f sidecar.yaml

pod/webserver created

$ kubectl get pods webserver

NAME        READY   STATUS              RESTARTS   AGE
webserver   0/2     ContainerCreating   0          4s

$ kubectl get pods webserver

NAME        READY   STATUS    RESTARTS   AGE
webserver   2/2     Running   0          5s
```

You will find that error.log does not contain any failure to begin with. It starts out as an empty file. With the following commands, you’ll provoke an error on purpose. After waiting for at least 10 seconds, you’ll find the expected message on the terminal, which you can query for with the logs command:

```shell
$ kubectl logs webserver -c sidecar

$ kubectl exec webserver -it -c sidecar -- /bin/sh

# wget -O- localhost?unknown

Connecting to localhost (127.0.0.1:80)
wget: server returned error: HTTP/1.1 404 Not Found

# cat /var/log/nginx/error.log

2020/07/18 17:26:46 [error] 29#29: *2 open() "/usr/share/nginx/html/unknown" failed (2: No such file or directory), client: 127.0.0.1, server: localhost, request: "GET /unknown HTTP/1.1", host: "localhost"

# exit

$ kubectl logs webserver -c sidecar

Error discovered!
```

### The Adapter Pattern

The basic purpose of an adapter container is to modify data, either on ingress or egress, to match some other need. Perhaps, an existing enterprise-wide monitoring tools has particular data format needs. An adapter would be an efficient way to standardize the output of the main container to be ingested by the monitoring tool, without having to modify the monitor or the containerized application. An adapter container transforms multiple applications to singular view.

This type of secondary container is useful to modify the data generated by the primary container. For example, the Microsoft version of ASCII is distinct from everyone else. You may need to modify a datastream for proper use.

As application developers, we want to focus on implementing business logic. For example, as part of a two-week sprint, say we’re tasked with adding a shopping cart feature. **In addition to the functional requirements, we also have to think about operational aspects like exposing administrative endpoints or crafting meaningful and properly formatted log output. It’s easy to fall into the habit of simply rolling all aspects into the application code, making it more complex and harder to maintain. Cross-cutting concerns in particular need to be replicated across multiple applications and are often copied and pasted from one code base to another.**

In Kubernetes, we can avoid bundling **cross-cutting concerns** into the application code by running them in another container apart from the main application container. The adapter pattern transforms the output produced by the application to make it consumable in the format needed by another part of the system. Figure 4-4 illustrates a concrete example of the adapter pattern.

<img src=".\images\pod-adapter-pattern.png"/>

The business application running the main container produces timestamped information—in this case, the available disk space—and writes it to the file diskspace.txt. As part of the architecture, we want to consume the file from a third-party monitoring application. The problem is that the external application requires the information to exclude the timestamp. Now, we could change the logging format to avoid writing the timestamp, but what do we do if we actually want to know when the log entry has been written? This is where the adapter pattern can help. An adapter container executes transformation logic that turns the log entries into the format needed by the external system without having to change application logic.

The YAML manifest shown below illustrates what this implementation of the adapter pattern could look like. The app container produces a new log entry every five seconds. The transformer container consumes the contents of the file, removes the timestamp, and writes it to a new file. Both containers have access to the same mount path through a Volume.

An exemplary adapter pattern implementation:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter
spec:
  containers:
  - args:
    - /bin/sh
    - -c
    - 'while true; do echo "$(date) | $(du -sh ~)" >> /var/logs/diskspace.txt; sleep 5; done;'
    image: busybox
    name: app
    volumeMounts:
      - name: config-volume
        mountPath: /var/logs
  - image: busybox
    name: transformer
    args:
    - /bin/sh
    - -c
    - 'sleep 20; while true; do while read LINE; do echo "$LINE" | cut -f2 -d"|" >> $(date +%Y-%m-%d-%H-%M-%S)-transformed.txt; done < /var/logs/diskspace.txt; sleep 20; done;'
    volumeMounts:
    - name: config-volume
      mountPath: /var/logs
  volumes:
  - name: config-volume
    emptyDir: {}
```

After creating the Pod, we’ll find two running containers. We should be able to locate the original file, '/var/logs/diskspace.txt', after shelling into the transformer container. The transformed data exists in a separate file in the user home directory:

```shell
$ kubectl create -f adapter.yaml

pod/adapter created

$ kubectl get pods adapter

NAME      READY   STATUS    RESTARTS   AGE
adapter   2/2     Running   0          10s

$ kubectl exec adapter --container=transformer -it -- /bin/sh

# cat /var/logs/diskspace.txt

Sun Jul 19 20:28:07 UTC 2020 | 4.0K	/root
Sun Jul 19 20:28:12 UTC 2020 | 4.0K	/root

# ls -l

total 40
-rw-r--r--  1  root  root  60 Jul 19 20:28 2020-07-19-20-28-28-transformed.txt

...

# cat 2020-07-19-20-28-28-transformed.txt

 4.0K	/root
 4.0K	/root
```

### The Ambassador Pattern

The ambassador pattern provides a proxy for communicating with external services.

There are many use cases that can justify the introduction of the ambassador pattern. **The overarching goal is to hide and/or abstract the complexity of interacting with other parts of the system.** Typical responsibilities include retry logic upon a request failure, security concerns like providing authentication or authorization, or monitoring latency or resource usage.

<img src=".\images\pod-ambassador-pattern.png"/>

In this example, we’ll want to implement rate-limiting functionality for HTTP(S) calls to an external service. For example, the requirements for the rate limiter could say that an application can only make a maximum of 5 calls every 15 minutes. Instead of strongly coupling the rate-limiting logic to the application code, it will be provided by an ambassador container. Any calls made from the business application need to be tunneled through the ambassador container. Example below shows a Node.js-based rate limiter implementation that makes calls to the external service Postman.

Node.js HTTP rate limiter implementation:

```javascript
const express = require('express');
const app = express();
const rateLimit = require('express-rate-limit');
const https = require('https');

const rateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message:
    'Too many requests have been made from this IP, please try again after an hour'
});

app.get('/test', rateLimiter, function (req, res) {
  console.log('Received request...');
  var id = req.query.id;
  var url = 'https://postman-echo.com/get?test=' + id;
  console.log("Calling URL %s", url);

  https.get(url, (resp) => {
    let data = '';

    resp.on('data', (chunk) => {
      data += chunk;
    });

    resp.on('end', () => {
      res.send(data);
    });

    }).on("error", (err) => {
      res.send(err.message);
    });
})

var server = app.listen(8081, function () {
  var port = server.address().port
  console.log("Ambassador listening on port %s...", port)
})
```

The corresponding Pod runs the main application container on a different port than the ambassador container. Every call to the HTTP endpoint of the container named business-app would delegate to the HTTP endpoint of the container named ambassador. It’s important to mention that containers running inside of the same Pod can communicate via localhost. No additional networking configuration is required.

An exemplary ambassador pattern implementation:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rate-limiter
spec:
  containers:
  - name: business-app
    image: bmuschko/nodejs-business-app:1.0.0
    ports:
    - containerPort: 8080
  - name: ambassador
    image: bmuschko/nodejs-ambassador:1.0.0
    ports:
    - containerPort: 8081
```

Let’s test the functionality. First, we’ll create the Pod, shell into the container that runs the business application, and execute a series of curl commands. The first five calls will be allowed to the external service. On the sixth call, we’ll receive an error message, as the rate limit has been reached within the given time frame:

```shell
$ kubectl create -f ambassador.yaml

pod/rate-limiter created

$ kubectl get pods rate-limiter

NAME           READY   STATUS    RESTARTS   AGE
rate-limiter   2/2     Running   0          5s

$ kubectl exec rate-limiter -it -c business-app -- /bin/sh

# curl localhost:8080/test

{"args":{"test":"123"},"headers":{"x-forwarded-proto":"https", \
"x-forwarded-port":"443","host":"postman-echo.com", \
"x-amzn-trace-id":"Root=1-5f177dba-e736991e882d12fcffd23f34"}, \
"url":"https://postman-echo.com/get?test=123"}
...

# curl localhost:8080/test

Too many requests have been made from this IP, please try again after an hour.
```

Ambassador is, an open source, Kubernetes-native API gateway for microservices built on Envoy.

It allows for access to the outside world without having to implement a service or another entry in an ingress controller: proxy local connection, reverse proxy, limits HTTP requests, re-route from the main container to the outside world.

This type of secondary container would be used to communicate with outside resources, often outside the cluster. Using a proxy, like Envoy or other, you can embed a proxy instead of using one provided by the cluster. It is helpful if you are unsure of the cluster configuration.
