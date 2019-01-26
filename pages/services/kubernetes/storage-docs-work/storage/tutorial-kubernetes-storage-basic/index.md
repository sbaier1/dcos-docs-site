---
layout: layout.pug
navigationTitle: Quick Start
title: Quick Start - Managing volumes with Kubernetes CSI
menuWeight: 1
excerpt: Run through a demonstration of each optional method of deployment lifecycle management.
---

This simple tutorial runs through each of two different methods available via the Amazon EBS CSI driver for volume lifecycle management using CSI with Kubernetes: a dynamic and a pre-provisioned method. The instructions in what follows assume you have `kubectl` access to your Kubernetes cluster being managed by [MKE]().

# Prerequisites

- a Kubernetes cluster (running 1.13?)
- access to `kubectl` connected to your Kubernetes cluster(s) being managed by MKE
- ability to provision volumes on AWS in the same AZ as the target Kubernetes cluster(s)

# Setting Up

## Download the demo repository

Download the demo deployments repository:

```bash
wget https://github.com/mesosphere/csi-driver-deployments/archive/master.zip -O csi-driver-deployments.zip
unzip csi-driver-deployments.zip && rm csi-driver-deployments.zip
cd csi-driver-deployments-master/aws-ebs/kubernetes
```

## Enable the cluster

To ready your cluster for the CSI driver, you first need to [prepare your Kubernetes cluster for the driver](https://kubernetes-csi.github.io/docs/Home.html) - by ensuring certain key features are enabled. For instance, CSI on Kubernetes requires [priveledged pods]() and [mount propagation]() to be enabled. Each feature can be enabled or disabled individually.

However, for this simple demonstration, running the `kubectl` and `kubeapiserver` commands with the following flag:

```bash
--feature-gates=VolumeSnapshotDataSource=true,KubeletPluginsWatcher=true,CSINodeInfo=true,CSIDriverRegistry=true
```

will enable all of the Kubernetes CSI features compatible with your respective versions of these Kubernetes components:

```bash
kubectl --feature-gates=VolumeSnapshotDataSource=true,KubeletPluginsWatcher=true,CSINodeInfo=true,CSIDriverRegistry=true
```

```bash
kube-apiserver --feature-gates=VolumeSnapshotDataSource=true,KubeletPluginsWatcher=true,CSINodeInfo=true,CSIDriverRegistry=true
```

## Grant Permissions

The CSI driver must be connected to the AWS API. This sample IAM policy can be used to grant the driver the necessary permissions:

<!-- following json asset taken from public repo: https://github.com/mesosphere/csi-driver-deployments/tree/master/aws-ebs/kubernetes -->

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteTags",
        "ec2:DeleteVolume",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

# Dynamic Kubernetes CSI Deployment

## Launch the dynamic CSI deployment

1. To begin the demonstration, launch the following application deployment:

    ```bash
    kubectl apply -f example-dynamic/
    ```

    The dummy app `example-dynamic` utilizes the dynamic method for Kubernetes CSI.

    <!-- What is the big difference between the two modes? What makes the dynamic method "dynamic" per se? -->

1. Request for a volume for the deployment:

    ```bash
    kubectl get pvc -w
    ```

1.  Ping the cluster for a description of attached persistant volume:

    ```bash
    kubectl decribe pv
    ```

1. Note the returned value of `VolumeHandle` from the CLI output and confirm this value matches in the AWS console:

    ---OUTPUT/SCREENSHOT NEEDED OF THIS VALIDATION STEP---

## Delete the attached pod

1. Get the name of the pod:

    ```bash
    kubectl get pods
    ```

1. Take note when the pod started writing data:

    ```bash
    kubectl exec -it __POD__ cat /data/out.txt
    ```

1. Delete the pod:

    ```bash
    kubectl delete pods __POD__
    ```

    **Deleting the pods takes a few seconds because the driver is unmounting the volume and detaching from the instance.**

1. Take note again of when the pod started writing data:

    ```bash
    kubectl exec -it __POD__ cat /data/out.txt
    ```
1. You can see that data persisted across pod restart:

    ----SCREENSHOT HERE FOR VALIDATION STEP----


## Delete the dynamic deployment and associated volume

1. Delete dynamic application deployment:

    ```bash
    kubectl delete deployment ebs-dynamic-app
    ```

1. Check the AWS console, see that the volume itself will still be "available":

      ----SCREENSHOT HERE FOR VALIDATION STEP----

1. Delete the dynamic deployment's pvc:

    ```bash
    kubectl delete pvc dynamic
    ```
1. Check the AWS console again, this time the volume is deleted and does not even show up:

      ----SCREENSHOT HERE FOR VALIDATION STEP----

# Pre-provisioned Kubernetes CSI deployment

## Launch the pre-provisioned deployment

 <!-- a use case is using an existing EBS volume in a new cluster -->

1. Create a new volume in the same AZ as the cluster in the AWS console, note the `volumeID` of the new cluster:

      ----SCREENSHOT HERE FOR VALIDATION STEP----

1. Next, edit the `pre-provisioned/pv.yaml`, by inserting the value of `volumeID` from the previous step in for the value of `volumeHandle` in the `spec:csi:volumeHandle`, replacing `__REPLACE_ME__`.

      ----SCREENSHOT HERE FOR VALIDATION STEP----

1. Launch the pre-provisioned application deployment:

    ```bash
    kubectl apply -f pre-provisioned/
    ```
## Delete the pre-provisioned deployment

1. Delete the application deployment:

    ``bash
    kubectl delete deployment ebs-pre-provisioned-app
    ```

1. Delete the PV and PVC

    ```bash
    kubectl delete pvc pre-provisioned
    kubectl delete pv pre-provisioned
    ```

1. Check the AWS console again, the volume will be "available" even though the PV and PVC have been deleted because we have set the appropriate retainment parameter:

`persistentVolumeReclaimPolicy: Retain`

so that the same EBS volume can be reused in other pods.
