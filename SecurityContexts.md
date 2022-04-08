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

# Security Contexts

By default, containers run with the privileges of a root user. That means full access to the filesystem and the ability to run any process, opening up the possibility of security breaches by malicious attackers. You can counteract that risk by defining a security context for a Pod or container. For example, you could specify that the container can only be run as a non-root user. It’s important to remember that the container-level definition takes precedence over the Pod-level security context.

## Understanding Security Contexts

Docker images can define security-relevant instructions to reduce the attack vector for the running container. By default, containers run with root privileges, which provide supreme access to all processes and the container’s filesystem. 

**As a best practice, you should craft the corresponding Dockerfile in a such a way that the container will be run with a user ID other than 0 with the help of the USER instruction.** 

Kubernetes, as the container orchestration engine, can apply additional configuration to increase container security. You’d do so by defining a security context. A security context defines privilege and access control settings for a Pod or a container. 

The following list provides some examples:
- The user ID that should be used to run the Pod and/or container.
- The group ID that should be used for filesystem access.
- Granting a running process inside the container some privileges of the root user but not all of them.

The security context is not a Kubernetes primitive. It is modeled as a set of attributes under the directive **securityContext** within the Pod specification. **Security settings defined on the Pod level apply to all containers running in the Pod; however, container-level settings take precedence.** For more information on Pod-level security attributes, see the **PodSecurityContext API**. Container-level security attributes can be found in the **SecurityContext API**.

To make the functionality more transparent, let’s have a look at a use case. Some images, like the one for the open source reverse-proxy server NGINX, must be run with the root user. Say you wanted to enforce that containers cannot be run as a root user as a sensible security strategy. The YAML manifest shown below defines the security configuration specifically to a container. If you were to run other containers inside the Pod, then the 'runAsNonRoot' setting would not have any effect on them.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: non-root
spec:
  containers:
  - image: nginx:1.18.0
    name: secured-container
    securityContext:
      runAsNonRoot: true
```

You will see that Kubernetes does its job; however, the image is not compatible. Therefore,  the  container  fails  during  the  startup  process  with  the  status  CreateContainerConfigError:

```shell
$ kubectl create -f container-root-user.yaml

pod/non-root created

$ kubectl get pods

NAME       READY   STATUS                       RESTARTS   AGE
non-root   0/1     CreateContainerConfigError   0          7s

$ kubectl describe pod/non-root

...
Events:
Type     Reason     Age              From               Message
----     ------     ----             ----               -------
Normal   Scheduled  <unknown>        default-scheduler  Successfully assigned default/non-root to minikube
Normal   Pulling    18s              kubelet, minikube  Pulling image "nginx:1.18.0"
Normal   Pulled     14s              kubelet, minikube  Successfully pulled image "nginx:1.18.0"
Warning  Failed     0s (x3 over 14s) kubelet, minikube  Error: container has runAsNonRoot and image will run as root
```

There are alternative NGINX images available that are not required to run with the root user. One example is bitnami/nginx. Upon a closer look at the Dockerfile that produced the image, you will find that the container runs with the user ID 1001. Starting the container with the runAsNonRoot directive will work just fine.

There are many other security restrictions you can impose on a container running in Kubernetes. For example, you may want to set the access control for files and directories. Say that, whenever a file is created on the filesystem, the owner of the file should be the arbitrary group ID 3500. The YAML manifest shown below assigns the security context settings on the Pod level as a direct child of the spec attribute.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fs-secured
spec:
  securityContext:
    fsGroup: 3500
  containers:
  - image: nginx:1.18.0
    name: secured-container
    volumeMounts:
    - name: data-volume
      mountPath: /data/app
  volumes:
  - name: data-volume
    emptyDir: {}
```

You can easily verify the effect of setting the filesystem group ID. Open an interactive shell to the container, navigate to the mounted Volume, and create a new file. Inspecting the ownership of the file will show the group ID 3500 automatically assigned to it:

```shell
$ kubectl create -f pod-file-system-group.yaml

pod/fs-secured created

$ kubectl get pods

NAME         READY   STATUS    RESTARTS   AGE
fs-secured   1/1     Running   0          24s

$ kubectl exec -it fs-secured -- /bin/sh

# cd /data/app
# touch logs.txt
# ls -l
-rw-r--r-- 1 root 3500 0 Jul  9 01:41 logs.txt
```
