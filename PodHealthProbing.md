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

# Pod Health Probing

Even with the best automated test coverage, it’s nearly impossible to find all bugs before deploying software to a production environment. That’s especially true for failure situations that only occur after operating the software for an extended period of time. It’s not uncommon to see memory leaks, deadlocks, infinite loops, and similar conditions crop up once the application has been put under load by end users.

Proper monitoring can help with identifying those issues; however, you still need to take an action to mitigate the situation. First of all, you’ll likely want to restart the application to prevent further outages. Second, the development team needs to identify the underlying root cause and fix the application’s code.

Kubernetes provides a concept called health probing to automate the detection and correction of such issues. You can configure a container to execute a periodic mini-process that checks for certain conditions. 

Each probe offers three distinct methods to verify the health of a container. You can define one or many of the health verification methods for a container. 

The available health verification methods, their corresponding YAML attribute, and their runtime behavior given the table below:

| Method | Option | Description |
| :--: | :--: | :--: |
| Custom command | exec.command | Executes a command inside of the container (e.g., a cat command) and checks its exit code. Kubernetes considers a zero exit code to be successful. A non-zero exit code indicates an error. |
| HTTP GET request | httpGet | Sends an HTTP GET request to an endpoint exposed by the application. An HTTP response code in the range of 200 and 399 indicates success. Any other response code is regarded as an error. |
| TCP socket connection | tcpSocket | Tries to open a TCP socket connection to a port. If the connection could be established, the probing attempt was successful. The inability to connect is accounted for as an error.|

Every probe offers a set of attributes that can further configure the runtime behavior.

| Attribute | Default value | Description |
| :--: | :--: | :--: |
| initialDelaySeconds | 0 | Delay in seconds until first check is executed.|
| periodSeconds | 10 | Interval for executing a check (e.g., every 20 seconds).|
| timeoutSeconds | 1 | Maximum number of seconds until check operation times out.|
| successThreshold | 1 | Number of successful check attempts until probe is considered successful after a failure.|
| failureThreshold | 3 | Number of failures for check attempts before probe is marked failed and takes action. |s

## Readiness Probe: readinessProbe

Even after an application has been started up, it may still need to execute configuration procedures—for example, connecting to a database and preparing data. This probe checks if the application is ready to serve incoming requests.

A readiness probe checks if the application is ready to accept traffic:

<img src=".\images\pod-readiness-probe.png"/>

Oftentimes, our application may have to initialize or be configured prior to being ready to accept traffic. As we scale up our application, we may have containers in various states of creation. Rather than communicate with a client prior to being fully ready, we can use a readinessProbe. The container will not accept traffic until the probe returns a healthy state.

With the exec statement, the container is not considered ready until a command returns a zero exit code. As long as the return is non-zero, the container is considered not ready and the probe will keep trying.

Another type of probe uses an HTTP GET request (httpGet). Using a defined header to a particular port and path, the container is not considered healthy until the web server returns a code 200-399. Any other code indicates failure, and the probe will try again.

The TCP Socket probe (tcpSocket) will attempt to open a socket on a predetermined port, and keep trying based on periodSeconds. Once the port can be opened, the container is considered healthy.

## Readiness Probe Scenario

In this scenario, we’ll want to define a readiness probe for a Node.js application. The Node.js application exposes an HTTP endpoint on the root context path and runs on port 3000. Dealing with a web-based application makes an HTTP GET request a perfect fit for probing its readiness.

In the YAML manifest shown below, the readiness probe executes its first check after two seconds and repeats checking every eight seconds thereafter. All other attributes use the default values. A readiness probe will continue to periodically check, even after the application has been successfully started.

A readiness probe that uses an HTTP GET request:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
spec:
  containers:
  - image: bmuschko/nodejs-hello-world:1.0.0
    name: hello-world
    ports:
    - name: nodejs-port
      containerPort: 3000
    readinessProbe:
      httpGet:
        path: /
        port: nodejs-port
      initialDelaySeconds: 2
      periodSeconds: 8
```

Create a Pod by pointing the create command to the YAML manifest. During the Pod’s startup process, it’s very possible that the status shows Running but the container isn’t ready to accept incoming requests yet, as indicated by 0/1 under the READY column:

```shell
$ kubectl create -f readiness-probe.yaml

pod/readiness-pod created

$ kubectl get pod readiness-pod

NAME                READY   STATUS    RESTARTS   AGE
pod/readiness-pod   0/1     Running   0          6s

$ kubectl get pod readiness-pod

NAME                READY   STATUS    RESTARTS   AGE
pod/readiness-pod   1/1     Running   0          68s

$ kubectl describe pod readiness-pod

...
Containers:
  hello-world:
    ...
    Readiness:      http-get http://:nodejs-port/ delay=2s timeout=1s \
                    period=8s #success=1 #failure=3
...
```

## Liveness Probe: livenessProbe

Just as we want to wait for a container to be ready for traffic, we also want to make sure it stays in a healthy state. Some applications may not have built-in checking, so we can use livenessProbes to continually check the health of a container. If the container is found to fail a probe, it is terminated. If under a controller, a replacement would be spawned.

Once the application is running, we’ll want to make sure that it still works as expected without issues. This probe periodically checks for the application’s responsiveness. Kubernetes restarts the Pod automatically if the probe considers the application be in an unhealthy state.

A liveness probe checks if the application is still considered healthy:

<img src=".\images\pod-liveness-probe.png"/>

## Liveness Probe Scenario

A liveness probe checks if the application is still working as expected down the road. For the purpose of demonstrating a liveness probe, we’ll use a custom command. A custom command is probably the most flexible way to verify the health of a container, as it allows for calling any command available to the container. That can either be a command-line tool that comes with the base image or a tool that you install as part of the containization process.

We’ll have the application create and update a file, '/tmp/heartbeat.txt', to show that it’s still alive. We’ll do this by it run the Unix touch command every five seconds. The probe will periodically check if the modification timestamp of the file is older than one minute. If it is, then Kubernetes can assume that the application isn’t functioning as expected and will restart the container.

A liveness probe that uses a custom command:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod
spec:
  containers:
  - image: busybox
    name: app
    args:
    - /bin/sh
    - -c
    - 'while true; do touch /tmp/heartbeat.txt; sleep 5; done;'
    livenessProbe:
      exec:
        command:
        - test `find /tmp/heartbeat.txt -mmin -1`
      initialDelaySeconds: 5
      periodSeconds: 30
```

The following command uses the YAML manifest stored in the file liveness-probe.yaml to create the Pod. Describing the Pod renders information on the liveness probe. Not only can we inspect the custom command and its configuration, we can also see how many times the container has been restarted upon a probing failure:

```shell
$ kubectl create -f liveness-probe.yaml

pod/liveness-pod created

$ kubectl get pod liveness-pod

NAME               READY   STATUS    RESTARTS   AGE
pod/liveness-pod   1/1     Running   0          22s

$ kubectl describe pod liveness-pod

...
Containers:
  app:
    ...
    Restart Count:  0
    Liveness:       exec [test `find /tmp/heartbeat.txt -mmin -1`] delay=5s \
                    timeout=1s period=30s #success=1 #failure=3
...
```

## Startup Probe: startupProbe

Becoming beta recently, we can also use the startupProbe. This probe is useful for testing an application which takes a long time to start.

Legacy applications in particular can take a long time to start up—we’re talking minutes sometimes. This probe can be instantiated to wait for a predefined amount of time before a liveness probe is allowed to start probing. By setting up a startup probe, you can prevent overwhelming the application process with probing requests. Startup probes kill the container if the application couldn’t start within the set time frame. 

<img src=".\images\pod-startup-probe.png"/>

If kubelet uses a startupProbe, it will disable liveness and readiness checks until the application passes the test. The duration until the container is considered failed is failureThreshold times periodSeconds. For example, if your periodSeconds was set to five seconds, and your failureThreshold was set to ten, kubelet would check the application every five seconds until it succeeds, or is considered failed after a total of 50 seconds. If you set the periodSeconds to 60 seconds, kubelet would keep testing for 300 seconds, or five minutes, before considering the container failed.

# Startup Probe Scenario

The purpose of a startup probe is to figure out when an application is fully started. Defining the probe is especially useful for an application that takes a long time to start up. 

The kubelet puts the readiness and liveness probes on hold while the startup probe is running. 

A startup probe finishes its operation under one of the following conditions:
- If it could verify that the application has been started.
- If the application doesn’t respond within the timeout period.

To demonstrate the functionality of the startup probe, Example below defines a Pod that runs the Apache HTTP server in a container. By default, the image exposes the container port 80, and that’s what we’re probing for using a TCP socket connection.

A startup probe that uses a TCP socket connection:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
spec:
  containers:
  - image: httpd:2.4.46
    name: http-server
    startupProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 15
```

As you can see in the following terminal output, the describe command can retrieve the configuration of a startup probe as well:

```shell
$ kubectl create -f startup-probe.yaml

pod/startup-pod created

$ kubectl get pod startup-pod

NAME              READY   STATUS    RESTARTS   AGE
pod/startup-pod   1/1     Running   0          31s

$ kubectl describe pod startup-pod

...
Containers:
  http-server:
     ...
     Startup:        tcp-socket :80 delay=3s timeout=1s period=15s \
                     #success=1 #failure=3
...
```
