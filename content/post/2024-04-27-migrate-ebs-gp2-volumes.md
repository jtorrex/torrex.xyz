---
title: "Migrate EBS gp2 volumes to gp3 for a MongoDB community on EKS"
date: "2024-04-27"
tags:
- aws
- kubernetes
- eks
- storage
categories:
- blog
- devops
- aws 
- cloud
author: jtorrex
---

# Migrate EBS gp2 volumes to gp3 for a MongoDB community on EKS

## Description:

This post describes how to migrate previously used EBS gp2 volumes created for a MongoDB Community deployment on EKS to EBS gp3, enabling you to leverage the benefits of performance and costs.

## EBS Volumes types

* The EBS gp2 volume type is the most commonly deployed for a broad range of use cases. EBS gp2 is a volume type that ties its performance to its volume size and offers a baseline performance of 3 IOPS per GiB of provisioned volume size.
For volumes under 1,000 GiB, GPS offers burst up to 3,000 IOPS based on a burst credit balance. Throughput performance is calculated based on volume size, up to the throughput limit of 250 MiB/s and is described here.

* The EBS gp3 volume type was released on December 1, 2020. The EBS gp3 provides a baseline performance of 3000 IOPS, and 125 MiB/s independent of volume size.
With EBS gp3 volumes, AWS allows the customers to modify both the IOPS and the throughput capabilities of each volume to help match a given workload requirement.

* https://aws.amazon.com/blogs/storage/how-to-choose-the-best-amazon-ebs-volume-type-for-your-self-managed-database-deployment/

![](https://gitlab.com/jtorrex/torrex.xyz/-/raw/54d539124088e6c3a2a3343162efbe0bc24502b5/resources/assets/images/mongodb-gp2-gp3/ebs-volumes.png)

## Scenario

The scenario involves a MongoDB Community deployment over EKS managed by FluxCD, which by default deploys two volumes for data and logs backed by a GP2 StorageClass. As mentioned previously, the intention of the process is to migrate both volumes to EBS gp3.

* https://github.com/mongodb/mongodb-kubernetes-operator/blob/master/config/samples/mongodb.com_v1_mongodbcommunity_cr.yaml

![](https://gitlab.com/jtorrex/torrex.xyz/-/raw/master/resources/assets/images/mongodb-gp2-gp3/statefulset-volumes.png)

A valid example manifest similar to the used on the MongoDB community deployment, can be:

```
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: example-mongodb
spec:
  members: 3
  type: ReplicaSet
  version: "6.0.5"
  security:
    authentication:
      modes: ["SCRAM"]
  users:
    - name: my-user
      db: admin
      passwordSecretRef:
        name: my-user-password
      roles:
        - name: clusterAdmin
          db: admin
        - name: userAdminAnyDatabase
          db: admin
      scramCredentialsSecretName: my-scram
  statefulSet:
    spec:
      volumeClaimTemplates:
        - metadata:
            name: data-volume
          spec:
            accessModes: [ "ReadWriteOnce", "ReadWriteMany" ]
            resources:
              requests:
                storage: 50Gi
```

This manifest deploys a StatefulSet that creates the necessary requirements to implement persistence in MongoDB.

It originally creates two PersistentVolumes for data and logs, along with their respective PersistentVolumeClaims.

The default StorageClass was backed by EBS gp2

## Strategy

The steps to follow during the migration are:

- Identify the volumes involved on the MongoDB deployment
- Back up current YAML files for PVC, PV, SC
- Downscale the MongoDB deployment to avoid unwanted modifications during migration and inconsistencies on the DB.
- Take snapshots of the EBS gp2 volumes
- Create new gp3 volumes from the previous snapshots
- Create a new gp3 type StorageClass and configure it as 'default'
- Delete the old PersistentVolumeClaims
- Delete the old PersistentVolumes
- Create new PersistentVolumes
- Rescale the MongoDB deployment to verify that the migration was successful

## Identify the volumes involved on the MongoDB deployment

First step should be to identify the volumes involved on the MongoDB deployment.

List the PersistentVolumeClaims in the MongoDB namespace:

```
❯ kubectl get pvc -n mongo
NAME                            STATUS   VOLUME                    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-volume-example-mongodb-0   Bound    pv-data-example-mongodb   10Gi       RWO            gp2            3d12h
logs-volume-example-mongodb-0   Bound    pv-logs-example-mongodb   2Gi        RWO            gp2            3d12h
```

In our case there are two, one for data and another for logs.

With this information, the PersistentVolumes involved are identified.

Describing the PersistentVolumes you can obtain the AWS EBS volumes reference for each of them.

```
❯ kubectl describe pv pv-data-example-mongodb | grep 'AWSElasticBlockStore' -A 4
    Type:       AWSElasticBlockStore (a Persistent Disk resource in AWS)
    VolumeID:   vol-0aa7e29abeb727d52
    FSType:     ext4
    Partition:  0
    ReadOnly:   false
```

Take care and remember the VolumeID parameter as it's required later.

## Backing up old YAML files

For having a reference of the old manifests applied to define the PersistentVolumes and PersistentVolumeClaims it is recommended to export it.

* Data PersistentVolumeClaim

```
> kubectl get pvc data-volume-example-mongodb-0 -n mongo -o yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: ebs.csi.aws.com
    volume.kubernetes.io/selected-node: ##########################################
    volume.kubernetes.io/storage-provisioner: ebs.csi.aws.com
  creationTimestamp: "2024-02-15T15:37:59Z"
  finalizers:
  - kubernetes.io/pvc-protection
  labels:
    app: example-mongodb-svc
  name: data-volume-example-mongodb-0
  namespace: mongo
  resourceVersion: "9319"
  uid: ####################################
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10G
  storageClassName: gp2
  volumeMode: Filesystem
  volumeName: pvc-example-logs
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  phase: Bound
```

* Data PersistentVolume
```
> kubectl get pv pv-data-example-mongodb -o yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/migrated-to: ebs.csi.aws.com
    pv.kubernetes.io/provisioned-by: kubernetes.io/aws-ebs
    volume.kubernetes.io/provisioner-deletion-secret-name: ""
    volume.kubernetes.io/provisioner-deletion-secret-namespace: ""
  creationTimestamp: "2024-02-15T15:38:03Z"
  finalizers:
  - kubernetes.io/pv-protection
  - external-attacher/ebs-csi-aws-com
  labels:
    topology.kubernetes.io/region: eu-west-1
    topology.kubernetes.io/zone: eu-west-1c
  name: pvc-####################################
  resourceVersion: "40549924"
  uid: ####################################
spec:
  accessModes:
  - ReadWriteOnce
  awsElasticBlockStore:
    fsType: ext4
    volumeID: vol-#################
  capacity:
    storage: 10Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: data-volume-example-mongodb-0
    namespace: mongo
    resourceVersion: "9274"
    uid: ####################################
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - eu-west-1c
        - key: topology.kubernetes.io/region
          operator: In
          values:
          - eu-west-1
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp2
  volumeMode: Filesystem
status:
  phase: Bound
```

* Logs PersistentVolumeClaim

```
> kubectl get pvc logs-volume-example-mongodb-0 -n mongo -o yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: ebs.csi.aws.com
    volume.kubernetes.io/selected-node: ##########################################
    volume.kubernetes.io/storage-provisioner: ebs.csi.aws.com
  creationTimestamp: "2024-02-15T15:37:59Z"
  finalizers:
  - kubernetes.io/pvc-protection
  labels:
    app: example-mongodb-svc
  name: logs-volume-example-mongodb-0
  namespace: mongo
  resourceVersion: "9322"
  uid: ####################################
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2G
  storageClassName: gp2
  volumeMode: Filesystem
  volumeName: pvc-####################################
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 2Gi
  phase: Bound
```

* Logs PersistentVolume

```
> kubectl get pv pv-data-example-mongodb -o yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/migrated-to: ebs.csi.aws.com
    pv.kubernetes.io/provisioned-by: kubernetes.io/aws-ebs
    volume.kubernetes.io/provisioner-deletion-secret-name: ""
    volume.kubernetes.io/provisioner-deletion-secret-namespace: ""
  creationTimestamp: "2024-02-15T15:38:03Z"
  finalizers:
  - kubernetes.io/pv-protection
  - external-attacher/ebs-csi-aws-com
  labels:
    topology.kubernetes.io/region: eu-west-1
    topology.kubernetes.io/zone: eu-west-1c
  name: pvc-####################################
  resourceVersion: "40550070"
  uid: ####################################
spec:
  accessModes:
  - ReadWriteOnce
  awsElasticBlockStore:
    fsType: ext4
    volumeID: vol-#################
  capacity:
    storage: 2Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: logs-volume-example-mongodb-0
    namespace: mongo
    resourceVersion: "9270"
    uid: ####################################
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - eu-west-1c
        - key: topology.kubernetes.io/region
          operator: In
          values:
          - eu-west-1
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp2
  volumeMode: Filesystem
status:
  phase: Bound
```

* StorageClass manifest

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"},"name":"gp2"},"parameters":{"fsType":"ext4","type":"gp2"},"provisioner":"kubernetes.io/aws-ebs","volumeBindingMode":"WaitForFirstConsumer"}
    storageclass.kubernetes.io/is-default-class: "false"
  creationTimestamp: "2024-02-15T15:17:10Z"
  name: gp2
  resourceVersion: "40387932"
  uid: ####################################
parameters:
  fsType: ext4
  type: gp2
provisioner: kubernetes.io/aws-ebs
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

## Downscale the MongoDB deployment

Next step should include stopping the MongoDB service to avoid inconsistencies while taking the snapshots on the EBS volumes.
In our scenario, the deployment was managed by FluxCD, so downscaling the ReplicaSet should be fine to stop the Pods and the components involved on accesing the volumes.
Edit the 'members:' to define a 0 replicas.

```
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: example-mongodb
  namespace: mongo
spec:
  members: 0
  type: ReplicaSet
  version: "6.0.5"
  security:
    authentication:
      modes: ["SCRAM-SHA-256"]
...
...
```

Changes should be commited and reconciled using FLuxCD.

```
❯ flux reconcile kustomization post-mongo --with-source
```

Verify that the Pods are "removed" from the cluster.

```
❯ kubectl get pods -n mongo
NAME                                           READY   STATUS    RESTARTS   AGE
mongodb-kubernetes-operator-6546d6fcfc-ktgf8   1/1     Running   0          71d
```

## Take snapshots from the EBS gp2 volumes

Afer the MongoDB deployment was stopped, taking snapshots from the EBS volumes involved 'data and logs' are mandatory.

Spapshots can be taken using the `aws cli` utility, configuration of the profiles and access keys are not covered in this post.

* Take a snapshot for the 'data' volume

```
❯ aws ec2 create-snapshot --volume-id vol-################# --region eu-west-1 --tag-specifications 'ResourceType=snapshot,Tags=[{Key="ec2:ResourceTag/ebs.csi.aws.com/cluster",Value="true"}]'
{
…
    "SnapshotId": "snap-###################
…
    "State": "pending",
    "VolumeId": "vol-###################
    "VolumeSize": 1,
…    
}
```
* Take a snapshot for the 'logs' volume

```
❯ aws ec2 create-snapshot --volume-id vol-################# --region eu-west-1 --tag-specifications 'ResourceType=snapshot,Tags=[{Key="ec2:ResourceTag/ebs.csi.aws.com/cluster",Value="true"}]'
{
…
    "SnapshotId": "snap-###################
…
    "State": "pending",
    "VolumeId": "vol-##################"
    "VolumeSize": 1,
…    
}
```
* Wait until the snapshots are in state 'completed'

* Verify that the snapshots are created properly.

```
❯ aws ec2 describe-snapshots --region eu-west-1 > snapshots

        {
            "Encrypted": true,
            "KmsKeyId": "###########################################################################",
            "OwnerId": "############",
            "Progress": "100%",
            "SnapshotId": "snap-[DATA_ID]",
            "StartTime": "2024-04-22T20:33:06.371000+00:00",
            "State": "completed",
            "VolumeId": "vol-[DATA_ID]",
            "VolumeSize": 10,
            "StorageTier": "standard"
        },
        {
            "Encrypted": true,
            "KmsKeyId": "###########################################################################",
            "OwnerId": "############",
            "Progress": "100%",
            "SnapshotId": "snap-[LOGS_ID]",
            "StartTime": "2024-04-22T20:38:53.416000+00:00",
            "State": "completed",
            "VolumeId": "vol-[LOGS_ID]",
            "VolumeSize": 2,
            "StorageTier": "standard"
        },

```

* Annotate the SnapshotId of each of them as they are needed later to recreate the new volumes.

##  Create new gp3 volumes from the previous snapshots

Now it's time to create newer EBS gp3 volumes from the previous snapshots, containg all the data.

Creating volumes can be done using the 'aws cli'.

* Create the new EBS gp3 volume for 'data'

```
❯ aws ec2 create-volume \
    --snapshot-id snap-[DATA_ID]
    --region eu-west-1
    --availability-zone eu-west-1a \
    --volume-type gp3 \
    --iops 3000 \
    --throughput 125 \
    --size 10
```

```
{
    "AvailabilityZone": "eu-west-1a",
    "CreateTime": "2024-04-26T23:58:06+00:00",
    "Encrypted": true,
    "KmsKeyId": "###########################################################################",
    "Size": 2,
    "SnapshotId": "snap-[DATA_ID]",
    "State": "creating",
    "VolumeId": "vol-[NEW_DATA_VOLUME_ID]",
    "Iops": 3000,
    "Tags": [],
    "VolumeType": "gp3",
    "MultiAttachEnabled": false,
    "Throughput": 125
}

```

* Create the new EBS gp3 volume for 'logs'

```
❯ aws ec2 create-volume \
    --snapshot-id snap-[LOGS_ID] \
    --region eu-west-1
    --availability-zone eu-west-1a \
    --volume-type gp3 \
    --iops 3000 \
    --throughput 125 \
    --size 2
```
```
{
    "AvailabilityZone": "eu-west-1a",
    "CreateTime": "2024-04-26T23:58:06+00:00",
    "Encrypted": true,
    "KmsKeyId": "###########################################################################",
    "Size": 2,
    "SnapshotId": "snap-[DATA_ID]",
    "State": "creating",
    "VolumeId": "vol-[NEW_DATA_VOLUME_ID]",
    "Iops": 3000,
    "Tags": [],
    "VolumeType": "gp3",
    "MultiAttachEnabled": false,
    "Throughput": 125
}

```

* Annotate the VolumeID as they will be used later to be referenced by the kubernetes PersistentVolumes.
* Remember that additional options like multi-attach can be configured using extra parameters.
* Follow the reference for crete-volume: https://docs.aws.amazon.com/cli/latest/reference/ec2/create-volume.html

##  Create a new gp3 type StorageClass and configure it as 'default'

With the volumes created and based on our scenario, a newer StorageClass for gp3 should be created.

* First, annotate the previous gp2 StorageClass to not be considered the default in the cluster. 

```
❯ kubectl annotate sc gp2 storageclass.kubernetes.io/is-default-class: "false"                                                             │
```

* And add a newer gp3 StorageClass that will be the default on the EKS cluster using kubectl or your gitops workflow

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: gp3
parameters:
  fsType: ext4
  type: gp3
provisioner: kubernetes.io/aws-ebs
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

* Verify that the new gp3 StorageClass was created and is the default one

```
❯ kubectl get sc
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2             kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  71d
gp3 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true                   7d13h
```

##  Delete the old PersistentVolumes

As new PersistentVolumes are going to be created, older ones are not longer required, and can be deleted from the cluster.

* Delete the 'data' PersistentVolume

```
❯ kubectl delete pv pv-data-example-mongodb
```

* Delete the 'logs' PersistentVolumes

```
❯ kubectl delete pv pv-logs-example-mongodb
```

##  Delete the old PersistentVolumeClaim

As new PersistentVolumes are going to be created, older ones are not longed requried as they will be recreated by the StatefulSet of the MongoDB Community deployment.

* Delete the 'data' PersistentVolumeClaim

```
❯ kubectl delete pvc data-volume-example-mongodb-0-n mongo
```

* Delete the 'logs' PersistentVolumeClaim

```
❯ kubectl delete pvc logs-volume-example-mongodb-0 -n mongo
```

## Create newer PersistentVolumes

Recreate the PersistentVolumes on the cluster.

Note that the manifests include the section awsElasticBlockStore, referencing the volumeID of the EBS volume that was created before.

Note that the manifests include the section claimRef, referencing a PersistentVolumesClaim that will be created later.

* Recreate the 'data' PersistentVolume applying the manifest by kubectl or by your gitops workflow

```
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/migrated-to: ebs.csi.aws.com
    pv.kubernetes.io/provisioned-by: kubernetes.io/aws-ebs
  labels:
    topology.kubernetes.io/region: eu-west-1
    topology.kubernetes.io/zone: eu-west-1c
  name: pv-data-example-mongodb
spec:
  storageClassName: gp3
  awsElasticBlockStore:
    fsType: ext4
    volumeID: vol-[NEW_DATA_VOLUME_ID]
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  claimRef:
    namespace: mongo
    name: data-volume-example-mongodb-0
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - eu-west-1c
        - key: topology.kubernetes.io/region
          operator: In
          values:
          - eu-west-1
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem

```

* Recreate the 'logs' PersistentVolume applying the manifest by kubectl or by your gitops workflow

```
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/migrated-to: ebs.csi.aws.com
    pv.kubernetes.io/provisioned-by: kubernetes.io/aws-ebs
  labels:
    topology.kubernetes.io/region: eu-west-1
    topology.kubernetes.io/zone: eu-west-1c
  name: pv-logs-example-mongodb
spec:
  storageClassName: gp3
  awsElasticBlockStore:
    fsType: ext4
    volumeID: vol-[NEW_LOGS_VOLUME_ID]
  capacity:
    storage: 2Gi
  accessModes:
  - ReadWriteOnce
  claimRef:
    namespace: mongo
    name: logs-volume-example-mongodb-0
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values:
          - eu-west-1c
        - key: topology.kubernetes.io/region
          operator: In
          values:
          - eu-west-1
  persistentVolumeReclaimPolicy: Retain
  volumeMode: Filesystem

```

* Verify that both volumes are created

```
❯ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                     STORAGECLASS   REASON   AGE
pv-data-example-mongodb                    11Gi       RWO            Retain           Available   mongo/data-volume-example-mongodb-0       gp3                     3d13h
pv-logs-example-mongodb                    2Gi        RWO            Retain           Available   mongo/logs-volume-example-mongodb-0       gp3                     3d13h
```

## Customize the MongoDB Community manifest to tweak the PersistentVolumesClaims

Having the new gp3 StorageClass and the PersistentVolumes pointing to the new EBS gp3 volumes, the next step is to ensure that the MongoDB Community StatefulSet creates the proper PersistentVolumesClaims configured for using the new gp3 StorageClass.

By default the MongoDB Community manifest creates a StatefulSet that defines two PersistentVolumesClaims called **data-volume** and **logs-volume** that later are bind mounted on the proper paths of the container (/data) and (/var/log/mongodb-mms-automation).

The names of the PersistentVolumesClaims **should be equivalent to the names defined on the section claimRef of the PersistentVolumes** that are created before (data-volume-example-mongodb-0) and (logs-volume-example-mongodb-0).

This previous references are not arbitrary, as the naming convention of the PersistentVolumesClaims created by a StatefulSet follows this syntax:

* {PersistentVolumesClaim name}-{StatefulSet name}-{ReplicaSet index}

This template can be configured on the MongoDBCommunity manifest, on the statefulSet section, to configure the PersistentVolumesClaims created, allowing to modify some parameters.

```
---
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: example-mongodb
  namespace: mongo
spec:
    ...
    ...
  statefulSet:
    spec:
      volumeClaimTemplates:
      - metadata:
          name: data-volume #data-volume-example-mongodb-0
        spec:
          accessModes: ["ReadWriteOnce"]
          storageClassName: "gp3"
          resources:
            requests:
              storage: 10Gi
      - metadata:
          name: logs-volume #logs-volume-example-mongodb-0
        spec:
          accessModes: ["ReadWriteOnce"]
          storageClassName: "gp3"
          resources:
            requests:
              storage: 2Gi
```

When the manifest will be applied, it will create two PersistentVolumesClaims (data-volume-example-mongodb-0) and (logs-volume-example-mongodb-0), those PersistentVolumesClaims are refered by the PersistentVolumes created before, so it will be used and mounted on the proper paths of the MongoDBCommunity containers.

##  Rescale the MongoDB deployment to verify that the volumes are properly mounted on the pods

When the changes are performed on the MongoDBCommunity manifest, rescale the ReplicaSet parameter to spin up newer Pods containing the MongoDB StatefulSet components.

```
---
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: example-mongodb
  namespace: mongo
spec:
  members: 1
  type: ReplicaSet
```

* Apply this change by kubectl or using your gitops workflow.
* Verify that the Pods are up and healthy

```
❯ kubectl get pods -n mongo
NAME                                           READY   STATUS    RESTARTS   AGE
mongodb-kubernetes-operator-6546d6fcfc-ktgf8   1/1     Running   0          71d
example-mongodb-0                              2/2     Running   0          34h
```

* Verify that the volumes are properly mounted

```
❯ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                     STORAGECLASS   REASON   AGE
pv-data-example-mongodb                    10Gi       RWO            Retain           Bound       mongo/data-volume-example-mongodb-0       gp3                     3d14h
pv-logs-example-mongodb                    2Gi        RWO            Retain           Bound       mongo/logs-volume-example-mongodb-0       gp3                     3d14h
```

* Verify the volumeAttachements on the EKS cluster

```
❯ kubectl get volumeattachments.storage.k8s.io -A
NAME                                                                   ATTACHER          PV                                         NODE                                         ATTACHED   AGE
csi-################################################################   ebs.csi.aws.com   pv-logs-example-mongodb                    ###############.eu-west-1.compute.internal   true       34h
csi-################################################################   ebs.csi.aws.com   pv-data-example-mongodb                    ###############.eu-west-1.compute.internal   true       34h
```

* Verify that the volumes are mounted in the Pods

```
❯ kubectl exec -it example-mongodb-0 -n mongo /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
root@example-mongodb-0:/$ ls /data/
WiredTiger                             collection-50--6481199116760641662.wt  index-33--6481199116760641662.wt
WiredTiger.lock                        collection-52--6481199116760641662.wt  index-34--6481199116760641662.wt
WiredTiger.turtle                      collection-54--6481199116760641662.wt  index-36--6481199116760641662.wt
WiredTiger.wt                          collection-56--6481199116760641662.wt  index-37--6481199116760641662.wt
WiredTigerHS.wt                        collection-58--6481199116760641662.wt  index-39--6481199116760641662.wt
_mdb_catalog.wt                        collection-6--6481199116760641662.wt   index-41--6481199116760641662.wt
automation-mongod.conf                 collection-60--6481199116760641662.wt  index-42--6481199116760641662.wt
collection-0--6481199116760641662.wt   collection-62--6481199116760641662.wt  index-44--6481199116760641662.wt
collection-0-6663422210246180860.wt    collection-64--6481199116760641662.wt  index-45--6481199116760641662.wt
collection-10--6481199116760641662.wt  collection-8--6481199116760641662.wt   index-47--6481199116760641662.wt
collection-12--6481199116760641662.wt  configdb                               index-5--6481199116760641662.wt
collection-14--6481199116760641662.wt  db                                     index-51--6481199116760641662.wt
collection-15--6481199116760641662.wt  diagnostic.data                        index-53--6481199116760641662.wt
collection-17--6481199116760641662.wt  index-0--1765585522502302116.wt        index-55--6481199116760641662.wt
collection-19--6481199116760641662.wt  index-1--6481199116760641662.wt        index-57--6481199116760641662.wt
collection-2--6481199116760641662.wt   index-1-6663422210246180860.wt         index-59--6481199116760641662.wt
collection-22--6481199116760641662.wt  index-11--6481199116760641662.wt       index-61--6481199116760641662.wt
collection-24--6481199116760641662.wt  index-13--6481199116760641662.wt       index-63--6481199116760641662.wt
collection-26--6481199116760641662.wt  index-16--6481199116760641662.wt       index-65--6481199116760641662.wt
collection-27--6481199116760641662.wt  index-18--6481199116760641662.wt       index-7--6481199116760641662.wt
collection-28--6481199116760641662.wt  index-20--6481199116760641662.wt       index-9--6481199116760641662.wt
collection-29--6481199116760641662.wt  index-21--6481199116760641662.wt       journal
collection-35--6481199116760641662.wt  index-23--6481199116760641662.wt       lost+found
collection-38--6481199116760641662.wt  index-25--6481199116760641662.wt       mongod.lock
collection-4--6481199116760641662.wt   index-3--6481199116760641662.wt        sizeStorer.wt
collection-40--6481199116760641662.wt  index-30--6481199116760641662.wt       storage.bson
collection-43--6481199116760641662.wt  index-31--6481199116760641662.wt
collection-46--6481199116760641662.wt  index-32--6481199116760641662.wt
```

## References

- https://aws.amazon.com/blogs/containers/migrating-amazon-eks-clusters-from-gp2-to-gp3-ebs-volumes/
- https://www.youtube.com/watch?v=ajhDlm5Yvpc
- https://loft.sh/blog/kubernetes-statefulset-examples-and-best-practices/
- https://medium.com/@kevken1000/eks-attaching-an-existing-volume-to-a-statefulset-b6d1ea9d2491
- https://github.com/mongodb/mongodb-kubernetes-operator/blob/master/config/samples/arbitrary_statefulset_configuration/mongodb.com_v1_custom_volume_cr.yaml
- https://github.com/mongodb/mongodb-kubernetes-operator/blob/master/config/samples/mongodb.com_v1_mongodbcommunity_cr.yaml
- https://wangwei1237.github.io/Kubernetes-in-Action-Second-Edition/docs/Managing_stateful_applications_with_Kubernetes_Operators.html
- https://cloud.google.com/kubernetes-engine/docs/how-to/persistent-volumes/preexisting-pd#pv_to_statefulset
- https://www.alibabacloud.com/help/en/ack/ack-managed-and-ack-dedicated/user-guide/enable-a-statefulset-to-support-persistent-storage
- https://www.eksworkshop.com/docs/fundamentals/storage/ebs/statefulset-with-ebs/
