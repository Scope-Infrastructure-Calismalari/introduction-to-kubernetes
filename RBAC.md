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

# RBAC

We will look at are in the rbac.authorization.k8s.io group. We actually have four resources: ClusterRole, Role, ClusterRoleBinding, and RoleBinding. They are used for Role Based Access Control (RBAC) to Kubernetes.

```shell
$ curl localhost:8080/apis/rbac.authorization.k8s.io/v1
```

```yaml
... 
    "groupVersion": "rbac.authorization.k8s.io/v1",
    "resources": [ 
... 
    "kind": "ClusterRoleBinding" 
... 
    "kind": "ClusterRole" 
... 
    "kind": "RoleBinding" 
... 
    "kind": "Role" 
...
```

These resources allow us to define Roles within a cluster and associate users to these Roles. For example, we can define a Role for someone who can only read pods in a specific namespace, or a Role that can create deployments, but no services.
