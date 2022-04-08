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

# Kubernetes APIs and Access

## API Access

Kubernetes has a powerful REST-based API. The entire architecture is API-driven. Knowing where to find resource endpoints and understanding how the API changes between versions can be important to ongoing administrative tasks, as there is much ongoing change and growth. Starting with v1.16 deprecated objects are no longer honored by the API server.

The main agent for communication between cluster agents and from outside the cluster is the kube-apiserver. A curl query to the agent will expose the current API groups. Groups may have multiple versions, which evolve independently of other groups, and follow a domain-name format with several names reserved, such as single-word domains, the empty group, and any name ending in .k8s.io.

## Swagger and OpenAPI

The entire Kubernetes API was built using a Swagger specification. This has been evolving towards the OpenAPI initiative. It is extremely useful, as it allows, for example, to auto-generate client code. All the stable resources definitions are available on the documentation site. 

You can browse some of the API groups via a Swagger UI on the OpenAPI Specification web page.

https://swagger.io/specification/

## API Maturity

The use of API groups and different versions allows for development to advance without changes to an existing group of APIs. This allows for easier growth and separation of work among separate teams. While there is an attempt to maintain some consistency between API and software versions, they are only indirectly linked. 

The use of JSON and Google's Protobuf serialization scheme will follow the same release guidelines.

**Alpha**: An Alpha level release, noted with alpha in the names, may be buggy and is disabled by default. Features could change or disappear at any time, and backward compatibility is not guaranteed. Only use these features on a test cluster which is often rebuilt.

**Beta**: The Beta levels, found with beta in the names, has more well-tested code and is enabled by default. It also ensures that, as changes move forward, they will be tested for backwards compatibility between versions. It has not been adopted and tested enough to be called stable. You can expect some bugs and issues.

**Stable**: Use of the Stable version, denoted by only an integer which may be preceded by the letter v, is for stable APIs. At the moment, v1 is the only stable version.

## Optimistic Concurrency

The default serialization for API calls must be JSON. There is an effort to use Google's protobuf serialization, but this remains experimental. While we may work with files in a YAML format, they are converted to and from JSON.

Kubernetes uses the resourceVersion value to determine API updates and implement optimistic concurrency. In other words, an object is not locked from the time it has been read until the object is written.

Instead, upon an updated call to an object, the resourceVersion is checked, and a 409 CONFLICT is returned, should the number have changed. The resourceVersion is currently backed via the modifiedIndex parameter in the etcd database, and is unique to the namespace, kind, and server. Operations which do not change an object, such as WATCH or GET, do not update this value.

## RESTful

kubectl makes API calls on your behalf, responding to typical HTTP verbs (GET, POST, DELETE). You can also make calls externally, using curl or other program. With the appropriate certificates and keys, you can make requests, or pass JSON files to make configuration changes.

```shell
$ curl --cert userbob.pem --key userBob-key.pem \  
--cacert /path/to/ca.pem \   
https://k8sServer:6443/api/v1/pods 
```

The ability to impersonate other users or groups, subject to RBAC configuration, allows a manual override authentication. This can be helpful for debugging authorization policies of other users.

## /etc/kubernetes/admin.conf & ~/.kube/config

Take a look at the output below:

```yaml
apiVersion: v1
clusters:
- cluster: 
    certificate-authority-data: LS0tLS1CRUdF..... 
    server: https://10.128.0.3:6443 
    name: kubernetes
contexts:
- context: 
    cluster: kubernetes 
    user: kubernetes-admin 
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin 
  user: 
    client-certificate-data: LS0tLS1CRUdJTib..... 
    client-key-data: LS0tLS1CRUdJTi....
```

The output above shows 19 lines of output, with each of the keys being heavily truncated. While the keys may look similar, close examination shows them to be distinct.

**apiVersion**: As with other objects, this instructs the kube-apiserver where to assign the data.

**clusters**: This key contains the name of the cluster, as well as where to send the API calls. The certificate-authority-data is passed to authenticate the curl request.

**contexts**: This is a setting which allows easy access to multiple clusters, possibly as various users, from one configuration file. It can be used to set namespace, user, and cluster.

**current-context**: This shows which cluster and user the kubectl command would use. These settings can also be passed on a per-command basis.

**kind**: Every object within Kubernetes must have this setting; in this case, a declaration of object type Config.

**preferences**: Currently not used, this is an optional settings for the kubectl command, such as colorizing output.

**users**: A nickname associated with client credentials, which can be client key and certificate, username and password, and a token. Token and username/password are mutually exclusive. These can be configured via the kubectl config set-credentials command. 

## Using kubectl to Interact with the Kubernetes Cluster

Kubernetes objects can be created, updated, and deleted by using the **kubectl** command-line tool along with an object configuration file written in YAML or JSON.

**kubectl** is the primary tool to interact with the Kubernetes clusters from the command line. Therefore, it’s paramount to understand its ins and outs and practice its use heavily.

A kubectl execution consists of a command, a resource type, a resource name, and optional command line flags:

```shell
$ kubectl [command] [TYPE] [NAME] [flags]
```

<img src=".\images\kubectl-usage-pattern.png"/>

Kubectl acts as a CLI-based client to interact with the Kubernetes cluster. You can use its commands and flags to manage Kubernetes objects.

## API Resources with kubectl

All API resources exposed are available via kubectl. 
To get more information, do '**kubectl help**'.

```shell
$ kubectl api-resources
```

Expect the list below to change:

- all
- events (ev)
- podsecuritypolicies (psp)
- certificatesigningrequests (csr)
- horizontalpodautoscalers (hpa)
- podtemplates
- clusterrolebindings
- ingresses (ing)
- replicasets (rs)
- clusterroles
- jobs
- replicationcontrollers (rc)
- clusters (valid only for federation apiservers)
- limitranges (limits)
- resourcequotas (quota)
- componentstatuses (cs)
- namespaces (ns)
- rolebindings
- configmaps (cm)
- networkpolicies (netpol)
- roles
- controllerrevisions
- nodes (no)
- secrets
- cronjobs
- persistentvolumeclaims (pvc)
- serviceaccounts (sa)
- customresourcedefinition (crd)
- persistentvolumes (pv)
- services (svc)
- daemonsets (ds)
- poddisruptionbudgets (pdb)
- statefulsets
- deployments (deploy)
- podpreset
- storageclasses
- endpoints (ep)
- pods (po)

## Checking Access

While there is more detail on security in a later chapter, it is helpful to check the current authorizations, both as an administrator, as well as another user. The following shows what user bob could do in the default namespace and the developer namespace, using the auth can-i subcommand to query: 

```shell
$ kubectl auth can-i create deployments
yes 

$ kubectl auth can-i create deployments --as bob
no 

$ kubectl auth can-i create deployments --as bob --namespace developer
yes 
```

There are currently three APIs which can be applied to set who and what can be queried:
- SelfSubjectAccessReview​: Access review for any user, helpful for delegating to others.
- LocalSubjectAccessReview: ​Review is restricted to a specific namespace. 
- SelfSubjectRulesReview​: A review which shows allowed actions for a user within a particular namespace. 

The use of reconcile allows a check of authorization necessary to create an object from a file. No output indicates the creation would be allowed.

## Manage API Resources with kubectl

Kubernetes exposes resources via RESTful API calls, which allows all resources to be managed via HTTP, JSON or even XML, the typical protocol being HTTP. The state of the resources can be changed using standard HTTP verbs (e.g. GET, POST, PATCH, DELETE, etc.).

kubectl has a verbose mode argument which shows details from where the command gets and updates information. Other output includes curl commands you could use to obtain the same result. While the verbosity accepts levels from zero to any number, there is currently no verbosity value greater than ten. You can check this out for kubectl get. The output below has been formatted for clarity:

```shell
$ kubectl --v=10 get pods firstpod

....
I1215 17:46:47.860958 29909 round_trippers.go:417]
curl -k -v -XGET -H "Accept: application/json"
-H "User-Agent: kubectl/v1.8.5 (linux/amd64) kubernetes/cce11c6"
https://10.128.0.3:6443/api/v1/namespaces/default/pods/firstpod
....
```

If you delete this pod, you will see that the HTTP method changes from XGET to XDELETE.

```shell
$ kubectl --v=10 delete pods firstpod

....
I1215 17:49:32.166115 30452 round_trippers.go:417]
curl -k -v -XDELETE -H "Accept: application/json, */*"
-H "User-Agent: kubectl/v1.8.5 (linux/amd64) kubernetes/cce11c6"
https://10.128.0.3:6443/api/v1/namespaces/default/pods/firstpod
....
```

### Object Management

You can create objects in a Kubernetes cluster in two ways: imperatively or declaratively. The following sections will describe each approach, including their benefits, drawbacks, and use cases.

### Imperative Approach

The imperative method for object creation does not require a manifest definition. You would use the kubectl run or kubectl create command to create an object on the fly. Any configuration needed at runtime is provided by command-line options. The benefit of this approach is the fast turnaround time without the need to wrestle with YAML structures:

```shell
$ kubectl run frontend --image=nginx --restart=Never --port=80

pod/frontend created
```

### Declarative Approach

The declarative approach creates objects from a manifest file (in most cases, a YAML file) using the kubectl create or kubectl apply command. The benefit of using the declarative method is reproducibility and improved maintenance, as the file is checked into version control in most cases. The declarative approach is the recommended way to create objects in production environments:

```shell
$ vim pod.yaml

$ kubectl create -f pod.yaml

pod/frontend created
```

### Hybrid Approach

Sometimes, you may want to go with a hybrid approach. You can start by using the imperative method to produce a manifest file without actually creating an object. You do so by executing the kubectl run command with the command-line options **'-o yaml'** and **'--dry-run=client'**:

```shell
$ kubectl run frontend --image=nginx --restart=Never --port=80 -o yaml --dry-run=client > pod.yaml

$ vim pod.yaml

$ kubectl create -f pod.yaml

pod/frontend created

$ kubectl describe pod frontend

Name:         frontend
Namespace:    default
Priority:     0
...
```

### Deleting Kubernetes Objects

You might create Kubernetes objects with incorrect configuration, or you may simply want to start over from scratch. By default, Kubernetes tries to delete objects gracefully, which can can take up to 30 seconds. Use the command line option **--grace-period=0** and **--force** to send a SIGKILL signal. 

The signal will delete a Kubernetes object immediately:

```shell
$ kubectl delete pod nginx --grace-period=0 --force
```

In a work environment, you’ll want to delete objects that are not needed anymore. The delete command offers two options: deleting an object by providing the name or deleting an object by pointing to the YAML manifest that created it:

```shell
$ kubectl delete pod frontend

pod "frontend" deleted

$ kubectl delete -f pod.yaml

pod "frontend" deleted
```

### Editing a live object

Say you already created an object and you wanted to make further changes to the live object. You have the option to modify the object in your editor of choice from the terminal using the edit command. After saving the object definition in the editor, Kubernetes will try to reflect those changes in the live object:

```shell
$ kubectl edit pod frontend
```

### Replacing a live object

Sometimes, you’ll just want to replace the definition of an existing object declaratively. The replace command overwrites the live configuration with the one from the provided YAML manifest. The YAML manifest you feed into the command must be a complete resource definition as observed with the create command:

```shell
$ kubectl replace -f pod.yaml
```

### Updating a live object

Finally, I want to briefly explain the apply command and the main difference to the create command. The create command instantiates a new object. Trying to execute the create command for an existing object will produce an error. The apply command is meant to update an existing object in its entirety or just incrementally. That’s why the provided YAML manifest may be a full definition of an object or a partial definition (e.g., just the number of replicas for a Deployment). Please note that the apply command behaves like the create command if the object doesn’t exist yet, however, the YAML manifest will need to contain a full definition of the object:

```shell
$ kubectl apply -f pod.yaml

pod/frontend configured
```

## Access from Outside the Cluster

The primary tool used from the command line will be kubectl, which calls curl on your behalf. You can also use the curl command from outside the cluster to view or make changes. 

The basic server information, with redacted TLS certificate information, can be found in the output of 

```shell
$ kubectl config view 
```

If you view the verbose output from a previous page, you will note that the first line references a configuration file where this information is pulled from, ~/.kube/config:

```shell
I1215 17:35:46.725407 27695 loader.go:357] 
     Config loaded from file /home/student/.kube/config 
```

Without the certificate authority, key and certificate from this file, only insecure curl commands can be used, which will not expose much due to security settings. We will use curl to access our cluster using TLS in an upcoming lab.

## Additional Resource Methods

In addition to basic resource management via REST, the API also provides some extremely useful endpoints for certain resources. 

For example, you can access the logs of a container, exec into it, and watch changes to it with the following endpoints: 

```shell
$ curl --cert /tmp/client.pem --key /tmp/client-key.pem \ 
--cacert /tmp/ca.pem -v -XGET \ 
 https://10.128.0.3:6443/api/v1/namespaces/default/pods/firstpod/log
```

This would be the same as the following. If the container does not have any standard out, there would be no logs. 

```shell
$ kubectl logs firstpod
```

There are other calls you could make, following the various API groups on your cluster: 

```http
GET /api/v1/namespaces/{namespace}/pods/{name}/exec
GET /api/v1/namespaces/{namespace}/pods/{name}/log
GET /api/v1/watch/namespaces/{namespace}/pods/{name}
```
