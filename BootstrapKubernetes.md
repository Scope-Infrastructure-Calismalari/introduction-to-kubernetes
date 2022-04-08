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

# Bootstrap Kubernetes

## kubeadm

**kubeadm** is a tool for creating Kubernetes clusters.

**kubeadm** performs the actions necessary to get a minimum viable cluster up and running. 

By design, it cares only about bootstrapping, not about provisioning machines. 

Likewise, installing various nice-to-have addons, like the Kubernetes Dashboard, monitoring solutions, and cloud-specific addons, is not in scope.

| command | description |
|-------- | ----------- |
| ***kubeadm init*** | to bootstrap a Kubernetes control-plane node |
| ***kubeadm join*** | to bootstrap a Kubernetes worker node and join it to the cluster |
| ***kubeadm upgrade*** | to upgrade a Kubernetes cluster to a newer version |
| ***kubeadm config*** | if you initialized your cluster using kubeadm v1.7.x or lower, to configure your cluster for kubeadm upgrade |
| ***kubeadm token*** | to manage tokens for kubeadm join |
| ***kubeadm reset*** | to revert any changes made to this host by kubeadm init or kubeadm join |
| ***kubeadm version*** | to print the kubeadm version |
| ***kubeadm alpha*** | to preview a set of features made available for gathering feedback from the community |

