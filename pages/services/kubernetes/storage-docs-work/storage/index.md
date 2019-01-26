---
layout: layout.pug
navigationTitle: Storage
title: Storage
menuWeight: 55
excerpt: How MKE leverages the Amazon EBS CSI driver to manage shared volumes
---

A CSI (Container Storage Interface) driver conforms to the [CSI protocol](https://github.com/container-storage-interface/spec/blob/master/spec.md) for storage plugin standardization. The aim of the protocol is so that storage providers (e.g. Amazon EBS) can develop one plugin that can work for any container orchestration system, including Kubernetes. Accordingly, MKE takes advantage of the [Amazon EBS CSI driver]() when managing the lifecycle of [Amazon EBS volumes]() across your Kubernetes clusters.

The Amazon EBS CSI enables two methods of managing the lifecycles of your Amazon EBS volumes connected to your Kubernetes clusters. One method is dynamic. The other method is predetermined.

The [basic tutorial that follows](/tutorial-kubernetes-storage-basic) demonstrates how to set up, then run through both methods of deployment lifecycle - dynamic and predetermined - each covering the basic steps of the deployment lifecycle: provisioning, attaching, deleting.
