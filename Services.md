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

# Services

You can communicate with a Pod by targeting its IP address. It’s important to recognize that Pods’ IP addresses are virtual and will therefore change to random values over time. A restart of a Pod will automatically assign a new virtual cluster IP address. Therefore, other parts of your system cannot rely on the Pod’s IP address if they need to talk to one another.

Pods need a way of finding other pods if they want to consume the services they provide. Configuring each client application by specifying the exact IP address or hostname of the server providing the service in the client’s configuration files will not work in Kubernetes because:

* Pods are ephemeral—They may come and go at any time.
* Kubernetes assigns an IP address to a pod after the pod has been scheduled to a node and before it’s started—Clients thus can’t know the IP address of the server pod up front.
* Horizontal scaling means multiple pods may provide the same service—Each of those pods has its own IP address. 

A Kubernetes Service is a resource created to make a single, constant point of entry to a group of pods providing the same service. 

The Kubernetes primitive Service implements an abstraction layer on top of Pods, assigning a fixed virtual IP fronting all the Pods with matching labels, and that virtual IP is called Cluster IP. This chapter will focus on the ins and outs of Services, and most importantly the exposure of Pods inside or outside of the cluster depending on their declared type.

By default, Kubernetes does not restrict inter-Pod communication in any shape or form. You can define a network policy to mitigate potential security risks. Network policies describe the access rules for incoming and outgoing network traffic to and from Pods. 

Each service has an IP address and port that never change while the service exists. Clients can open connections to that IP and port, and those connections are then routed to one of the pods backing that service. 

Pod-to-Pod communication should be through a Service. Services provide networking rules for a select set of Pods. Any traffic routed through the Service will be forwarded to Pods. You can assign one of the following types to a Service: ClusterIP, the default type; NodePort; LoadBalancer; or ExternalName. The selected type determines how the Pods are made available—for example, only from within the cluster or accessible from outside of the cluster. In practice, you’ll commonly see a Deployment and a Service working together, though they serve different purposes and can operate independently.

## Understanding Services

Services are one of the central concepts in Kubernetes. Without a Service, you won’t be able to expose your application to consumers in a stable and predictable fashion. In a nutshell, Services provide discoverable names and load balancing to Pod replicas. The services and Pods remain agnostic from IPs with the help of the Kubernetes DNS control plane component. Similar to a Deployment, the Service determines the Pods it works on with the help of label selection.

Clients of a service don’t know the location of individual pods providing the service, allowing those pods to be moved around the cluster at any time.

<img src=".\images\p3_KubernetesService.jpg"/>

A service can be backed by more than one pod. Connections to the service are load-balanced across all the backing pods. 

Label selectors determine which pods belong to the Service.

<img src=".\images\p3_service_labelselector.jpg"/>

With every object and agent decoupled we need a flexible and scalable operator which connects resources together and will reconnect, should something die and a replacement is spawned. Each Service is a microservice handling a particular bit of traffic, such as a single NodePort or a LoadBalancer to distribute inbound requests among many Pods.

A Service also handles access policies for inbound requests, useful for resource control, as well as for security.

A service, as well as kubectl, uses a selector in order to know which objects to connect. There are two selectors currently supported:

- **equality-based**: Filters by label keys and their values. Three operators can be used, such as =, ==, and !=. If multiple values or keys are used, all must be included for a match.
- **set-based**: Filters according to a set of values. The operators are in, notin, and exists. For example, the use of status notin (dev, test, maint) would select resources with the key of status which did not have a value of dev, test, nor maint.

## Service Types

Every Service needs to define a type. The type determines how the Service exposes the matching Pods, as listed in Table below.

| Type | Description |
| :--: | :--: |
| ClusterIP | Exposes the Service on a cluster-internal IP. Only reachable from within the cluster. |
| NodePort | Exposes the Service on each node’s IP address at a static port. Accessible from outside of the cluster. |
| LoadBalancer | Exposes the Service externally using a cloud provider’s load balancer. |
| ExternalName | Maps a Service to a DNS name. |

The most important types you will need to understand are ClusterIP and NodePort. Those types make Pods reachable from within the cluster and from outside of the cluster. 

## Creating Services

Service is created by posting a JSON or YAML descriptor to the Kubernetes API server.

We’ll look at creating a Service from both the imperative and declarative approach angles. In fact, there are two ways to create a Service imperatively.

The command create service instantiates a new Service. You have to provide the type of the Service as the third, mandatory command-line option. That’s also the case for the default type, ClusterIP. In addition, you can optionally provide the port mapping.

```shell
$ kubectl create service clusterip nginx-service --tcp=80:80

service/nginx-service created
```

Instead of creating a Service as a standalone object, you can also expose a Pod or Deployment with a single command. The run command provides an optional --expose command-line option, which creates a new Pod and a corresponding Service with the correct label selection in place:

```shell
$ kubectl run nginx --image=nginx --restart=Never --port=80 --expose

service/nginx created
pod/nginx created
```

For an existing Deployment, you can expose the underlying Pods with a Service using the expose deployment command:

```shell
$ kubectl expose deployment my-deploy --port=80 --target-port=80

service/my-deploy exposed
```

The expose command and the --expose command-line option are welcome shortcuts as a means to creating a new Service with a fast turnaround time.

Using the declarative approach, you would define a Service manifest in YAML form as shown in example below. As you can see, the key of the label selector uses the value app. After creating the Service, you will likely have to change the label selection criteria to meet your needs, as the create service command does not offer a dedicated command-line option for it.

A Service defined by a YAML manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP
  selector:
    app: nginx-service
  ports:
  - port: 80
    targetPort: 80
```
## Exposing multiple ports in the same service

Services can support multiple ports. For example, if your pods listened on two ports, 8080 for HTTP and 8443 for HTTPS. You could use a single service to forward both port 80 and 443 to the pod’s ports 8080 and 8443. You don’t need to create two different services in such cases. Using a single, multi-port service exposes all the service’s ports through a single cluster IP.

When creating a service with multiple ports, you must specify a name for each port.

<img src=".\images\p3_service_multipleport.jpg"/>

The label selector applies to the service as a whole—it can’t be configured for each port individually. If you want different ports to map to different subsets of pods, you need to create two services.

You can give a name to each pod’s port and refer to it by name in the service spec. The biggest benefit of doing so is that it enables you to change port numbers later without having to change the service spec. 

<img src=".\images\p3_service_namedports_pod.jpg"/>

<img src=".\images\p3_service_namedports_service.jpg"/>

## Listing Services

You can observe the most important attributes of a Service when rendering the full list for a namespace. The following command shows the type, the cluster IP, an optional external IP, and the mapped ports:

```shell
$ kubectl get services

NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.109.125.232   <none>        80/TCP    82m
```

## Rendering Service Details

The describe command helps with retrieving even more details about a Service. The label selector will be included in the description of the Service, represented by the attribute Selector. That’s important information when troubleshooting a Service object:

```shell
$ kubectl describe service nginx-service

Name:              nginx-service
Namespace:         default
Labels:            app=nginx-service
Annotations:       <none>
Selector:          app=nginx-service
Type:              ClusterIP
IP:                10.109.125.232
Port:              80-80  80/TCP
TargetPort:        80/TCP
Endpoints:         <none>
Session Affinity:  None
Events:            <none>
```

## Port Mapping

In “Creating Services”, we only briefly touched on the topic of port mapping. The correct port mapping determines if the incoming traffic actually reaches the application running inside of the Pods that match the label selection criteria of the Service. **A Service always defines two different ports: the incoming port accepting traffic and the outgoing port, also called the target port.** Their functionality is best illustrated by example.

Figure below shows a Service that accepts incoming traffic on port 3000. That’s the port defined by the attribute 'ports.port' in the manifest. Any incoming traffic is then routed toward the target port, represented by ports.targetPort. The target port is the same port as defined by the container running inside of the label-selected Pod. In this case, that’s port 80.

Port mapping of a Service to a Pod:

<img src=".\images\port-mapping-service-pod.png"/>

## Discovering Services

By creating a service, you now have a single and stable IP address and port that you can hit to access your pods. This address will remain unchanged throughout the whole lifetime of the service. Pods behind this service may come and go, their IPs may change, their number can go up or down, but they’ll always be accessible through the service’s single and constant IP address.

But how do the client pods know the IP and port of a service?

* Discover services through environment variables: When a pod is started, Kubernetes initializes a set of environment variables pointing to each service that exists at that moment. If you create the service before creating the client pods, processes in those pods can get the IP address and port of the service by inspecting their environment variables.

* Discovering services through DNS: The kube-system namespace includes a pod for DNS and a corresponding service with the name kube-dns. The pod runs a DNS server, which all other pods running in the cluster are automatically configured to use (Kubernetes does that by modifying each container’s /etc/resolv.conf file). Any DNS query performed by a process running in a pod will be handled by Kubernetes’ own DNS server, which knows all the services running in your system. Each service gets a DNS entry in the internal DNS server, and client pods that know the name of the service can access it through its fully qualified domain name (FQDN) instead of resorting to environment variables.

## Connecting to services living outside the cluster

Services don’t link to pods directly. Instead, a resource sits in between—the Endpoints resource. An Endpoints resource is a list of IP addresses and ports exposing a service. Pod selector defined in the service spec is not used directly when redirecting incoming connections. Instead, the selector is used to build a list of IPs and ports, which is then stored in the Endpoints resource. When a client connects to a service, the service proxy selects one of those IP and port pairs and redirects the incoming connection to the server listening at that location. 

## Exposing services to external clients

Services can be made accessible externally by:

* ClusterIP service type
* NodePort service type
* LoadBalancer service type
* Creating an Ingress resource

## Accessing a Service with Type ClusterIP

ClusterIP is the default type of Service. It exposes the Service on a cluster-internal IP address. Figure below shows how to reach a Pod exposed by the ClusterIP type from another Pod from within the cluster. You can also create a proxy from outside of the cluster using the kubectl proxy command. Using a proxy is not only meant for production environments but can also be helpful for troubleshooting a Service.

Accessibility of a Service with the type ClusterIP:

<img src=".\images\service-clusterip.png"/>

To demonstrate the use case, we’ll opt for a quick way to create the Pod and the corresponding Service with the same command. The command automatically takes care of properly mapping labels and ports:

```shell
$ kubectl run nginx --image=nginx --restart=Never --port=80 --expose

service/nginx created
pod/nginx created

$ kubectl get pod,service

NAME        READY   STATUS    RESTARTS   AGE
pod/nginx   1/1     Running   0          26s

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/nginx   ClusterIP   10.96.225.204   <none>        80/TCP    26s
```

Remember that the Service of type ClusterIP can only be reached from within the cluster. To demonstrate the behavior, we’ll create a new Pod running in the same cluster and execute a wget command to access the application. 

Have a look at the cluster IP exposed by the Service—that’s 10.96.225.204. The port is 80. Combined as a single command, you can resolve the application via wget -O- 10.96.225.204:80 from the temporary Pod:

```shell
$ kubectl run busybox --image=busybox --restart=Never -it -- /bin/sh

# wget -O- 10.96.225.204:80

Connecting to 10.96.225.204:80 (10.96.225.204:80)
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                    100% |****************************|   612  0:00:00 ETA
written to stdout
/ # exit
```

The proxy command can establish a direct connection to the Kubernetes API server from your localhost. With the following command, we are opening port 9999 on which to run the proxy:

```shell
$ kubectl proxy --port=9999

Starting to serve on 127.0.0.1:9999
```

After running the command, you will notice that the shell is going to wait until you break out of the operation. To try talking to the proxy, you will have to open another terminal window. Say you have the curl command-line tool installed on your machine to make a call to an endpoint of the API server. 

The following example uses localhost:9999—that’s the proxy entry point. As part of the URL, you’re providing the endpoint to the Service named nginx running in the default namespace according to the API reference:

```shell
$ curl -L localhost:9999/api/v1/namespaces/default/services/nginx/proxy

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and \
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Accessing a Service with Type NodePort

Declaring a Service with type NodePort exposes access through the node’s IP address and can be resolved from outside of the Kubernetes cluster. The node’s IP address can be reached in combination with a port number in the range of 30000 and 32767, assigned automatically upon the creation of the Service. Figure below illustrates the routing of traffic to Pods via a NodePort-typed Service.

Accessibility of a Service with the type NodePort:

<img src=".\images\service-nodeport.png"/>


By creating a NodePort service, you make Kubernetes reserve a port on all its nodes (the same port number is used across all of them) and forward incoming connections to the pods that are part of the service.

<img src=".\images\p3_service_nodeport_yaml.jpg"/>

NodePort service can be accessed not only through the service’s internal cluster IP, but also through any node’s IP and the reserved node port.

<img src=".\images\service_nodeport_access.jpg"/>

Let’s enhance the example from the previous section. We’ll change the existing Service named nginx to use the type NodePort instead of ClusterIP. There are various ways to implement the change. For this example, we’ll use the patch command. When listing the Service, you will find the changed type and the port you can use to reach the Service. The port that has been assigned in this example is 32300:

```shell
$ kubectl patch service nginx -p '{ "spec": {"type": "NodePort"} }'

service/nginx patched

$ kubectl get service nginx

NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.96.225.204   <none>        80:32300/TCP   3d21h
```

You should now be able to access the Service using the node IP address and the node port. One way to discover the IP address of the node is by first listing all available nodes and then inspecting the relevant ones for details. In the following commands, we are only running on a single-node Kubernetes cluster, which makes things easy:

```shell
$ kubectl get nodes

NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   91d   v1.18.3

$ kubectl describe node minikube | grep InternalIP:

  InternalIP:  192.168.64.2

$ curl 192.168.64.2:32300

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and \
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.co</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

## Exposing a service through an external load balancer

The service type is set to LoadBalancer. This type of service obtains a load balancer from the infrastructure hosting the Kubernetes cluster. The load balancer will have its own unique, publicly accessible IP address and will redirect all connections to your service. You can thus access your service through the load balancer’s IP address.

<img src=".\images\p3_service_loadbalancer_yaml.jpg"/>

<img src=".\images\p3_service_loadbalancer_access.jpg"/>

## Exposing services externally through an Ingress resource

Each LoadBalancer service requires its own load balancer with its own public IP address, whereas an Ingress only requires one, even when providing access to dozens of services.

<img src=".\images\p3_service_ingress.jpg"/>

To make Ingress resources work, an Ingress controller needs to be running in the cluster.

The client first performed a DNS lookup of kubia.example.com, and the DNS server (or the local operating system) returned the IP of the Ingress controller. The client then sent an HTTP request to the Ingress controller and specified kubia.example.com in the Host header. From that header, the controller determined which service the client is trying to access, looked up the pod IPs through the Endpoints object associated with the service, and forwarded the client’s request to one of the pods.

<img src=".\images\p3_service_ingress_howitworks.jpg"/>

You can map multiple paths on the same host to different services.

You can use an Ingress to map to different services based on the host in the HTTP request instead of (only) the path

## Pod Readiness Probe

Pods are included as endpoints of a service if their labels match the service’s pod selector. As soon as a new pod with proper labels is created, it becomes part of the service and requests start to be redirected to the pod. But what if the pod isn’t ready to start serving requests immediately?

The pod may need time to load either configuration or data, or it may need to perform a warm-up procedure to prevent the first user request from taking too long and affecting the user experience. In such cases you don’t want the pod to start receiving requests immediately, especially when the already-running instances can process requests properly and quickly. It makes sense to not forward requests to a pod that’s in the process of starting up until it’s fully ready.

Kubernetes allows you to also define a readiness probe for your pod. The readiness probe is invoked periodically and determines whether the specific pod should receive client requests or not. When a container’s readiness probe returns success, it’s signaling that the container is ready to accept requests. A pod whose readiness probe fails is removed as an endpoint of a service.

Three types of readiness probes exist:

* An Exec probe, where a process is executed. The container’s status is determined by the process’ exit status code.
* An HTTP GET probe, which sends an HTTP GET request to the container and the HTTP status code of the response determines whether the container is ready or not.
* A TCP Socket probe, which opens a TCP connection to a specified port of the container. If the connection is established, the container is considered ready.

## Headless Services

Services can be used to provide a stable IP address allowing clients to connect to pods backing the services. Each connection to the service is forwarded to one randomly selected backing pod. But what if the client needs to connect to all of those pods? What if the backing pods themselves need to each connect to all the other backing pods? Connecting through the service clearly isn’t the way to do this.

For a client to connect to all pods, it needs to figure out the IP of each individual pod. One option is to have the client call the Kubernetes API server and get the list of pods and their IP addresses through an API call, but because you should always strive to keep your apps Kubernetes-agnostic, using the API server isn’t ideal.

**When you perform a DNS lookup for a service, the DNS server returns a single IP—the service’s cluster IP. But if you tell Kubernetes you don’t need a cluster IP for your service (you do this by setting the clusterIP field to None in the service specification), the DNS server will return the pod IPs instead of the single service IP.**

## Troubleshooting services

When you’re unable to access your pods through the service, you should start by going through the following list:

* First, make sure you’re connecting to the service’s cluster IP from within the cluster, not from the outside.
* Don’t bother pinging the service IP to figure out if the service is accessible (remember, the service’s cluster IP is a virtual IP and pinging it will never work).
* If you’ve defined a readiness probe, make sure it’s succeeding; otherwise the pod won’t be part of the service.
* To confirm that a pod is part of the service, examine the corresponding Endpoints object with kubectl get endpoints.
* If you’re trying to access the service through its FQDN or a part of it (for example, myservice.mynamespace.svc.cluster.local or myservice.mynamespace) and it doesn’t work, see if you can access it using its cluster IP instead of the FQDN.
* Check whether you’re connecting to the port exposed by the service and not the target port.
* Try connecting to the pod IP directly to confirm your pod is accepting connections on the correct port.
* If you can’t even access your app through the pod’s IP, make sure your app isn’t only binding to localhost.

## Deployments and Services

A Deployment manages Pods and their replication. A Service routes network requests to a set of Pods. Both primitives use label selection to connect with an associated set of Pods.

Relationship between a Deployment and Service:

<img src=".\images\relationship-deployment-service.png"/>
