# Backup & Restore of MongoDB Database


## Prerequisite
- A mongodb replicaSet deployed in the cluster. If you don't have one, Please follow the first chapter to deploy one.
- Access to a s3 compatible bucket storage. We can use minio s3 bucket deploying in our cluster. 
  - To install minio, we can run the following helm command.
      ```shell
      $ helm repo add minio https://charts.min.io/ 
      $ helm repo update
      $ helm search repo minio/minio
        NAME       	CHART VERSION	APP VERSION                 	DESCRIPTION               
        minio/minio	5.0.8        	RELEASE.2023-04-13T03-08-07Z	Multi-Cloud Object Storage
      $ helm install minio minio/minio \
        --version 5.0.8 \
        --namespace minio --create-namespace \
        --set resources.requests.memory=512Mi \
        --set replicas=1 --set persistence.enabled=false \
        --set mode=standalone \
        --set rootUser=rootuser,rootPassword=rootpass123 \
        --set buckets[0].name=mongo
      ```
  - Now, we can port-forward the minio console to generate Access Key and Secret Key.
      ```shell
      $ kubectl port-forward svc/minio-console -n minio 9001
      ``` 
  - Let's visit the minio console via http://localhost:9001 and generate an Access Key and Secret Key.
  - After generating the Access Key and Secret key, we have to use this key in our storage secret.

## Backup
Now, we'll backup the database using Stash.

First, We'll insert some documents in a collection, so that we can backup those documents and restore them later.

```shell
$ kubectl exec -n demo mg-rs-2 -it -- bash

# You'll be connected to the pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD

# You'll be connected to the mongo shell
$ use kubecon
```

Now, we can run the following script in the mongo shell to insert 100 documents in a collection.

```javascript
for(var i = 1; i <= 100; i++ ) {
    db.participants.insert({
            "id": i,
            "name": "Participant "+ i ,
    })
}
```

Now, we can run the following command in the mongo shell to check if the documents were inserted successfully.

```shell
$ db.participants.find()
$ db.participants.count()
```

Now, Let's setup our backup.

First, We'll need a Repository to point our bucket. To apply the repository, we'll need a secret that has the s3 credentials. 
We can use the credentials that we have generated from the minio console and replace the respective keys in the `storage-secret.yaml`.

```shell
$ kubectl apply -f ./ch3/yamls/storage-secret.yaml
$ kubectl apply -f ./ch3/yamls/repository.yaml
$ kubectl get repository -n demo
```

Now, Let's apply the BackupConfiguration.

```shell
$ kubectl apply -f ./ch3/yamls/backupconfiguration.yaml
$ kubectl get backupconfiguration -n demo
```

We can manually trigger a backup by creating a BackupSession manually.

```shell
$ kubectl apply -f ./ch3/yamls/backupsession.yaml
$ watch kubectl get backupsession -n demo
```

When the BackupSession is successful, we can check our bucket if the data is there in our mentioned bucket.

Now, we can pause our backup so that no other Backup is taken, as we will perform restore operation now.
```shell
$ kubectl patch backupconfiguration -n demo mg-rs-backup --type="merge" --patch='{"spec": {"paused": true}}'
$ kubectl get backupconfiguration -n demo
```

## Restore

First, we will simulate a disaster scenario by dropping the `kubecon` database.

```shell
$ kubectl exec -n demo mg-rs-2 -it -- bash

# You'll be connected to the pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD

# You'll be connected to the mongo shell
$ use kubecon
$ db.dropDatabase()
$ show collections
$ show dbs
```

Now, we'll restore the database from our s3 backup. We have to create a RestoreSession to restore the database.

```shell
$ kubectl apply -f ./ch3/yamls/restoresession.yaml
```

We can watch the RestoreSession for the restore progress.

```shell
$ watch kubectl get restoresession -n demo
```

After the RestoreSession is Successful, we can verify the documents for the mongo shell.

```shell
$ kubectl exec -n demo mg-rs-0 -it -- bash

# You'll be connected to the pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD

# You'll be connected to the mongo shell
$ show dbs
$ use kubecon
$ show collections
$ db.participants.find()
$ db.participants.count()
```

So, the database is restored successfully.
