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

# Volumes and Data

Containers store data in a temporary filesystem, which is empty each time a new Pod is started. Application developers need to persist data beyond the lifecycles of the containers, Pods, node, and cluster. Typical examples include persistent log files or data in a database.

Each container running in a Pod provides a temporary filesystem. Applications running in the container can read from it and write to it. A container’s temporary filesystem is isolated from any other container or Pod and is not persisted beyond a Pod restart. 

Applications in a container perform file I/O only to the container’s file system. If the read/write location is not associated to an external mount, then the files are lost at the end of the container’s life.

Container engines have traditionally not offered storage that outlives the container. As containers are considered transient, this could lead to a loss of data, or complex exterior storage options. A Kubernetes volume shares the Pod lifetime, not the containers within. Should a container terminate, the data would continue to be available to the new container. 

Kubernetes offers the concept of a Volume to implement the use case. A Pod mounts a Volume to a path in the container. Any data written to the mounted storage will be persisted beyond a container restart. Kubernetes offers a wide range of Volume types to fulfill different requirements.

A Volume is a Kubernetes capability that persists data beyond a Pod restart. A volume is a directory, possibly pre-populated, made available to containers in a Pod. Essentially, a Volume is a directory that’s shareable between multiple containers of a Pod. The creation of the directory, the backend storage of the data and the contents depend on the volume type. As of v1.13, there were 27 different volume types ranging from rbd to gain access to Ceph, to NFS, to dynamic volumes from a cloud provider like Google's gcePersistentDisk. Each has particular configuration options and dependencies. 

The Container Storage Interface (CSI) adoption enables the goal of an industry standard interface for container orchestration to allow access to arbitrary storage systems. Currently, volume plugins are "in-tree", meaning they are compiled and built with the core Kubernetes binaries. This "out-of-tree" object will allow storage vendors to develop a single driver and allow the plugin to be containerized. This will replace the existing Flex plugin which requires elevated access to the host node, a large security concern. 

Should you want your storage lifetime to be distinct from a Pod, you can use Persistent Volumes. These allow for empty or pre-populated volumes to be claimed by a Pod using a Persistent Volume Claim, then outlive the Pod. Data inside the volume could then be used by another Pod, or as a means of retrieving data. 

There are two API objects which exist to provide data to a Pod already. Encoded data can be passed using a Secret and non-encoded data can be passed with a ConfigMap. These can be used to pass important data like SSH keys, passwords, or even a configuration file like '**/etc/hosts**'.

Be aware that any capacity mentioned does not represent an actual limit on the amount of space a container can consume. Should a volume have a capacity of, for example, 10G, that does not mean there is 10G of backend storage available. There could be more or less. If there is more and an application were to continue to write it, there is no block on how much space is used, with the possible exception of a newer limit on ephemeral storage usage. A new CSI driver has become available, so at least we can track actual usage.

PersistentVolumes even store data beyond a Pod or cluster/node restart. Those objects are decoupled from the Pod’s lifecycle and are therefore represented by a Kubernetes primitive. The PersistentVolumeClaim abstracts the underlying implementation details of a PersistentVolume and acts as an intermediary between Pod and PersistentVolume.

Persistent Volumes are a specific category of the wider concept of Volumes. The mechanics for Persistent Volumes are slightly more complex. The Persistent Volume is the resource that actually persists the data to an underlying physical storage. The Persistent Volume Claim represents the connecting resource between a Pod and a Persistent Volume responsible for requesting the storage. Finally, the Pod needs to claim the Persistent Volume and mount it to a directory path available to the containers running inside of the Pod.

## Introducing Volumes

Applications running in a container can use the temporary filesystem to read and write files. In case of a container crash or a cluster/node restart, the kubelet will restart the container. Any data that had been written to the temporary filesystem is lost and cannot be retrieved anymore. The container effectively starts with a clean slate again.

A container using the temporary filesystem versus a Volume:

<img src=".\images\container-temporary-filesystem-versus-volume.png"/>

A Pod specification can declare one or more volumes and where they are made available. Each requires a name, a type, and a mount point. 

Keeping acquired data or ingesting it into other containers is a common task, typically requiring the use of a PersistentVolumeClaim (pvc).

The same volume can be made available to multiple containers within a Pod, which can be a method of container-to-container communication. A volume can be made available to multiple Pods, with each given an access mode to write. There is no concurrency checking, which means data corruption is probable, unless outside locking takes place.

<img src=".\images\KubernetesPodVolumes.png"/>

A particular access mode is part of a Pod request. As a request, the user may be granted more, but not less access, though a direct match is attempted first. The cluster groups volumes with the same mode together, then sorts volumes by size, from smallest to largest. The claim is checked against each in that access mode group, until a volume of sufficient size matches. 

The three access modes are:
- **ReadWriteOnce**: which allows read-write by a single node
- **ReadOnlyMany**: which allows read-only by multiple nodes
- **ReadWriteMany**: which allows read-write by many nodes. 

Thus two pods on the same node can write to a **ReadWriteOnce**, but a third pod on a different node would not become ready due to a **FailedAttachVolume** error.

We had seen an example of containers within a pod while learning about networking. Using the same example we see that the same volume can be used by all containers. In this case both are using the /data/ directory. MainApp reads and writes to the volume, and Logger uses the volume read-only. It then writes to a different volume /log/.

Multiple Volumes in a Pod:

<img src=".\images\NewPodNetwork.png"/>

When a volume is requested, the local kubelet uses the **kubelet_pods.go** script to map the raw devices, determine and make the mount point for the container, then create the symbolic link on the host node filesystem to associate the storage to the container. The API server makes a request for the storage to the **StorageClass** plugin, but the specifics of the requests to the backend storage depend on the plugin in use.
 
If a request for a particular **StorageClass** was not made, then the only parameters used will be access mode and size. The volume could come from any of the storage types available, and there is no configuration to determine which of the available ones will be used.

## Volume Spec

One of the many types of storage available is an **emptyDir**. The kubelet will create the directory in the container, but not mount any storage. Any data created is written to the shared container space. As a result, it would not be persistent storage. When the Pod is destroyed, the directory would be deleted along with the container.

```yaml
apiVersion: v1
kind: Pod
metadata: 
    name: fordpinto 
    namespace: default
spec: 
    containers: 
    - image: simpleapp 
      name: gastank 
      command: 
        - sleep 
        - "3600" 
      volumeMounts: 
      - mountPath: /scratch 
        name: scratch-volume 
    volumes: 
    - name: scratch-volume 
      emptyDir: {}
```

The YAML file above would create a Pod with a single container with a volume named **scratch-volume** created, which would create the '**/scratch**' directory inside the container.

## Volume Types

There are several types that you can use to define volumes, each with their pros and cons. Some are local, and many make use of network-based resources.

### GCEpersistentDisk and awsElasticBlockStore

In GCE or AWS, you can use volumes of type GCEpersistentDisk or awsElasticBlockStore, which allows you to mount GCE and EBS disks in your Pods, assuming you have already set up accounts and privileges.

### emptyDir and hostPath

**emptyDir** and **hostPath** volumes are easy to use. As mentioned, **emptyDir** is an empty directory that gets erased when the Pod dies, but is recreated when the container restarts. The **hostPath** volume mounts a resource from the host node filesystem. The resource could be a directory, file socket, character, or block device. These resources must already exist on the host to be used. There are two types, **DirectoryOrCreate** and **FileOrCreate**, which create the resources on the host, and use them if they don't already exist. 

### configMap and secret

Provides a way to inject configuration data.

### NFS and iSCSI

NFS (Network File System) and iSCSI (Internet Small Computer System Interface) are straightforward choices for multiple readers scenarios.

### rbd, CephFS and GlusterFS

rbd for block storage or CephFS and GlusterFS, if available in your Kubernetes cluster, can be a good choice for multiple writer needs.

### Other Volume Types

Besides the volume types we just mentioned, there are many other possible, with more being added: azureDisk, azureFile, csi, downwardAPI, fc (fibre channel), flocker, gitRepo, local, projected, portworxVolume, quobyte, scaleIO, secret, storageos, vsphereVolume, persistentVolumeClaim, CSIPersistentVolumeSource, etc.​

CSI allows for even more flexibility and decoupling plugins without the need to edit the core Kubernetes code. It was developed as a standard for exposing arbitrary plugins in the future. 

## Creating and Accessing Volumes

Defining a Volume for a Pod requires two steps. First, you need to declare the Volume itself using the attribute 'spec.volumes'. As part of the definition, you provide the name and the type. Just declaring the Volume won’t be sufficient, though. Second, the Volume needs to be mounted to a path of the consuming container via 'spec.containers.volumeMounts'. The mapping between the Volume and the Volume mount occurs by the matching name.

In example below, stored in the file pod-with-volume.yaml here, you can see the definition of a Volume with type emptyDir. The Volume has been mounted to the path '/var/logs' inside of the container named nginx:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: business-app
spec:
  volumes:
  - name: logs-volume
    emptyDir: {}
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/logs
      name: logs-volume
```

Let’s create the Pod and see if we can interact with the mounted Volume. The following commands open an interactive shell after the Pod’s creation, then navigate to the mount path. You can see that the Volume type emptyDir initializes the mount path as an empty directory. New files and directories can be created as needed without limitations:

```shell
$ kubectl create -f pod-with-volume.yaml

pod/business-app created

$ kubectl get pod business-app

NAME           READY   STATUS    RESTARTS   AGE
business-app   1/1     Running   0          43s

$ kubectl exec business-app -it -- /bin/sh

# cd /var/logs
# pwd
/var/logs
# ls
# touch app-logs.txt
# ls
app-logs.txt
```

## Shared Volume Example

The following YAML file creates a pod, exampleA, with two containers, both with access to a shared volume:

For an illustrative use case of the shared volume type mounted by more than one container::

```yaml
....
   containers:
   - name: alphacont
     image: busybox
     volumeMounts:
     - mountPath: /alphadir
       name: sharevol
   - name: betacont
     image: busybox
     volumeMounts:
     - mountPath: /betadir
       name: sharevol
   volumes:
   - name: sharevol
     emptyDir: {}   
```

```shell
$ kubectl exec -ti exampleA -c betacont -- touch /betadir/foobar

$ kubectl exec -ti exampleA -c alphacont -- ls -l /alphadir

total 0
-rw-r--r-- 1 root root 0 Nov 19 16:26 foobar
```

You could use emptyDir or hostPath easily, since those types do not require any additional setup, and will work in your Kubernetes cluster.

Note that one container (betacont) wrote, and the other container (alphacont) had immediate access to the data. There is nothing to keep the containers from overwriting the other's data. Locking or versioning considerations must be part of the containerized application to avoid corruption.

## Persistent Volumes and Claims

Data stored on Volumes outlive a container restart. In many applications, the data lives far beyond the lifecycles of the applications, container, Pod, nodes, and even the clusters themselves. **Data persistence ensures the lifecycles of the data are decoupled from the lifecycles of the cluster resources.** 

A typical example would be data persisted by a database. That’s the responsibility of a Persistent Volume. 

Kubernetes models persist data with the help of two primitives: the PersistentVolume and the PersistentVolumeClaim.

The PersistentVolume is the storage device in a Kubernetes cluster. The PersistentVolume is completely decoupled from the Pod and therefore has its own lifecycle. The object captures the source of the storage (e.g., storage made available by a cloud provider). A PersistentVolume is either provided by a Kubernetes administrator or assigned dynamically by mapping to a storage class.

A **persistent volume (pv)** is a storage abstraction used to retain data longer then the Pod using it. Pods define a volume of type **persistentVolumeClaim (pvc)** with various parameters for size and possibly the type of backend storage known as its **StorageClass**. The cluster then attaches the **persistentVolume**. 

The PersistentVolumeClaim requests the resources of a PersistentVolume—for example, the size of the storage and the access type. In the Pod, you will use the type persistentVolumeClaim to mount the abstracted PersistentVolume by using the PersistentVolumeClaim.

Kubernetes will dynamically use volumes that are available, irrespective of its storage type, allowing claims to any backend storage.

Figure below shows the relationship between the Pod, the PersistentVolumeClaim, and the PersistentVolume.

Claiming a PersistentVolume from a Pod:

<img src=".\images\claiming-persistentvolume-pod.png"/>

```shell
$ kubectl get pv

$ kubectl get pvc
```

There are several phases to persistent storage. 

### Phases to Persistent Storage

**Provision**: **Provisioning** can be from PVs created in advance by the cluster administrator, or requested from a dynamic source, such as the cloud provider.

**Bind**: **Binding** occurs when a control loop on the cp notices the PVC, containing an amount of storage, access request, and optionally, a particular **StorageClass**. The watcher locates a matching PV or waits for the **StorageClass** provisioner to create one. The PV must match at least the storage amount requested, but may provide more.

**Use**: The **use** phase begins when the bound volume is mounted for the Pod to use, which continues as long as the Pod requires.

**Release**: **Releasing** happens when the Pod is done with the volume and an API request is sent, deleting the PVC. The volume remains in the state from when the claim is deleted until available to a new claim. The resident data remains depending on the **persistentVolumeReclaimPolicy**.

**Reclaim**: The reclaim phase has three options:
- **Retain**, which keeps the data intact, allowing for an administrator to handle the storage and data.
- **Delete** tells the volume plugin to delete the API object, as well as the storage behind it. 
- The **Recycle** option runs an '**rm -rf /mountpoint**' and then makes it available to a new claim. With the stability of dynamic provisioning, the Recycle option is planned to be deprecated.

## Creating PersistentVolumes

A PersistentVolume can only be created using the manifest-first approach. At this time, kubectl does not allow the creation of a PersistentVolume using the create command. Every PersistentVolume needs to define the storage capacity using 'spec.capacity' and an access mode set via 'spec.accessModes'. Table below provides a high-level overview of the available access modes.

PersistentVolume access modes:

| Type | Description |
| :--: | :--: |
| ReadWriteOnce | Read/write access by a single node. |
| ReadOnlyMany | Read-only access by many nodes. |
| ReadWriteMany | Read/write access by many nodes. |

The following example shows a basic declaration of a Persistent Volume using the **hostPath** type.

```yaml
kind: PersistentVolume 
apiVersion: v1 
metadata: 
  name: 10Gpv01 
  labels: 
    type: local 
spec: 
  capacity: 
    storage: 10Gi 
  accessModes: 
    - ReadWriteOnce 
  hostPath: 
    path: "/somepath/data01"
```

Each type will have its own configuration settings. For example, an already created Ceph or GCE Persistent Disk would not need to be configured, but could be claimed from the provider.

Persistent volumes are not a namespaces object, but persistent volume claims are. A beta feature of v1.13 allows for static provisioning of Raw Block Volumes, which currently support the Fibre Channel plugin, AWS EBS, Azure Disk and RBD plugins among others. There is a lot of development and change in this area, with plugins adding dynamic provisioning.

The use of locally attached storage has been graduated to a stable feature. This feature is often used as part of distributed filesystems and databases.

Example below creates a PersistentVolume named db-pv with a storage capacity of 1Gi and read/write access by a single node. The attribute hostPath mounts the directory /data/db from the host node’s filesystem. We’ll store the YAML mainfest in the file db-pv.yaml.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/db
```

Upon inspection of the created PersistentVolume, you’ll find most of the information you provided in the manifest. The status Available indicates that the object is ready to be claimed. The reclaim policy determines what should happen with the PersistentVolume after it has been released from its claim. By default, the object will be retained. The following example uses the short-form command pv to avoid having to type persistentvolume:

```shell
$ kubectl create -f db-pv.yaml

persistentvolume/db-pv created

$ kubectl get pv db-pv

NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
db-pv  1Gi        RWO            Retain           Available            10s
```

The PersistentVolume has not been mounted by a Pod yet. Therefore, inspecting the details of the object shows <none>. Using the describe command is a good way to verify if the PersistentVolumeClaim was mounted properly:

```shell
$ kubectl describe pvc db-pvc

...
Mounted By:    <none>
...
```

## Persistent Volume Claim

With a persistent volume created in your cluster, you can then write a manifest for a claim, and use that claim in your pod definition. In the Pod, the volume uses the **persistentVolumeClaim**.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1 
metadata: 
  name: myclaim
spec: 
  accessModes: 
    - ReadWriteOnce 
  resources: 
    requests: 
      storage: 8GI
```

In the Pod configuration you use created PersistentVolumeClaim as given below:

```yaml
    containers:
.... 
    volumes: 
      - name: test-volume 
        persistentVolumeClaim: 
          claimName: myclaim
```

Another example for the mounting the PersistentVolumeClaim in the Pod can be given as follows. You already learned how to mount a Volume in a Pod. The big difference here, is using 'spec.volumes.persistentVolumeClaim' and providing the name of the PersistentVolumeClaim.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-consuming-pvc
spec:
  volumes:
    - name: app-storage
      persistentVolumeClaim:
        claimName: db-pvc
  containers:
  - image: alpine
    name: app
    command: ["/bin/sh"]
    args: ["-c", "while true; do sleep 60; done;"]
    volumeMounts:
      - mountPath: "/mnt/data"
        name: app-storage
```
Let’s assume we stored the configuration in the file app-consuming-pvc.yaml. After creating the Pod from the manifest, you should see the Pod transitioning into the Ready state. The describe command will provide additional information on the Volume:

```shell
$ kubectl create -f app-consuming-pvc.yaml

pod/app-consuming-pvc created

$ kubectl get pods

NAME                READY   STATUS    RESTARTS   AGE
app-consuming-pvc   1/1     Running   0          3s

$ kubectl describe pod app-consuming-pvc

...
Volumes:
  app-storage:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim \
                in the same namespace)
    ClaimName:  db-pvc
    ReadOnly:   false
...
```

The PersistentVolumeClaim now also shows the Pod that mounted it:

```shell
$ kubectl describe pvc db-pvc

...
Mounted By:    app-consuming-pvc
...
```

You can now go ahead and open an interactive shell to the Pod. Navigating to the mount path at '/mnt/data' gives you access to the underlying PersistentVolume:

```shell
$ kubectl exec app-consuming-pvc -it -- /bin/sh

# cd /mnt/data
# ls -l
total 0
# touch test.db
# ls -l
total 0
-rw-r--r--    1 root     root             0 Sep 29 23:59 test.db
```

For other Volume types, the Pod configuration could also be as complex as this example:

```yaml
  volumeMounts: 
    - name: Cephpd 
      mountPath: /data/rbd 
  volumes: 
    - name: rbdpd 
      rbd: 
        monitors: 
        - '10.19.14.22:6789' 
        - '10.19.14.23:6789' 
        - '10.19.14.24:6789' 
        pool: k8s 
        image: client 
        fsType: ext4 
        readOnly: true 
        user: admin 
        keyring: /etc/ceph/keyring 
        imageformat: "2" 
        imagefeatures: "layering"
```

## Static Versus Dynamic Provisioning

A PersistentVolume can be created statically or dynamically. If you go with the static approach, then you need to create storage device first and reference it by explicitly creating an object of kind PersistentVolume. The dynamic approach doesn’t require you to create a PersistentVolume object. It will be automatically created from the PersistentVolumeClaim by setting a storage class name using the attribute 'spec.storageClassName'.

A storage class is an abstraction concept that defines a class of storage device (e.g., storage with slow or fast performance) used for different application types. It’s usually the job of a Kubernetes administrator to set up storage classes. Minikube already creates a default storage class named standard, which you can query with the following command:

```shell
$ kubectl get storageclass

NAME                PROVISIONER               RECLAIMPOLICY    VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)  k8s.io/minikube-hostpath  Delete           Immediate   false                  108d

```

## Dynamic Provisioning

While handling volumes with a persistent volume definition and abstracting the storage provider using a claim is powerful, a cluster administrator still needs to create those volumes in the first place. Starting with Kubernetes v1.4, **Dynamic Provisioning allowed for the cluster to request storage from an exterior, pre-configured source. API calls made by the appropriate plugin allow for a wide range of dynamic storage use.** 

The **StorageClass** API resource allows an administrator to define a persistent volume provisioner of a certain type, passing storage-specific parameters. 

With a **StorageClass** created, a user can request a claim, which the API Server fills via auto-provisioning. The resource will also be reclaimed as configured by the provider. AWS and GCE are common choices for dynamic storage, but other options exist, such as a Ceph cluster or iSCSI. Single, default class is possible via annotation.

Here is an example of a **StorageClass** using GCE: 

```yaml
apiVersion: storage.k8s.io/v1 
kind: StorageClass
metadata: 
  name: fast      # Could be any name
provisioner: kubernetes.io/gce-pd
parameters: 
  type: pd-ssd 
```

There are also providers you can create inside of you cluster, which use Custom Resource Definitions and local operators to manage storage.

- Rook: Is a graduated CNCF project which allows easy management of several back-end storage types, including an easy way to deploy Ceph.
- Longhorn: Is a CNCF project in sandbox status, developed by Rancher which also allows management of local storage.

## StorageClass

StorageClass is a mechanism of defining different classes of storages in Kubernetes. Your Kubernetes administrator along with your storage administrator might classify different types of storages available in your organization and make a reference of it in Kubernetes. These **StorageClass**es can then be directly referenced in a **Persistent Volume Claim** which can later be assigned to a pod. 

StorageClass definition requires the below information - 

- **Provisioners**: AWSElasticBlockStore, AzureFile, AzureDisk, GCEPersistentDisk, Glusterfs, iSCSI, NFS, VsphereVolume etc.
- **Parameters**: type of storage (pd,ssd,magnetic), diskformat, datastore etc. 
- **Reclaim Polcy**: Retain or Delete

Kubernetes ships some provisioners which are also called as internal provisioners. Some examples are EBS, Azure Disk, GCE PD etc. These internal provisioners are usually referred with a prefix of `kubernetes.io`. The kubernetes incubator repository also has a variety of external provisioners which can be used with storage types that dont have an internal provisioners. Few examples of external provisioners are - AWS EFS provisioner, CephFS, iSCSI, FlexVolumes, etc. 

Storage classes helps in dynamic provisioning of PV. Which means that your developers / devops engineers need not worry about PV provisioning before hand. Your Kubernetes administrator can set a default storage class for your cluster. If a PVC doesnt specify a PV or a storage class name, the default storage class is used. This PVC then automatically creates a new PV and the corresponding storage is then assigned. 

Managed Kubernetes services like AKS, EKS and GKE provides a default storage class which points to their respective disk storage. 

## Using Rook for Storage Orchestration

In keeping with the decoupled and distributed nature of the Cloud technology, the Rook project allows orchestration of storage using multiple storage providers.
As with other agents of the cluster, Rook uses custom resource definitions (CRD) and a custom operator to provision storage according to the backend storage type, upon API call.

Several storage providers are supported:
- Ceph
- Cassandra
- Network File System (NFS).

