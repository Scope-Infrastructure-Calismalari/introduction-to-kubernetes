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

# Service Accounts

A Service Account defines the permissions for a Pod that needs to communicate with the API server. **Every Pod uses a Service Account. If none is defined, Kubernetes will automatically assign the default Service Account.** The default Service Account uses the privileges of an unauthenticated user. You can create a custom Service Account to allow for more fine-grained control. Assigning a custom Service Account to a Pod is as easy as defining it with 'spec.serviceAccountName'.

## Understanding Service Accounts

We’ve been using the kubectl executable to run operations against a Kubernetes cluster. Under the hood, its implementation calls the API server by making an HTTP call to the exposed endpoints. **Some applications running inside of a Pod may have to communicate with the API server as well.** For example, the application may ask for specific cluster node information or available namespaces.

**Pods use a Service Account to authenticate with the API server through an authentication token. A Kubernetes administrator assigns rules to a Service Account via role-based access control (RBAC) to authorize access to specific resources and actions.** 

Using a Service Account to communicate with an API server:

<img src=".\images\using-service-account.png"/>

So far, we haven’t defined a Service Account for a Pod. **If not assigned explicitly, a Pod uses the default Service Account. The default Service Account has the same permissions as an unauthenticated user. This means that the Pod cannot view or modify the cluster state nor list or modify any of its resources.**

You can query for the available Service Accounts with the subcommand serviceaccounts. You should only see the default Service Account listed:

```shell
$ kubectl get serviceaccounts

NAME      SECRETS   AGE
default   1         25d
```

Kubernetes models the authentication token with the Secret primitive. It’s easy to identify the corresponding Secret for a Service Account. Retrieve the YAML representation of the Service Account and look at the attribute secrets. **In the Secret, you can find the Base64-encoded values of the current namespace, the cluster certificate, and the authentication token**:

```shell
$ kubectl get serviceaccount default -o yaml | grep -A 1 secrets:

secrets:
- name: default-token-bf8rh

$ kubectl get secret default-token-bf8rh -o yaml

apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiB...0FURS0tLS0tCg==
  namespace: ZGVmYXVsdA==
  token: ZXlKaGJHY2lPaUp...ThzU0poeFMxR013
kind: Secret
...
```

You will find that any live Pod object indicates its assigned Service Account in the spec section. The following command renders the value in the terminal:

```shell
$ kubectl run nginx --image=nginx --restart=Never

pod/nginx created

$ kubectl get pod nginx -o yaml

apiVersion: v1
kind: Pod
metadata:
  ...
spec:
  serviceAccountName: default
...
```

## Creating and Assigning Custom Service Accounts

It’s very possible that you’ll want to grant certain permissions to an application running in a Pod. For that purpose, you’d create a custom Service Account and bind the relevant permissions to it. For the most part, this is the job of a Kubernetes administrator; however, it’s good to have a basic understanding of the process from the perspective of an application developer.

To create a new Service Account, you can simply use the create command:

```shell
$ kubectl create serviceaccount custom

serviceaccount/custom created
```

Now, there are two ways to assign the Service Account to a Pod. You can either edit the YAML manifest and add the serviceAccountName attribute as shown above, or you can use the --serviceaccount flag in conjunction with the run command when creating the Pod:

```shell
$ kubectl run nginx --image=nginx --restart=Never --serviceaccount=custom

pod/nginx created

$ kubectl get pod nginx -o yaml

apiVersion: v1
kind: Pod
metadata:
  ...
spec:
  serviceAccountName: custom
...
```

