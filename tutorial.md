This tutorial shows you how to [dynamically provision](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning) 
Kubernetes storage volumes in Google Kubernetes Engine 
from [Cloud Filestore](https://cloud.google.com/filestore/) using the 
[Kubernetes NFS-Client Provisioner](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client).

[![button](http://gstatic.com/cloudssh/images/open-btn.png)](https://console.cloud.google.com/cloudshell/open?git_repo=https://github.com/wardharold/gke-filestore-dynamic-provisioning&page=editor&tutorial=tutorial.md)

## Task 0 (OPTIONAL) Create a project with a billing account attached 
**(you can also use an existing project and skip to the next step)**
```sh
ORG=[YOUR_ORG]
BILLING_ACCOUNT=[YOUR_BILLING_ACCOUNT_NAME]
PROJECT=[NAME FOR THE PROJECT YOU WILL CREATE]
ZONE=[COMPUTE ZONE YOU WANT TO USE]
gcloud projects create $PROJECT --organization=$ORG
gcloud beta billing projects link $PROJECT --billing-account=$(gcloud beta billing accounts list | grep $BILLING_ACCOUNT | awk '{print $1}')
gcloud config configurations create -- activate $PROJECT
gcloud config set compute/zone $ZONE
```
## Task 1 Enable the required Google APIs
```sh
gcloud services enable file.googleapis.com
```

## Task 2 Create a Cloud Filestore volume
1. Create a Cloud Filestore instance with 1TB of stroage capacity
```sh
FS=[NAME FOR THE FILESTORE YOU WILL CREATE]
gcloud beta filestore instances create ${FS} \
    --project=${PROJECT} \
    --location=${ZONE} \
    --tier=STANDARD \
    --file-share=name="volumes",capacity=1TB \
    --network=name="default",reserved-ip-range="10.0.0.0/29"
```
2. Retrieve the IP address of the Cloud Filestore instance
```sh
FSADDR=$(gcloud beta filestore instances describe ${FS} \
             --project=${PROJECT} \
             --location=${ZONE} \
             --format="value(networks.ipAddresses[0])")
```

## Task 3 Create a Kubernetes Engine cluster
1. Create the cluster and get its credentials
```sh
CLUSTER=[NAME OF THE KUBERNETES CLUSTER YOU WILL CREATE]
gcloud container clusters create ${CLUSTER}
gcloud container clusters get-credentials ${CLUSTER}
```
2. Grant yourself cluster-admin privileges
```sh
ACCOUNT=$(gcloud config get-value core/account)
kubectl create clusterrolebinding core-cluster-admin-binding \
    --user ${ACCOUNT} \
    --clusterrole cluster-admin
```
3. Install [Helm](https://github.com/helm/helm)
Follow the instructions found [here](https://docs.helm.sh/using_helm/#installing-helm). *Note*: since this is an 
RBAC enabled cluster you will need to follow [these](https://docs.helm.sh/using_helm/#role-based-access-control)
steps when initializing Tiller.

## Task 4 Deploy the NFS-Client Provisioner
```sh
helm install stable/nfs-client-provisioner --name nfs-cp --set nfs.server=${FSADDR} --set nfs.path=/volumes
watch kubectl get po -l app=nfs-client-provisioner
```
Press Ctrl-C when the provisioner pod's status changes to Running

## Task 5 Make a Persistent Volume Claim
While you can use any application that uses storage classes to do [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
to test the nfs-client provisioner, in this tutorial you will deploy a [PostgreSQL](https://www.postgresql.org/) instance verify
the configuration.
```sh
helm install --name postgresql --set persistence.storageClass=nfs-client stable/postgresql
watch kubectl get po -l app=postgresql
```
Press Ctrl-C when the database pod's status changes to Running.

The PostgreSQL Helm chart creates an 8Gb persistent volume claim on Cloud Filestore and mounts it at ```/var/lib/postgresql/data/pgdata```
in the database pod. In the next task you will create a small VM and verify that the database actually created its files in
Cloud Filestore.

## Task 6 Verify Cloud Filestore volume directory creation
1. Create an ```f1-micro`` Compute Engine instance
```sh
gcloud compute --project=${PROJECT} instances create check-nfs-provisioner \
    --zone=us-central1-c \
    --machine-type=f1-micro \
    --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append \
    --image=debian-9-stretch-v20180911 \
    --image-project=debian-cloud \
    --boot-disk-size=10GB \
    --boot-disk-type=pd-standard \
    --boot-disk-device-name=check-nfs-provisioner
```
2. Log into the check-nfs-provisioner instance
```sh
gcloud compute ssh check-nfs-provisioner
```
3. Install the nfs-common package on check-nfs-provisioner
```sh
sudo apt update -y
sudo apt install nfs-common -y
```
4. Mount the Cloud Filestore volume on check-nfs-provisioner
```sh
sudo mkdir /mnt/gke-volumes
sudo mount [FILESTORE IP ADDRESS]:/volumes /mnt/gke-volumes
```
5. Display the PostgreSQL database directory structure
```sh
sudo find /mnt/gke-volumes -type d
```
You will see output like the following modulo the name of the PostgreSQL pod
```sh
/mnt/gke-volumes
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_twophase
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_notify
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_clog
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_stat_tmp
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_stat
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_serial
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/global
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_commit_ts
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_dynshmem
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_snapshots
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_replslot
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_logical
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_logical/snapshots
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_logical/mappings
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_tblspc
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_subtrans
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/base
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/base/1
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/base/12406
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/base/12407
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_xlog
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_xlog/archive_status
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_multixact
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_multixact/offsets
/mnt/gke-volumes/default-nfs-postgres-postgresql-pvc-f739e9a1-c032-11e8-9994-42010af00013/postgresql-db/pg_multixact/members
```
6. Log out of check-nfs-provisioner by pressing Ctrl-D

## Task 7 Clean up
1. Delete the check-nfs-provisioner instance
```sh
gcloud compute instances delete check-nfs-provisioner
```
2. Delete the Kubernetes Engine cluster
```sh
helm destroy postgresql
helm destroy nfs-cp
gcloud container clusters delete ${CLUSTER}
```
3. Delete the Cloud Filestore instance
```sh
gcloud beta filestore instances delete ${FS} --location ${ZONE}
```
