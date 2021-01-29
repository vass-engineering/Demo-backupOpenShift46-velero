# Velero integration with OpenShift 4.6 and OCS.

* Velero is an open source tool to safely backup and restore, perform disaster recovery, and migrate Kubernetes cluster resources and persistent volumes.
* We will use a bucket S3 from Noobaa for Object Store, using AWS Velero provider, and Container Storage Interface (CSI) Velero provider for Volume Snapshotter.


## Index

1. Requirements
2. Create a bucket S3 in Noobaa for Velero
3. Check Volume Snapshot Classes for CSI provider
4. Install Velero integrated with AWS provider and CSI-provider.
5. Deploy a WordPress using MariaDB as the database with a persistent volume as an example.
6. Apply an example Theme to the wordpress.
7. Make a backup of our WordPress project
8. Delete and recover our WordPress project


## 1) Requirements:

* OpenShift 4.6 Installed.
* OpenShift Container Storage.

## 2)Create a bucket S3 in Noobaa for Velero

Access to Noobaa console and login with your OCP credentials

```
 oc get routes -n openshift-storage
NAME          HOST/PORT                                                PATH   SERVICES      PORT         TERMINATION          WILDCARD
noobaa-mgmt   noobaa-mgmt-openshift-storage.apps.pilabs.labs.vass.es          noobaa-mgmt   mgmt-https   reencrypt/Redirect   None
s3            s3-openshift-storage.apps.pilabs.labs.vass.es                   s3            s3-https     reencrypt            None
```

From the console, create a bucket, apply a policy and a region.

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/noobaaConsole.png)


![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/CreateBucket.png)

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/CreateBucketPolicy.png =250x250)

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/AssignRegionToBucket.png)


## 3)  Check Volume Snapshot Classes for CSI provider

You can think about "SnapShots" in K8S as persistent volumes and persistent volumes claims, where the Volume Snapshots is the PVC and the  Volume Snapshot Contents is the PV. When you create a Snapshot, you will use a VolumeSnapshotClass, from where you will configure the "Deletion Policy". The Volume Snapshots is not a cluster-wide object, so when you will delete the namespace, you will delete the Volume Snapshots and if the "Deletion Policy" of the  VolumeSnapshotClass is "Deleted" you will delete the Volume Snapshot Contents where the "data" is keep it. So in order to keep the data of the backup, be sure that the "Deletion Policy" of the VolumeSnapshotClass is retain.

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/VolumeSnapshoClasses.png)

```
oc get VolumeSnapshotClass
NAME                                        DRIVER                                  DELETIONPOLICY   AGE
ocs-storagecluster-cephfsplugin-snapclass   openshift-storage.cephfs.csi.ceph.com   Retain           14d
ocs-storagecluster-rbdplugin-snapclass      openshift-storage.rbd.csi.ceph.com      Retain           14d
```

You can change the policy from the console or using the cli.

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/snapclassdeletionPolicyDelete.png)

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/snapclassdeletionPolicyRetain.png)



## 4)Install Velero integrated with AWS provider and CSI-provider

#### Obtain Noobaa bucket connections and create credentials file.

* From Noobaa console, obtain the connection details for your Bucket:

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/ConnectApplication.png)


* Create a file "credentials-velero" with the S3(Noobaa) credential access.

```
vi credentials-velero
```

```
[default]
aws_access_key_id = 8Vv12B5LdnlB1w8xqACC
aws_secret_access_key = m0scKSLvhv90UIxuKxeHwq5jGr8NTFOkWSI4F8iN
```

* The Cluster External Name (The URL Connection) by default is a connection secure https, but OpenShift will create and mount an auto signed certificate, and Velero will crash due the auto signed certificate. You can mount a valid signed certificate or just create a plain Route http for demo purpose or  internal  communications.

```
oc expose service s3 --name=s3-insecure --port=s3  --hostname=s3-insecure-openshift-storage.apps.pilabs.labs.vass.es
```

#### Install Velero with the infomation obtained.

* Change <> with your environment details:

```
velero install --provider <provider>  --plugins <plugins,...> --bucket <bucket-name> --secret-file <secret-file> --backup-location-config  region=<region>,s3ForcePathStyle="true",s3Url=<S3-URL>  --use-volume-snapshots=true  --image velero/velero:v1.4.0 --snapshot-location-config region="default" --features=EnableCSI
```

* Example:

```
velero install --provider aws  --plugins velero/velero-plugin-for-aws:v1.0.1,velero/velero-plugin-for-csi:v0.1.0 --bucket backupocp46 --secret-file ./credentials-velero --backup-location-config  region=vass,s3ForcePathStyle="true",s3Url=http://s3-insecure-openshift-storage.apps.pilabs.labs.vass.es  --use-volume-snapshots=true  --image velero/velero:v1.4.0 --snapshot-location-config region="default" --features=EnableCSI
```


## 5) Deploy a WordPress using MariaDB as the database with a persistent volume as an example.

* Create a new project for wordpress

```
oc new-project wordpress
```

* Deploy MariaDB usint OpenShift templates

```
oc get templates -n openshift | grep mariadb
```

```
oc process --parameters -n openshift mariadb-persistent
NAME                    DESCRIPTION                                                               GENERATOR           VALUE
MEMORY_LIMIT            Maximum amount of memory the container can use.                                               512Mi
NAMESPACE               The OpenShift Namespace where the ImageStream resides.                                        openshift
DATABASE_SERVICE_NAME   The name of the OpenShift Service exposed for the database.                                   mariadb
MYSQL_USER              Username for MariaDB user that will be used for accessing the database.   expression          user[A-Z0-9]{3}
MYSQL_PASSWORD          Password for the MariaDB connection user.                                 expression          [a-zA-Z0-9]{16}
MYSQL_ROOT_PASSWORD     Password for the MariaDB root user.                                       expression          [a-zA-Z0-9]{16}
MYSQL_DATABASE          Name of the MariaDB database accessed.                                                        sampledb
MARIADB_VERSION         Version of MariaDB image to be used (10.3-el7, 10.3-el8, or latest).                          10.3-el8
VOLUME_CAPACITY         Volume space available for data, e.g. 512Mi, 2Gi.                                             1Gi
```

```
oc process --parameters -n openshift mariadb-persistent  | awk '{if (NR!=1) {print $1}}' >  mariadbParameter.env
```
```
vi mariadbParameter.env
```

* Add the next info as an example:

```
MEMORY_LIMIT=512Mi
NAMESPACE=openshift
DATABASE_SERVICE_NAME=mariadb
MYSQL_USER=test
MYSQL_PASSWORD=test
MYSQL_ROOT_PASSWORD=test
MYSQL_DATABASE=test
MARIADB_VERSION=10.3-el8
VOLUME_CAPACITY=1Gi
```


```
oc process  mariadb-persistent  -n openshift  --param-file=mariadbParameter.env -o yaml > mariadb-persistentTemplate.yaml
```

```
oc create -f mariadb-persistentTemplate.yaml

oc get pods
NAME               READY   STATUS      RESTARTS   AGE
mariadb-1-55ttz    1/1     Running     1          2m57s
mariadb-1-deploy   0/1     Completed   0          3m1s

```

* Deploy WordPress using OpenShift templates from github

```
oc new-app php~https://github.com/wordpress/wordpress
```

```
oc get pods
NAME                        READY   STATUS      RESTARTS   AGE
mariadb-1-55ttz             1/1     Running     1          12m
mariadb-1-deploy            0/1     Completed   0          12m
wordpress-1-build           0/1     Completed   0          8m42s
wordpress-5c87864d4-f7r2d   1/1     Running     0          4m24s
```

```
oc get svc
NAME        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
mariadb     ClusterIP   172.30.220.148   <none>        3306/TCP            4m27s
wordpress   ClusterIP   172.30.100.148   <none>        8080/TCP,8443/TCP   61s
```

```
oc expose svc wordpress
route.route.openshift.io/wordpress exposed
```

```
 oc get routes          
NAME        HOST/PORT                                      PATH   SERVICES    PORT       TERMINATION   WILDCARD
wordpress   wordpress-wordpress.apps.pilabs.labs.vass.es          wordpress   8080-tcp                 None
```

* Install WordPress

Access to WordPress console and install the site.


![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/InstallWordPress.png)

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/InstallWP.png)

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/RunInstallationWP.png)


## 6)Apply an example Theme to the wordpress.

* Once WP is installed, just make a custom change. For example apply a theme.

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/ActiveThemeWP.png)

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/CheckyourWP.png)



## 7)Make a backup of our WordPress project


* Create the Backup

```
velero backup create wordpressbck  --snapshot-volumes=true  --include-namespaces  wordpress --wait
```

* Check that the status backup

```
Backup request "wordpressbck" submitted successfully.
Waiting for backup to complete. You may safely press ctrl-c to stop waiting - your backup will continue in the background.
...........................
Backup completed with status: Completed. You may check for more information using the commands `velero backup describe wordpressbck` and `velero backup logs wordpressbck`.
[root@bastion Velero]# velero get backups
NAME           STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
wordpressbck   Completed   0        0          2021-01-29 10:20:45 +0100 CET   29d       default            <none>
```

## 8) Delete and recover our WordPress project

* Delete the project/namespaces wordpress.

```
oc delete project wordpress
```

* Check that the project has been remove.

```
oc get all -n wordpress
No resources found in wordpress namespace.
```

* Recover from your backup.

```
velero get backups

NAME           STATUS      ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
wordpressbck   Completed   0        0          2021-01-29 10:20:45 +0100 CET   29d       default            <none>
```

```
velero restore create --from-backup  wordpressbck

Restore request "wordpressbck-20210129102324" submitted successfully.
Run `velero restore describe wordpressbck-20210129102324` or `velero restore logs wordpressbck-20210129102324` for more details.
```

* Check  the restore

```
velero get restores

NAME                          BACKUP         STATUS            STARTED   COMPLETED   ERRORS   WARNINGS   CREATED                         SELECTOR
wordpressbck-20210129102324   wordpressbck   PartiallyFailed   <nil>     <nil>       1        1          2021-01-29 10:23:46 +0100 CET   <none>
```

* Check the pods and objets from your project.


```
oc get pods

NAME                         READY   STATUS     RESTARTS   AGE
mariadb-1-vg76x              1/1     Running    0          57s
wordpress-1-build            0/1     Init:0/2   0          55s
wordpress-5bcc896d8d-854th   1/1     Running    0          56s

oc get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
mariadb   Bound    pvc-93189147-46ad-4458-9530-036822d5e229   20Gi       RWO            ocs-storagecluster-ceph-rbd   8h

...

```

* Check your database to check that we have recover all the informacion:

```
oc rsh mariadb-1-vg76x  
sh-4.4$ mysql
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 6519
Server version: 10.3.27-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> use test;
Database changed
MariaDB [test]> show tables;
+---------------------------+
| Tables_in_test            |
+---------------------------+
| wp_testcommentmeta        |
| wp_testcomments           |
| wp_testlinks              |
| wp_testoptions            |
| wp_testpostmeta           |
| wp_testposts              |
| wp_testterm_relationships |
| wp_testterm_taxonomy      |
| wp_testtermmeta           |
| wp_testterms              |
| wp_testusermeta           |
| wp_testusers              |
+---------------------------+
12 rows in set (0.001 sec)
```

* Check your WebSite.

![alt text](https://github.com/vass-engineering/Demo-backupOpenShift46-velero/blob/main/DocsImages/CheckyourWP.png)
