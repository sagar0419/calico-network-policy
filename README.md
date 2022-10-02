# Backup, Restore and Migrate Kubernetes Cluster resources using Velero.
## Introduction
Kubernetes has become the de-facto choice of every organisation for dealing with microservices. It is used by organisations for mission-critical applications and there is a critical need for backing up the application and its associated data. 
Backup and restoration of data is a very crucial part. You can quickly recover from any disaster if you have backups. In this blog post, we will talk about the backup and restoration of Kubernetes resources using Velero.
Velero is an open-source tool for the backup and recovery of resources running in the Kubernetes cluster. Velero uses Kubernetes API discovery capabilities to collect the data to be backed up. Velero can perform manual backup and automated backups, restore, disaster recovery, migrating resources and PV from one cluster to another Kubernetes cluster.
In this tutorial, We will guide you to take a backup of resources running in the source Kubernetes cluster and we will also guide you to migrate that backup to the target cluster.
Prerequisite
 - 2 Kubernetes cluster up and running, as we are explaining migration also in this tutorial.
 - You are able to run the kubectl command against both Kubernetes clusters.
 - An AWS IAM user with an access key and secret key and permission to put and get objects in the AWS S3 bucket.
 - Create a bucket on AWS S3 to store the backup.
 - Storage Provider for our workload.
 - Before the migration, clear the abnormal pod resources in the source cluster. If the pod is in an abnormal state and has a PVC mounted, the PVC will remain in the pending state after the cluster is migrated.

For this tutorial, We have created 2 Kubernetes clusters (Source and Target). On the Source cluster, our WordPress application is running and on the target cluster, we will move and restore the backup of the WordPress application.

## Velero Installation: 
Velero uses a backup location to perform backup and restore your cluster. The backup location can be an AWS S3, Azure Blob storage or any S3 compatible storage. For this tutorial, We are going to use AWS S3 to back up and restore cluster data.

Create a file with the name `"credentials"` in your present working directory and paste the AWS access key and secret in that file. Make sure you use those credentials which have access to the AWS S3 as mentioned in the prerequisite.

```bash
[default]
aws_access_key_id = AKXXXXXXXXXXXXXXXT
aws_secret_access_key = bXXXXXXXXXXXXXXXXXXXXz
```
Now our credentials file is created, let's install Velero CLI on our local machine. First, we need to download the tar file of Velero.

``` bash wget https://github.com/vmware-tanzu/velero/releases/download/v1.9.2/velero-v1.9.2-linux-amd64.tar.gz``` 

After running the above command successfully the Velero gets downloaded to your present directory. As you can see the downloaded file is in tar format let's untar the tar file.

``` bash tar -xvf velero-v1.9.2-linux-amd64.tar.gz ```

Once your file is untared move the Velero to the "/usr/local/bin/" directory.

``` bash sudo mv velero-v1.9.2-linux-amd64/velero /usr/local/bin/```

To check if the velero version, run this command.

``` bash velero version - client-only ```

You will get an output like this.

```bash 
Client:
 Version: v1.9.2
 Git commit: 82a100981cc66d119cf9b1d121f45c5c9dcf99e1 
```
Now, velero is installed on our local machine. Let's deploy velero in our Kubernetes cluster, and for that run the following command.

``` bash
velero install \
 - provider aws \
 - plugins velero/velero-plugin-for-aws:v1.4.1 \
 - bucket sagar-backup-pv \
 - secret-file ./credentials \
 - use-volume-snapshots=false \
 - use-restic \
 - backup-location-config region=us-west-2
```
In the above command `"sagar-backup-pv"` is the name of the AWS S3 bucket which we have created for this tutorial, `"credentials"` is the name of the file in which we have stored the AWS credentials to access the AWS S3 bucket and `"us-west-2"` is the AWS bucket location. This command will install all the velero components on the Kubernetes cluster.

`Make sure that you run this command against both of your clusters.`

##Annotations:

If you need to back up the data of storage volume in the pod, add an annotation to the pod. The annotation template is as follows:

```bash kubectl annotate <pod/pod_name> backup.velero.io/backup-volumes=<volume_name>  -n <namespace> ```

If you have multiple volumes attached to the pod you can add annotation like this and only those volumes names specified in the command will be taken for backup.

```bash kubectl -n <namespace> annotate <pod/pod_name> backup.velero.io/backup-volumes=<volume_name_1>,<volume_name_2>,… ```

- <namespace>: namespace where the pod is located.
- <pod_name>: pod name.
- <volume_name>: name of the persistent volume mounted to the pod.

You can run the describe statement to query the pod information. The Volume field indicates the names of all persistent volumes attached to the pod.
You can check the volume name by running the pod describe command.

 ``` bash kubectl  describe <pod/pod_name> -n <namespace> ```

Backing up the application in the source cluster:

We have deployed WordPress as a demo application on our source Kubernetes cluster. We have used the bitnami helm chart for this deployment.
Screenshot of the application running on the source Kubernetes cluster.As you can see in the screenshot we are using the longhorn storage on our source cluster. 

Screenshot of storage class used in source Kubernetes cluster. We will migrate the PVC from longhorn storage to Rook CEPH storage.

We have made a few changes to the WordPress site which are shown in this screenshot.
 
# Note: - You can Ignore this step if you are using the same storage solution on the source and the target Kubernetes clusters.
In our case, We are migrating the application with PVC using storage class longhorn on the source cluster to the target cluster where we have rook-cephfs installed.
 
There are 2 ways to do this:-
 - By changing the name of the storage class.
 - By adding configmap on the target cluster.

Both ways are explained below you can choose any of them.

 ### Changing the name of the storage class.

In this approach, we need to create the storage class with the name of the storage solution which is available in the source cluster.
In the below example, we have created a storage class manifest for Rook-ceph and under the metadata section, we named it longhorn.

 ```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn
# Change "rook-ceph" provisioner prefix to match the operator namespace if needed
provisioner: rook-ceph.cephfs.csi.ceph.com # driver:namespace:operator
parameters:
  # clusterID is the namespace where the rook cluster is running
  # If you change this namespace, also change the namespace below where the secret namespaces are defined
  clusterID: rook-ceph # namespace:cluster

  # CephFS filesystem name into which the volume shall be created
  fsName: k8sfs

  # Ceph pool into which the volume shall be created
  # Required for provisionVolume: "true"
  pool: k8sfs-replicated

  # The secrets contain Ceph admin credentials. These are generated automatically by the operator
  # in the same namespace as the cluster.
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph # namespace:cluster
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph # namespace:cluster

  # (optional) The driver can use either ceph-fuse (fuse) or ceph kernel client (kernel)
  # If omitted, default volume mounter will be used - this is determined by probing for ceph-fuse
  # or by setting the default mounter explicitly via --volumemounter command-line argument.
  # mounter: kernel
reclaimPolicy: Delete
allowVolumeExpansion: true
mountOptions:
  # uncomment the following line for debugging
  #- debug
```
 
` Please note that we have only changed the name of the storage class.`

Create a storage class manifest and apply that manifest to the target cluster. Once done you can verify that the storage class is created or not by running the following command.

 ``` bash kubectl get storageclass ```
 

The other way to change the storage class is, you just have to create the configmap on the target cluster under the Velero namespace. Then velero will change the storage class of the PVC during the migration process. Below is the configmap manifest which you can use to change the name of the storage class during migration.

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  # any name can be used; Velero uses the labels (below)
  # to identify it rather than the name
  name: change-storage-class-config
  # must be in the velero namespace
  namespace: velero
  # the below labels should be used verbatim in your
  # ConfigMap.
  labels:
    # this value-less label identifies the ConfigMap as
    # config for a plugin (i.e. the built-in restore item action plugin)
    velero.io/plugin-config: ""
    # this label identifies the name and kind of plugin
    # that this ConfigMap is for.
    velero.io/change-storage-class: RestoreItemAction
data:
  # add 1+ key-value pairs here, where the key is the old
  # storage class name and the value is the new storage
  # class name.
  # <old-storage-class>: <new-storage-class>
  longhorn: rook-cephfs
```
 
In the above code, We have mentioned the old storage class name longhorn and the new storage class name rook-cephfs.
You can create the configmap using the command mentioned below: -

```bash kubectl apply -f <name-of-configmap> ```
 
Create the configmap on the target Kubernetes cluster. Once it is deployed we will move to the next step.

For this tutorial, we have used the second approach we have created the configmap in the velero namespace on the target kubernetes cluster.

Assuming that, You have completed all the required steps mentioned in this document and you are ready to take the backup and migrate the application.
Now let's back up this deployment. Use the below-mentioned command to take the backup.

Run these backup commands on the source Kubernetes cluster.

`velero backup create <backup-name> - include-namespaces <namespace>`

You can check the status of the backup by running the following command.
`velero backup describe <backup-name>`

Assuming that your backup is taken now let's move to the target cluster.

### Restore

Before restoring the backup verify that you can access the backup which is stored on S3. Use the following command to check.

Run these backup commands on the target Kubernetes cluster.

 `velero backup get`
The output of the above command will be: -

To restore the backup on the target Kubernetes cluster run the below-mentioned command on the target cluster.

 ```bash velero restore create --from-backup <backup-name>--include-namespaces demo```

You can check the status of the backup by running the command.

 ```bash velero restore describe <backup-name> ```

Once the restore is completed you can check the deployment on the target Kubernetes cluster.

### Automating Backups
We can also automate the backup of our cluster resources. So that we can recover data in case of any disaster.
For automating backup jobs we use corn expression. This schedule operation allows us to create a backup of our data at the time mentioned in the cron expression.
 
```bash velero schedule create NAME - schedule="* * * * *" [flags] ```

 ```bash
Cron schedule uses the format mentioned below:
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday;
# │ │ │ │ │                                   7 is also Sunday on some systems)
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```
 
for example:
```bash velero schedule create demo-schedule - schedule="0 9* * *" ```

This command will create the backup, demo-schedule, within Velero, but the backup will not be taken until the next scheduled time, 9 AM. Backups created by a schedule are saved with the name <SCHEDULE NAME>-<TIMESTAMP>, where <SCHEDULE NAME> is the name which we mentioned in the command and <TIMESTAMP> is formatted as YYYYMMDDhhmmss. 

To check the full list of available configurations, flags use the Velero CLI help command.

 ```yaml
velero schedule create --help
```
