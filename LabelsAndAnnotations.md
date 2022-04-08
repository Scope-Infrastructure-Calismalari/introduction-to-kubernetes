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

# Labels and Annotations

Part of the metadata of an object is a label. Though labels are not API objects, they are an important tool for cluster administration. They can be used to select an object based on an arbitrary string, regardless of the object type. Labels are immutable as of API version **apps/v1**.

Labels are an essential tool for querying, filtering, and sorting Kubernetes objects. 
Annotations only represent descriptive metadata for Kubernetes objects but have no ability to be used for queries. 

## Understanding Labels

Kubernetes lets you assign key-value pairs to objects so that you can use them later within a search query. Those key-value pairs are called labels. 

A label describes a Kubernetes object in distinct terms (e.g., a category like “frontend” or “backend”) but is not meant for elaborate, multi-word descriptions of its functionality. As part of the specification, Kubernetes limits the length of a label to a maximum of 63 characters and a range of allowed alphanumeric and separator characters.

Pod with labels:

<img src=".\images\pod-with-labels.png"/>

It’s common practice to assign one or many labels to an object at creation time; however, you can modify them as needed for a live object. When confronted with labels for the first time, they might seem like an insignificant feature—but their importance cannot be overstated. They’re essential for understanding the runtime behavior of more advanced Kubernetes objects like a Deployment and a Service.

## Organizing Pods with Labels

With microservices architectures, the number of deployed microservices can easily reach high values. Those components will probably be replicated (multiple copies of the same component will be deployed) and multiple versions or releases (stable, beta, canary, and so on) will run concurrently. This can lead to hundreds of pods in the system. Without a mechanism for organizing them, you end up with a big, incomprehensible mess.

<img src=".\images\p3_uncategorized_pods_example.jpg"/>

Organizing pods and all other Kubernetes objects is done through labels.

<img src=".\images\p3_categorized_pods_example.jpg"/>

Labels go hand in hand with label selectors. Label selectors allow you to select a subset of pods tagged with certain labels and perform an operation on those pods. A label selector is a criterion, which filters resources based on whether they include a certain label with a certain value.

## Declaring Labels

Labels can be declared imperatively with the run command or declaratively in the metadata.labels section in the YAML manifest. The command-line option --labels (or -l in its short form) defines a comma-separated list of labels when creating a Pod. 

The following command creates a new Pod with two labels from the command line:

```shell
$ kubectl run labeled-pod --image=nginx --restart=Never --labels=tier=backend,env=dev

pod/labeled-pod created
```

Assigning labels to Kubernetes objects by editing the manifest requires a change to the metadata section. Example below shows the same Pod definition from the previous command if we were to start with the YAML manifest.

A Pod defining two labels:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    env: dev
    tier: backend
spec:
  containers:
  - image: nginx
    name: nginx
```

## Inspecting Labels

Every resource can contain labels in its metadata. You can inspect the labels assigned to a Kubernetes object from different angles. Here, we’ll want to look at the most common ways to identify the labels of a Pod. As with any other runtime information, you can use the describe or get commands to retrieve the labels:

```shell
$ kubectl describe pod labeled-pod | grep -C 2 Labels:

...
Labels:       env=dev
              tier=backend
...

$ kubectl get pod labeled-pod -o yaml | grep -C 1 labels:

metadata:
  labels:
    env: dev
    tier: backend
...
```

If you want to list the labels for all object types or a specific object type, use the **--show-labels** command-line option. This option is convenient if you need to sift through a longer list of objects. The output automatically adds a new column named LABELS:

```shell
$ kubectl get pods --show-labels

NAME          READY   STATUS    RESTARTS   AGE   LABELS
labeled-pod   1/1     Running   0          38m   env=dev,tier=backend
```

## Modifying Labels for a Live Object

At any given point in time, you can add or remove a label from an existing Kubernetes object, or simply modify an existing label. One way to achieve this is by editing the live object and changing the label definition in the metadata.labels section. The other option that offers a slightly faster turnaround is the label command. The following commands add a new label, change the value of the label, and then remove the label with the minus character:

```shell
$ kubectl label pod labeled-pod region=eu

pod/labeled-pod labeled

$ kubectl get pod labeled-pod --show-labels

NAME          READY   STATUS    RESTARTS   AGE   LABELS
labeled-pod   1/1     Running   0          22h   env=dev,region=eu,tier=backend

$ kubectl label pod labeled-pod region=us --overwrite

pod/labeled-pod labeled

$ kubectl get pod labeled-pod --show-labels

NAME          READY   STATUS    RESTARTS   AGE   LABELS
labeled-pod   1/1     Running   0          22h   env=dev,region=us,tier=backend

$ kubectl label pod labeled-pod region-

pod/labeled-pod labeled

$ kubectl get pod labeled-pod --show-labels

NAME          READY   STATUS    RESTARTS   AGE   LABELS
labeled-pod   1/1     Running   0          22h   env=dev,tier=backend

```

## Using Label Selectors

Labels allow for objects to be selected, which may not share other characteristics. For example, if a developer were to label their pods using their name, they could affect all of their pods, regardless of the application or deployment the pods were using.

Labels are how operators, also known as watch-loops, track and manage objects. As a result, if you were to hard-code the same label for two objects, they may be managed by different operators and cause issues. For example, one deployment may call for ten pods, while another with the same label calls for five. All the pods would be in a constant state of restarting, as each operator tries to start or stop pods until the status matches the spec.

Consider the possible ways you may want to group your pods and other objects in production. Perhaps you may use development and production labels to differentiate the state of the application. Perhaps you may want to add labels for the department, team, and primary application developer.

Selectors are namespace-scoped. Use the --all-namespaces argument to select matching objects in all namespaces.

Labels only really become meaningful when combined with the selection feature. A label selector uses a set of criteria to query for Kubernetes objects. 

For example, you could use a label selector to express “select all Pods with the label assignment env=dev, tier=frontend, and have a label with the key version independent of the assigned value,”.

Selecting Pods by label criteria:

<img src=".\images\selecting-pods-label-criteria.png"/>

Kubernetes offers two ways to select objects by labels: from the command line and within a manifest.

### Label Selection from the Command Line

On the command line, you can select objects by label using the **--selector** option, or **-l** in its short-form notation. Objects can be filtered by an equality-based requirement or a set-based requirement. Both requirement types can be combined in a single query.

**An equality-based requirement can use the operators =, ==, or !=. You can separate multiple filter terms with a comma and then combine them with a boolean AND.** At this time, equality-based label selection cannot express a boolean OR operation. A typical expression could say, “select all Pods with the label assignment env=prod.”

**A set-based requirement can filter objects based on a set of values using the operators in, notin, and exists.** The in and notin operators work based on a boolean OR. A typical expression could say, “select all Pods with the label key env and the value prod or dev.”

To demonstrate the functionality, we’ll start by setting up three different Pods with labels. All kubectl commands use the command-line option --show-labels to compare the results with our expectations. The --show-labels option is not needed for label selection, though:

```shell
$ kubectl run frontend --image=nginx --restart=Never --labels=env=prod,team=shiny

pod/frontend created

$ kubectl run backend --image=nginx --restart=Never --labels=env=prod,team=legacy,app=v1.2.4

pod/backend created

$ kubectl run database --image=nginx --restart=Never --labels=env=prod,team=storage

pod/database created

$ kubectl get pods --show-labels

NAME       READY   STATUS    RESTARTS   AGE   LABELS
backend    1/1     Running   0          37s   app=v1.2.4,env=prod,team=legacy
database   1/1     Running   0          32s   env=prod,team=storage
frontend   1/1     Running   0          42s   env=prod,team=shiny
```

We’ll start by filtering the Pods with an equality-based requirement. The use of -l or --selector options can be used with kubectl. Here, we are looking for all Pods with the label assignment env=prod. The result returns all three Pods:

```shell
$ kubectl get pods --selector env=prod --show-labels

NAME       READY   STATUS    RESTARTS   AGE   LABELS
backend    1/1     Running   0          37s   app=v1.2.4,env=prod,team=legacy
database   1/1     Running   0          32s   env=prod,team=storage
frontend   1/1     Running   0          42s   env=prod,team=shiny
```

The next filter operation uses a set-based requirement. We are asking for all Pods that have the label key team with the values storage or shiny. The result only returns the Pods named backend and frontend:

```shell
$ kubectl get pods -l 'team in (shiny, legacy)' --show-labels

NAME       READY   STATUS    RESTARTS   AGE   LABELS
backend    1/1     Running   0          19m   app=v1.2.4,env=prod,team=legacy
frontend   1/1     Running   0          20m   env=prod,team=shiny
```

Finally, we’ll combine an equality-based requirement with a set-based requirement. The result returns only the backend Pod:

```shell
$ kubectl get pods -l 'team in (shiny, legacy)',app=v1.2.4 --show-labels

NAME      READY   STATUS    RESTARTS   AGE   LABELS
backend   1/1     Running   0          29m   app=v1.2.4,env=prod,team=legacy
```

### Label Selection in a Manifest

Some advanced Kubernetes objects such as Deployments, Services, or network policies act as configuration proxies for Pods. They usually select a set of Pods by labels and then provide added value. 

For example, a network policy controls network traffic from and to a set of Pods. Only the Pods with matching labels will apply the network rules. The following YAML manifest applies the network policy to Pods with the equality-based requirement tier=frontend.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network-policy
spec:
  podSelector:
    matchLabels:
      tier: frontend
...
```

There are several built-in object labels. For example nodes have labels such as the arch, hostname, and os, which could be used for assigning pods to a particular node, or type of node.

```shell
$ kubectl get node worker

....
   creationTimestamp: "2020-05-12T14:23:04Z"
   labels:
     beta.kubernetes.io/arch: amd64
     beta.kubernetes.io/os: linux
     kubernetes.io/arch: amd64
     kubernetes.io/hostname: worker
     kubernetes.io/os: linux
   managedFields:
....
```

The nodeSelector: entry in the podspec could use this label to cause a pod to be deployed on a particular node with an entry such as:

```yaml
     spec:
       nodeSelector:
         kubernetes.io/hostname: worker
       containers:
```

The way you define label selection in a manifest is based on the API version of the Kubernetes resources and may differ between different types.

## Using labels and selectors to constrain pod scheduling

Labels and label selectors can be used to constrain pod scheduling. As an example, let one of the nodes in your cluster contains a GPU to be used for general-purpose GPU computing. You add the label gpu=true to the nodes showing this feature.

Now imagine you want to deploy a new pod that needs a GPU to perform its work. To ask the scheduler to only choose among the nodes that provide a GPU, you’ll add a node selector to the pod’s YAML.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - name: kubia
    image: luksa/kubia
```

nodeSelector tells Kubernetes to deploy this pod only to nodes containing the gpu=true label. 

## Understanding Annotations

Labels are used to work with objects or collections of objects; annotations are not.

Instead, annotations allow for metadata to be included with an object that may be helpful outside of the Kubernetes object interaction. Similar to labels, they are key to value maps. They are also able to hold more information, and more human-readable information than labels.
 
Having this kind of metadata can be used to track information such as a timestamp, pointers to related objects from other ecosystems, or even an email from the developer responsible for that object's creation. 

The annotation data could otherwise be held in an exterior database, but that would limit the flexibility of the data. The more this metadata is included, the easier it is to integrate management and deployment tools or shared client libraries. 

Annotations are declared similarly to labels, but they serve a different purpose. They represent key-value pairs for providing descriptive metadata. The most important differentiator is that annotations cannot be used for querying or selecting objects. Typical examples of annotations may include SCM commit hash IDs, release information, or contact details for teams operating the object. Make sure to put the value of an annotation into single- or double-quotes if it contains special characters or spaces. Figure below illustrates a Pod with three annotations.

Pod with annotations:

<img src=".\images\pod-with-annotations.png"/>

## Declaring Annotations

The kubectl run command does not provide a command-line option for defining annotations that’s similar to the one for labels. You will have to start by writing a YAML manifest and adding the desired annotations under metadata.annotation.

A Pod defining three annotations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotated-pod
  annotations:
    commit: 866a8dc
    author: 'Benjamin Muschko'
    branch: 'bm/bugfix'
spec:
  containers:
  - image: nginx
    name: nginx
```

## Inspecting Annotations

Similar to labels, you can use the describe or get commands to retrieve the assigned annotations:


```shell
$ kubectl describe pod annotated-pod | grep -C 2 Annotations:

...
Annotations:  author: Benjamin Muschko
              branch: bm/bugfix
              commit: 866a8dc
...

$ kubectl get pod annotated-pod -o yaml | grep -C 3 annotations:

metadata:
  annotations:
    author: Benjamin Muschko
    branch: bm/bugfix
    commit: 866a8dc
...
```

## Modifying Annotations for a Live Object

The annotate command is the counterpart of the labels command but for annotations. As you can see in the following examples, the usage pattern is the same:

```shell
$ kubectl annotate pod annotated-pod oncall='800-555-1212'

pod/annotated-pod annotated

$ kubectl annotate pod annotated-pod oncall='800-555-2000' --overwrite

pod/annotated-pod annotated

$ kubectl annotate pod annotated-pod oncall-

pod/annotated-pod annotated
```

For example, to annotate only Pods within a namespace, you can overwrite the annotation, and finally delete it: 

```shell
$ kubectl annotate pods --all description='Production Pods' -n prod 

$ kubectl annotate --overwrite pod webpod description="Old Production Pods" -n prod 

$ kubectl -n prod annotate pod webpod description-
```

