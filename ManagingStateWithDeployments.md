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

# Managing State with Deployments

The default controller for a container deployed via kubectl run is a Deployment. As with other objects, a deployment can be made from a YAML or JSON spec file. When added to the cluster, the controller will create a ReplicaSet and a Pod automatically. The containers, their settings and applications can be modified via an update, which generates a new ReplicaSet, which, in turn, generates new Pods.

The updated objects can be staged to replace previous objects as a block or as a rolling update, which is determined as part of the deployment specification. Most updates can be configured by editing a YAML file and running **kubectl apply**. You can also use **kubectl edit** to modify the in-use configuration. Previous versions of the ReplicaSets are kept, allowing a rollback to return to a previous configuration.

Labels are essential to administration in Kubernetes, but are not an API resource. They are user-defined key-value pairs which can be attached to any resource, and are stored in the metadata. Labels are used to query or select resources in your cluster, allowing for flexible and complex management of the cluster. 

As a label is arbitrary, you could select all resources used by developers, or belonging to a user, or any attached string, without having to figure out what kind or how many of such resources exist.

## Deployments

Deployments allow server-side updates to pods at a specified rate. They are used for canary and other deployment patterns. Deployments generate ReplicaSets, which offer more selection features than ReplicationControllers, such as matchExpressions. 

```shell
$ kubectl create deployment dev-web --image=nginx:1.13.7-alpine

deployment "dev-web" created
```

The basic outline of an application running on Kubernetes is as follows.

<img src=".\images\p3_deployments_general_architecture.jpg"/>

The pods are backed by a ReplicationController or a ReplicaSet. A Service exists through which apps running in other pods or external clients access the pods. 

Initially, the pods run the first version of your application—let’s suppose its image is tagged as v1. You then develop a newer version of the app and push it to an image repository as a new image, tagged as v2. You’d next like to replace all the pods with this new version. 

You have two ways of updating all those pods:

* If you have a ReplicationController managing a set of v1 pods,  replace them by modifying the pod template so it refers to version v2 of the image. Then delete the old pod instances. The ReplicationController will notice that no pods match its label selector and it will spin up new instances.  

<img src=".\images\p3_deployments_update_pods_v1.jpg"/>

* Blue-Green Deployment: Start new pods and once they’re up, delete the old ones. 

    You can do this either by adding all the new pods and then deleting all the old ones at once, or sequentially, by adding new pods and removing old ones gradually.


<img src=".\images\p3_deployments_blue_green_update_v1.jpg"/>

<img src=".\images\p3_deployments_rolling_update.jpg"/>

A Deployment is a higher-level resource meant for deploying applications and updating them declaratively, instead of doing it through a ReplicationController or a ReplicaSet, which are both considered lower-level concepts.

When you create a Deployment, a ReplicaSet resource is created underneath. Replica-Sets replicate and manage pods. When using a Deployment, the actual pods are created and managed by the Deployment’s ReplicaSets, not by the Deployment directly.

 Replication of Pods with a Deployment:

<img src=".\images\replication-pods-deployment.png"/>

## Object Relationship

Here you can see the relationship between objects from the container, which Kubernetes does not directly manage, up to the deployment.

<img src=".\images\KubernetesNestedObjects.png"/>

The boxes and shapes are logical, in that they represent the controllers, or watch loops, running as a thread of the kube-controller-manager. Each controller queries the kube-apiserver for the current state of the object they track. The state of each object on a worker node is sent back from the local kubelet.

The graphic in the upper left represents a container running nginx 1.11. Kubernetes does not directly manage the container. Instead, the kubelet daemon checks the pod specifications by asking the container engine, which could be Docker or cri-o, for the current status. The graphic to the right of the container shows a pod which represents a watch loop checking the container status. kubelet compares the current pod spec against what the container engine replies and will terminate and restart the pod if necessary.

A multi-container pod is shown next. While there are several names used, such as sidecar or ambassador, these are all multi-container pods. The names are used to indicate the particular reason to have a second container in the pod, instead of denoting a new kind of pod.
On the lower left we see a replicaSet. This controller will ensure you have a certain number of pods running. The pods are all deployed with the same podSpec, which is why they are called replicas. Should a pod terminate or a new pod be found, the replicaSet will create or terminate pods until the current number of running pods matches the specifications. Any of the current pods could be terminated should the spec demand fewer pods running. 

The graphic in the lower right shows a deployment. This controller allows us to manage the versions of images deployed in the pods. Should an edit be made to the deployment, a new replicaSet is created, which will deploy pods using the new podSpec. The deployment will then direct the old replicaSet to shut down pods as the new replicaSet pods become available. Once the old pods are all terminated, the deployment terminates the old replicaSet and the deployment returns to having only one replicaSet running.

## Creating a Deployment

Deployments can be created imperatively with the create deployment command. The options you can provide to configure the Deployment are somewhat limited and do not resemble the ones you know from the run command. The following command creates a new Deployment that uses the image nginx:1.14.2 for a single replica:

```shell
$ kubectl create deployment my-deploy --image=nginx:1.14.2

deployment.apps/my-deploy created
```

Often, you will find yourself generating and further modifying the YAML manifest. 

A Deployment is composed of a label selector, a desired replica count, a pod template and a deployment strategy that defines how an update should be performed when the Deployment resource is modified.

The selector spec.selector.matchLabels matches on the key-value pair app=nginx with the label defined under the template section.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

```shell
kubectl apply -f nginx-deployment.yaml
```

## Listing Deployments

Once created, a Deployment and all of its corresponding objects can be listed. The following get command lists all Deployments, Pods, and ReplicaSets. If a Pod or ReplicaSet is managed by a Deployment, the name of the object will reflect that connection.

To generate the YAML file of the newly created objects, do:

```shell
$ kubectl get deployments,rs,pods -o yaml
```

Sometimes, a JSON output can make it more clear:

```shell
$ kubectl get deployments,rs,pods -o json
```

## Deployment Details

Now we will look at the YAML output, which also shows default values not passed to the object when created:

```yaml
apiVersion: v1
items:
- apiVersion: apps/v1
  kind: Deployment
```

**apiVersion**: A value of v1 indicates this object is considered to be a stable resource. In this case, it is not the deployment. It is a reference to the List type. 

**items**: As the previous line is a List, this declares the list of items the command is showing. 

**- apiVersion**: The dash is a YAML indication of the first item of the list, which declares the apiVersion of the object as apps/v1. This indicates the object is considered stable. Deployments are an operator used in many cases. 

**kind**: This is where the type of object to create is declared, in this case, a deployment.

## Deployment Configuration Metadata

Continuing with the YAML output, we see the next general block of output concerns the metadata of the deployment. This is where we would find labels, annotations, and other non-configuration information. Note that this output will not show all possible configuration. Many settings which are set to false by default are not shown, like podAffinity or nodeAffinity.

```yaml
metadata: 
  annotations: 
    deployment.kubernetes.io/revision: "1" 
  creationTimestamp: 2017-12-21T13:57:07Z 
  generation: 1 
  labels: 
    app: dev-web 
  name: dev-web 
  namespace: default 
  resourceVersion: "774003"
  uid: d52d3a63-e656-11e7-9319-42010a800003 
```

**annotations**: These values do not configure the object, but provide further information that could be helpful to third-party applications or administrative tracking. Unlike labels, they cannot be used to select an object with kubectl.

**creationTimestamp**: Shows when the object was originally created. Does not update if the object is edited.

**generation**: How many times this object has been edited, such as changing the number of replicas, for example.

**labels**: Arbitrary strings used to select or exclude objects for use with kubectl, or other API calls. Helpful for administrators to select objects outside of typical object boundaries.

**name**: This is a required string, which we passed from the command line. The name must be unique to the namespace.

**resourceVersion**: A value tied to the etcd database to help with concurrency of objects. Any changes to the database will cause this number to change.

**uid**: Remains a unique ID for the life of the object.

## Deployment Configuration Spec

There are two spec declarations for the deployment. The first will modify the ReplicaSet created, while the second will pass along the Pod configuration. 

```yaml
spec:   
  progressDeadlineSeconds: 600   
  replicas: 1   
  revisionHistoryLimit: 10   
  selector:     
    matchLabels:       
      app: dev-web   
  strategy:     
    rollingUpdate:       
      maxSurge: 25%        
      maxUnavailable: 25%     
    type: RollingUpdate
```

**spec**: A declaration that the following items will configure the object being created.

**progressDeadlineSeconds**: Time in seconds until a progress error is reported during a change. Reasons could be quotas, image issues, or limit ranges.

**replicas**: As the object being created is a ReplicaSet, this parameter determines how many Pods should be created. If you were to use kubectl edit and change this value to two, a second Pod would be generated.

**revisionHistoryLimit**: How many old ReplicaSet specifications to retain for rollback.

**selector**: A collection of values ANDed together. All must be satisfied for the replica to match. Do not create Pods which match these selectors, as the deployment controller may try to control the resource, leading to issues.

**matchLabels**: Set-based requirements of the Pod selector. Often found with the matchExpressions statement, to further designate where the resource should be scheduled.

**strategy**: A header for values having to do with updating Pods. Works with the later listed type. Could also be set to Recreate, which would delete all existing pods before new pods are created. With RollingUpdate, you can control how many Pods are deleted at a time with the following parameters.

**maxSurge**: Maximum number of Pods over desired number of Pods to create. Can be a percentage, default of 25%, or an absolute number. This creates a certain number of new Pods before deleting old ones, for continued access.

**maxUnavailable**: A number or percentage of Pods which can be in a state other than Ready during the update process.

**type**: Even though listed last in the section, due to the level of white space indentation, it is read as the type of object being configured. (e.g. RollingUpdate).

## Deployment Configuration Pod Template

Next, we will take a look at a configuration template for the pods to be deployed. We will see some similar values.

```yaml
template: 
  metadata: 
  creationTimestamp: null 
    labels: 
      app: dev-web 
  spec: 
    containers: 
    - image: nginx:1.17.7-alpine 
      imagePullPolicy: IfNotPresent 
      name: dev-web 
      resources: {} 
      terminationMessagePath: /dev/termination-log 
      terminationMessagePolicy: File 
    dnsPolicy: ClusterFirst 
    restartPolicy: Always 
    schedulerName: default-scheduler 
    securityContext: {} 
    terminationGracePeriodSeconds: 30
```

**Note**: If the meaning is basically the same as what was defined before, we will not repeat the definition.

### Explanation of Configuration Elements

**template**: Data being passed to the ReplicaSet to determine how to deploy an object (in this case, containers).

**containers**: Key word indicating that the following items of this indentation are for a container.

**image**: This is the image name passed to the container engine, typically Docker. The engine will pull the image and create the Pod.

**imagePullPolicy**: Policy settings passed along to the container engine, about when and if an image should be downloaded or used from a local cache.

**name**: The leading stub of the Pod names. A unique string will be appended.

**resources**: By default, empty. This is where you would set resource restrictions and settings, such as a limit on CPU or memory for the containers.

**terminationMessagePath**: A customizable location of where to output success or failure information of a container.

**terminationMessagePolicy**: The default value is File, which holds the termination method. It could also be set to FallbackToLogsOnError, which will use the last chunk of container log if the message file is empty and the container shows an error.

**dnsPolicy**: Determines if DNS queries should go to coredns or, if set to Default, use the node's DNS resolution configuration.

**restartPolicy**: Should the container be restarted if killed? Automatic restarts are part of the typical strength of Kubernetes.

**scheduleName**: Allows for the use of a custom scheduler, instead of the Kubernetes default.

**securityContext**: Flexible setting to pass one or more security settings, such as SELinux context, AppArmor values, users and UIDs for the containers to use.

**terminationGracePeriodSeconds**: The amount of time to wait for a SIGTERM to run until a SIGKILL is used to terminate the container.

## Deployment Configuration Status

The status output is generated when the information is requested:

```yaml
status: 
  availableReplicas: 2 
  conditions: 
  - lastTransitionTime: 2017-12-21T13:57:07Z 
    lastUpdateTime: 2017-12-21T13:57:07Z 
    message: Deployment has minimum availability. 
    reason: MinimumReplicasAvailable 
    status: "True" 
    type: Available
  - lastTransitionTime: "2021-07-29T06:00:24Z"
    lastUpdateTime: "2021-07-29T06:00:33Z"
    message: ReplicaSet "test-5f6778868d" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  observedGeneration: 2 
  readyReplicas: 2 
  replicas: 2 
  updatedReplicas: 2 
```

The output above shows what the same deployment were to look like if the number of replicas were increased to two. The times are different than when the deployment was first generated.

### Explanation of Additional Elements

**availableReplicas**: Indicates how many were configured by the ReplicaSet. This would be compared to the later value of readyReplicas, which would be used to determine if all replicas have been fully generated and without error.

**observedGeneration**: Shows how often the deployment has been updated. This information can be used to understand the rollout and rollback situation of the deployment.

## Rendering Deployment Details

You can inspect the details of a Deployment using the describe command. Not only does the output provide information on the number and availability of replicas, it also presents you with the reference to the ReplicaSet. Inspecting the ReplicaSet or the replicated Pods renders references to the parent object managing it:

```shell
$ kubectl describe deployment.apps/my-deploy

...
Replicas:               1 desired | 1 updated | 1 total | 1 available | \
                        0 unavailable
...
NewReplicaSet:   my-deploy-8448c488b5 (1/1 replicas created)
...

$ kubectl describe replicaset.apps/my-deploy-8448c488b5

....
Controlled By:  Deployment/my-deploy
....

$ kubectl describe pod/my-deploy-8448c488b5-mzx5g

....
Controlled By:  ReplicaSet/my-deploy-8448c488b5
....
```

## Labels

By default, creating a Deployment with **kubectl create** adds a label, as we saw in:

```yaml
.... 
    labels: 
        pod-template-hash: "3378155678" 
        run: ghost ....
```

You could then view labels in new columns: 

```shell
$ kubectl get pods -l run=ghost 

NAME                    READY  STATUS   RESTARTS  AGE 
ghost-3378155678-eq5i6  1/1    Running  0         10m

$ kubectl get pods -L run 

NAME                    READY  STATUS   RESTARTS  AGE  RUN
ghost-3378155678-eq5i6  1/1    Running  0         10m  ghost
nginx-3771699605-4v27e  1/1    Running  1         1h   nginx
```

While you typically define labels in pod templates and in the specifications of Deployments, you can also add labels on the fly:

```shell
$ kubectl label pods ghost-3378155678-eq5i6 foo=bar

$ kubectl get pods --show-labels

NAME                    READY  STATUS   RESTARTS  AGE  LABELS
ghost-3378155678-eq5i6  1/1    Running  0         11m  foo=bar, pod-template-hash=3378155678,run=ghost
```

For example, if you want to force the scheduling of a pod on a specific node, you can use a nodeSelector in a pod definition, add specific labels to certain nodes in your cluster and use those labels in the pod. 

```yaml
....
spec: 
    containers: 
    - image: nginx 
    nodeSelector: 
        disktype: ssd
```

## Manually Scaling a Deployment

The API server allows for the configurations settings to be updated for most values. There are some immutable values, which may be different depending on the version of Kubernetes you have deployed. 

A common update is to change the number of replicas running. If this number is set to zero, there would be no containers, but there would still be a ReplicaSet and Deployment. This is the backend process when a Deployment is deleted.

The scaling process is completely abstracted from the end user. You just have to tell the Deployment that you want to scale to a specified number of replicas. Kubernetes will take care of the rest.

Say we wanted to scale from one replica to five replicas,

<img src=".\images\scaling-deployment.png"/>

We have two options: using the scale command or changing the value of the replicas attribute for the live object. 

The following set of commands show the effect of scaling up a Deployment. You can scale a Deployment by changing replica count.

```shell
$ kubectl scale deploy/dev-web --replicas=5

deployment "dev-web" scaled

$ kubectl get deployments

NAME     READY   UP-TO-DATE  AVAILABLE  AGE
dev-web  5/5     5           5          20s
```

A Deployment records scaling activities in its events, which we can view using the describe deployment command:

```shell
$ kubectl describe deployment.apps/dev-web
```

Non-immutable values can be edited via a text editor, as well. Use edit to trigger an update. For example, to change the deployed version of the nginx web server to an older version: 

```shell
$ kubectl edit deployment nginx
....
      containers:
      - image: nginx:1.8 #<<---Set to an older version 
        imagePullPolicy: IfNotPresent
                name: dev-web
....
```

This would trigger a rolling update of the deployment. While the deployment would show an older age, a review of the Pods would show a recent update and older version of the web server application deployed.

## Updating a Deployment

The only thing you need to do is modify the pod template defined in the Deployment resource and Kubernetes will take all the steps necessary to get the actual system state to what’s defined in the resource. Similar to scaling a ReplicationController or ReplicaSet up or down, all you need to do is reference a new image tag in the Deployment’s pod template and leave it to Kubernetes to transform your system so it matches the new desired state.

```shell
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```

## Deployment Rollbacks

Deployments make it easy to roll back to the previously deployed version by telling Kubernetes to undo the last rollout of a Deployment. You can roll back to a specific revision by specifying the revision in the undo command.

Deployment ensures that only a certain number of Pods are down while they are being updated. By default, it ensures that at least 75% of the desired number of Pods are up (25% max unavailable).

Deployment also ensures that only a certain number of Pods are created above the desired number of Pods. By default, it ensures that at most 125% of the desired number of Pods are up (25% max surge).

<img src=".\images\p3_deployments_maxsurge_maxunavailable.jpg"/>

With some of the previous ReplicaSets of a Deployment being kept, you can also roll back to a previous revision by scaling up and down. The number of previous configurations kept is configurable, and has changed from version to version. 

## Rolling Out a New Revision

Application development is usually not stagnant. As part of the software development lifecycle, you build a new feature or create a bug fix and deploy the changes to the Kubernetes cluster as part of the release process. In practice, you’d push a new Docker image to the registry bundling the changes so that they can be run in a container. By default, a Deployment rolls out a new container image using a zero-downtime strategy by updating Pods one by one. Figure below shows the rolling update process for a Deployment controlling two replicas from version 1.2.3 to 2.0.0.

Rolling update of Pods managed by a Deployment:

<img src=".\images\rolling-update-pods-managed-deployment.png"/>

Every Deployment keeps a record of the rollout history. Within the history, a new version of a rollout is called a revision. Before experiencing the rollout of a new revision in practice, let’s inspect the initial state of the Deployment named my-deploy. 

Next, we will have a closer look at rollbacks, using the --record option of the kubectl create command, which allows annotation in the resource definition. The create generator does not have a record function. 

The rollout command shows revision 1, which represents the creation of the Deployment with all its settings:

```shell
$ kubectl create deployment my-deploy --image=nginx:1.14.2 --record

deployment.apps/my-deploy created

$ kubectl describe deployment.apps/my-deploy

...
deployment.kubernetes.io/revision: "1" 
...

$ kubectl rollout history deployment my-deploy

deployment.apps/my-deploy
REVISION  CHANGE-CAUSE
1         <none>

$ kubectl get pods

```

In the next step, we will update the container image used on the Deployment from nginx:1.14.2 to nginx:1.19.2. To do so, either edit the live object or run the set image command:

```shell
$ kubectl set image deployment my-deploy nginx=nginx:1.19.2 --all

deployment.apps/my-deploy image updated
```

Looking at the rollout history again now shows revision 1 and 2. When changing the Pod template of a Deployment—for example, by updating the image—a new ReplicaSet is created. The Deployment will gradually migrate the Pods from the old ReplicaSet to the new one. Inspecting the Deployment details reveals a different name—in this case

```shell
$ kubectl rollout history deployment my-deploy

deployment.apps/my-deploy
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

$ kubectl describe deployment.apps/my-deploy

...
NewReplicaSet:   my-deploy-775ccfcbc8 (1/1 replicas created)
...

$ kubectl rollout status deployment my-deploy

deployment "my-deploy" successfully rolled out

$ kubectl get pods

```

By default, a Deployment persists a maximum of 10 revisions in its history. You can change the limit by assigning a different value to spec.revisionHistoryLimit.

You can also retrieve detailed information about a revision with the rollout history command by providing the revision number using the --revision command-line option. The details of a revision can give you an indication of what exactly changed between revisions:

```shell
$ kubectl rollout history deployments my-deploy --revision=2

deployment.apps/my-deploy with revision #2
Pod Template:
  Labels:	app=my-deploy
	pod-template-hash=9df7d9c6
  Containers:
   nginx:
    Image:	nginx:1.19.2
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

The rolling update strategy ensures that the application is always available to end users. This approach implies that two versions of the same application are available during the update process. As an application developer, you have to be aware that convenience doesn’t come without potential side effects. If you happen to introduce a breaking change to the public API of your application, you might temporarily break consumers, as they could hit revision 1 or 2 of the application. You can change the default update strategy of a Deployment by providing a different value to the attribute strategy.type; however, consider the trade-offs. For example, the value Recreate kills all Pods first, then creates new Pods with the latest revision, causing a potential downtime for consumers. Other strategies like blue-green or canary deployments can be set up.

## Rolling Back to a Previous Revision

Despite the best efforts to avoid them by writing extensive test suites, bugs happen. Not only can the rollout command deploy a new version of an application, you can also roll back to an earlier revision. In the previous section, we rolled out revisions 1 and 2. 

Assume revision 2 contains a bug and we need to quickly revert to revision 1. Should an update fail, due to an improper image version, for example, you can roll back the change to a working version with kubectl rollout undo. You can roll back to a specific revision with the --to-revision option. The following command demonstrates the process:

```shell
$ kubectl rollout undo deployment my-deploy --to-revision=1

deployment.apps/my-deploy rolled back
```

If you look at the rollout history, you’ll find revisions 2 and 3. Kubernetes recognizes that revisions 1 and 3 are exactly the same. For that reason, the rollout history deduplicates revision 1 effectively; revision 1 became revision 3:

```shell
$ kubectl rollout history deployment my-deploy

deployment.apps/my-deploy
REVISION  CHANGE-CAUSE
2         <none>
3         <none>

```

The rollback process works pretty much the same way as rolling out a new revision. Kubernetes switches back to the “old” ReplicaSet, drains the Pods with the image nginx:1.19.2, and starts new Pods with the image nginx:1.14.2.

```shell
$ kubectl get pods

```

You can also edit a Deployment using the kubectl edit command.

You can also pause a Deployment, and then resume.

```shell
$ kubectl rollout pause deployment/ghost

​$ kubectl rollout resume deployment/ghost
```

Please note that you can still do a rolling update on ReplicationControllers with the kubectl rolling-update command, but this is done on the client side. Hence, if you close your client, the rolling update will stop.

## Deployment States

A Deployment enters various states during its lifecycle.

### Progressing:

Kubernetes marks a Deployment as progressing when one of the following tasks is performed:
- The Deployment creates a new ReplicaSet.
- The Deployment is scaling up its newest ReplicaSet.
- The Deployment is scaling down its older ReplicaSet(s).
- New Pods become ready or available (ready for at least MinReadySeconds).

### Complete: 

Kubernetes marks a Deployment as complete when it has the following characteristics:
- All of the replicas associated with the Deployment have been updated to the latest version you've specified, meaning any updates you've requested have been completed.
- All of the replicas associated with the Deployment are available.
- No old replicas for the Deployment are running.

### Failed: 

Your Deployment may get stuck trying to deploy its newest ReplicaSet without ever completing. This can occur due to some of the following factors:
- Insufficient quota
- Readiness probe failures
- Image pull errors
- Insufficient permissions
- Limit ranges
- Application runtime misconfiguration
