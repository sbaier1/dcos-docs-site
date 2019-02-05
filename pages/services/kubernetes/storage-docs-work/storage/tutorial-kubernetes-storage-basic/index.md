---
layout: layout.pug
navigationTitle: Quick Start
title: Kubernetes CSI with MKE
menuWeight: 1
excerpt: Getting started using both dynamic and pre-provisioned Kubernetes CSI deployments
---

A CSI (Container Storage Interface) driver conforms to the [CSI protocol](https://github.com/container-storage-interface/spec/blob/master/spec.md) for storage plugin standardization. The aim of the standard is so that storage providers (e.g. Amazon EBS) can develop one plugin that can work for any container orchestration system, including Kubernetes. Accordingly, MKE takes advantage of the [Amazon EBS CSI driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver) when managing the lifecycle of [Amazon EBS volumes](https://aws.amazon.com/ebs/) across your Kubernetes clusters.

The Amazon EBS CSI enables two methods of using CSI with your Kubernetes clusters. One method is dynamic and the other method is predetermined. The [basic tutorial that follows](/tutorial-kubernetes-storage-basic) demonstrates how to set up, then run through both methods of deployment lifecycle - dynamic and predetermined - each covering the basic steps of the deployment lifecycle: provisioning, attaching, deleting.


# Quick Start - How to Manage Volumes using AWS ELB CSI Driver

This simple tutorial runs through each of two different methods available via the [Amazon EBS CSI driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver) for volume lifecycle management using CSI with a Kubernetes cluster: a dynamic and a pre-provisioned method. The instructions in what follows assume you have `kubectl` access to your Kubernetes cluster being managed by [MKE](https://docs.mesosphere.com/services/kubernetes/2.1.1-1.12.4/overview/).

## Prerequisites

- wget utility installed in your bash environment
- a Kubernetes 1.13 cluster
- access to `kubectl` connected to your Kubernetes cluster(s) being managed by MKE
- ability to provision volumes on AWS in the same AZ as the target Kubernetes cluster(s)

## Setting Up

1. Download the demo repository to your working directory:

    ```bash
    wget https://github.com/mesosphere/csi-driver-deployments/archive/master.zip -O csi-driver-deployments.zip
    unzip csi-driver-deployments.zip && rm csi-driver-deployments.zip
    cd csi-driver-deployments-master/aws-ebs/kubernetes
    ```

1. Grant AWS API IAM permissions:

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

    The recommended approach in this case is to [add the above policy to the EC2 instance roles](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html). If this is not possible in your case, you will need to modify the `secrets.yaml` with `key_id`, `access_key` and optionally `session_token` credentials for a IAM user that does have this policy.


1. Install the AWS EBS CSI driver:

    ```bash
    kubectl apply -f 0.2.0/
    ```

Your Kubernetes cluster is ready to provision volumes via AWS EBS.

## Dynamically Provisioned Volume

### Launch a deployment with a dynamically provisioned EBS volume


1. To begin the demonstration, launch the following application deployment:

    ```bash
    kubectl apply -f example-dynamic/
    ```

    The dummy app `example-dynamic` utilizes dynamically provisioned EBS volumes created by the CSI driver.


1. Request for a volume for the deployment:

    ```bash
    kubectl get pvc -w
    ```

1.  Ping the cluster for a description of attached persistant volume:

    ```bash
    kubectl describe pv
    ```

1. Note the returned value of `VolumeHandle` from the CLI output and confirm this value matches in the AWS console.


### Delete the attached pod

1. Get the name of the pod:

    ```bash
    kubectl get pods
    ```

1. Take note when the pod started writing data:

    ```bash
    kubectl exec -it <pod-name> cat /data/out.txt
    ```

1. Delete the pod:

    ```bash
    kubectl delete pods <pod-name>
    ```

    Deleting the pods takes a few seconds because the driver is unmounting the volume and detaching from the instance.

1. Get a list of your pods again:

    ```bash
    kubectl get pods
    ```

    However, because the deployment is still active, Kubernetes will immediately reattach the volume to a new pod.

1. Now, take note of when this new pod started writing data:

    ```bash
    kubectl exec -it <new-pod-name> cat /data/out.txt
    ```
    You can see that data persisted across pod restart, as the log begins at the same time.


### Delete the deployment and associated dynamically provisioned volume

1. Delete dynamic application deployment:

    ```bash
    kubectl delete deployment ebs-dynamic-app
    ```

1. Check the AWS console, see that the volume will still be "available":

    ![Screenshot of AWS EBS from above still "Available".](/services/img/valavailablevolk8s.png)

1. Delete the dynamic deployment's pvc:

    ```bash
    kubectl delete pvc dynamic
    ```
1. Check the AWS console again, this time the volume is deleted and does not even show up:

![Screenshot of AWS EBS volume absent from where it had been.](/services/img/validatevolumedeletedk8s.png)

## Pre-provisioned Volume

### Launch a deployment with a pre-provisioned EBS volume

 <!-- a use case is using an existing EBS volume in a new cluster -->

1. Create a new EBS volume in the same AZ as the cluster in the AWS console, note the `volumeID` of the new volume:

  ![Screenshot of AWS EBS volume "Created" dialog for newly created volume.](/services/img/createvol.png)

1. Next, edit the `pre-provisioned/pv.yaml`, by inserting the value of `volumeID` from the previous step in for the value of `volumeHandle` in the `spec.csi.volumeHandle`, replacing `__REPLACE_ME__`:

    ```json
        apiVersion: v1
        kind: PersistentVolume
        metadata:
        name: pre-provisioned
        annotations:
            pv.kubernetes.io/provisioned-by: ebs.csi.aws.com
        spec:
        accessModes:
            - ReadWriteOnce
        capacity:
            storage: 1Gi
        csi:
            driver: ebs.csi.aws.com
            fsType: ext4
            volumeHandle: __REPLACE_ME__
        claimRef:
            namespace: default
            name: pre-provisioned
        persistentVolumeReclaimPolicy: Retain
    ```

1. Launch the application deployment with a pre-provisioned EBS volume:

    ```bash
    kubectl apply -f pre-provisioned/
    ```

## Delete the pre-provisioned deployment

1. Delete the application deployment:

    ```bash
    kubectl delete deployment ebs-pre-provisioned-app
    ```

1. Delete the PV and PVC:

    ```bash
    kubectl delete pvc pre-provisioned
    kubectl delete pv pre-provisioned
    ```

1. Check the AWS console again, the volume will be "available" even though the PVC and PV have been deleted. This is because we have set the appropriate retainment parameter in the PV configuration:

    ```json
    persistentVolumeReclaimPolicy: Retain`
    ```

    so that that same EBS volume can be reused in other pods later on if desired.
