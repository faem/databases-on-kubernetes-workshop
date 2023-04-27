# Day 2 Operations of MongoDB Database

Now, We'll see some day 2 operations of a mongodb database on kubernetes, such as version update, horizontal & vertical scaling, reconfiguration & volume expansion.

## Prerequisite
- A mongodb replicaSet deployed in the cluster. Please follow the first chapter to deploy one.

## Version Update
Let's apply the version update ops request.
```shell
$ kubectl apply -f ./ch2/yamls/1_update_version.yaml
```

Watch the database pods to see them restart one by one.

```shell
$ watch kubectl get pods -n demo --label-columns kubedb.com/role
```

Wait for the ops request to be Successful.
```shell
$ watch kubectl get mongodbopsrequests -n demo
```

After the ops request is Successful, Let's verify the version is updated successfully from the MongoDB yaml and the related pod and StatefulSets.
```shell
$ kubectl get mg -n demo mg-rs -o=jsonpath='{.spec.version}{"\n"}'
$ kubectl get sts -n demo mg-rs -o=jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
$ kubectl get pods -n demo mg-rs-0 -o=jsonpath='{.spec.containers[0].image}{"\n"}'
```

Also, We can verify by connecting to a member of the replicaset. 
```shell
$ kubectl exec -n demo mg-rs-1 -it -- bash

# You'll be connected to the pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD --tls --tlsCAFile /var/run/mongodb/tls/ca.crt --tlsCertificateKeyFile /var/run/mongodb/tls/client.pem

# You'll be connected to the mongo shell
$ db.version()
```

We can also verify that the documents that we've inserted previously is retained after the version update.
```shell
$ show dbs
$ use kubecon
$ show collections
$ db.workshop.find()
```

## Horizontal Scaling

Let's Scale the replicas of our database to 4 from 3. To do that, we need to apply a new op request of type HorizontalScaling.
```shell
$ kubectl apply -f ./ch2/yamls/2_horizontal_scale_up.yaml
```

Watch the database pods to see a new secondary pod to come up and running.

```shell
$ watch kubectl get pods -n demo --label-columns kubedb.com/role
```

Wait for the ops request to be Successful.
```shell
$ watch kubectl get mongodbopsrequests -n demo
```

After the ops request is Successful, Let's verify the number of replicas this database has from the MongoDB object, number of pods the statefulSet has,

```shell
$ kubectl get mongodb -n demo mg-rs -o json | jq '.spec.replicas'
$ kubectl get sts -n demo mg-rs -o json | jq '.spec.replicas'
```

We can verify by connecting to a member of the replicaset.
```shell
$ kubectl exec -n demo mg-rs-1 -it -- bash

# You'll be connected to the pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD --tls --tlsCAFile /var/run/mongodb/tls/ca.crt --tlsCertificateKeyFile /var/run/mongodb/tls/client.pem

# You'll be connected to the mongo shell
$ rs.status()
```

Similarly, we can scale down the replicaSet from 4 to 3 replicas using another ops request.

```shell
$ kubectl apply -f ./ch2/yamls/3_horizontal_scale_down.yaml
```

Watch the database pods to see a new secondary pod to come up and running.

```shell
$ watch kubectl get pods -n demo --label-columns kubedb.com/role
```

Wait for the ops request to be Successful.
```shell
$ watch kubectl get mongodbopsrequests -n demo
```

After the ops request is Successful, Let's verify the number of replicas this database has from the MongoDB object, number of pods the statefulSet has,

```shell
$ kubectl get mongodb -n demo mg-rs -o json | jq '.spec.replicas'
$ kubectl get sts -n demo mg-rs -o json | jq '.spec.replicas'
```

We can verify by connecting to a member of the replicaset.
```shell
$ kubectl exec -n demo mg-rs-0 -it -- bash

# You'll be connected to the pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD --tls --tlsCAFile /var/run/mongodb/tls/ca.crt --tlsCertificateKeyFile /var/run/mongodb/tls/client.pem

# You'll be connected to the mongo shell
$ rs.status()
```

## Vertical Scaling

Let's scale the cpu and memory of the members of the replicaSet. Before that, Let's check the current cpu and memory. 

```shell
$ kubectl get mg mg-rs -n demo -o=jsonpath='{.spec.podTemplate.spec.resources}{"\n"}'
$ kubectl get sts mg-rs -n demo -o jsonpath='{.spec.template.spec.containers[0].resources}'
$ kubectl get pods mg-rs-0 -n demo -o=jsonpath='{.spec.containers[0].resources}{"\n"}'
```

Now, to vertically scale the database, we need to apply an ops request of type `VerticalScaling`.

```shell
$ kubectl apply -f ./ch2/yamls/4_vertical_scaling.yaml
```

Watch the database pods to see them restart one by one.

```shell
$ watch kubectl get pods -n demo --label-columns kubedb.com/role
```

Wait for the ops request to be Successful.
```shell
$ watch kubectl get mongodbopsrequests -n demo
```

After the ops request is Successful, Let's verify the cpu and memory this database has from the MongoDB object, resources the pods and the statefulSet has,

```shell
$ kubectl get mg mg-rs -n demo -o=jsonpath='{.spec.podTemplate.spec.resources}{"\n"}'
$ kubectl get sts mg-rs -n demo -o jsonpath='{.spec.template.spec.containers[0].resources}'
$ kubectl get pods mg-rs-0 -n demo -o=jsonpath='{.spec.containers[0].resources}{"\n"}'
```

## Reconfiguration

Now, we'll reconfigure the database with a new configuration. Let's check our current configuration file.


```shell
# Connect to the shell of one mongodb pod
$ kubectl exec -n demo mg-rs-0 -it -- bash
# You'll be connected pod shell
$ cat /data/configdb/mongod.conf
```

Now, to reconfigure the database, we need to apply an ops request of type `Reconfigure`.

```shell
$ kubectl apply -f ./ch2/yamls/5_reconfigure.yaml
```

Watch the database pods to see them restart one by one.

```shell
$ watch kubectl get pods -n demo --label-columns kubedb.com/role
```

Wait for the ops request to be Successful.
```shell
$ watch kubectl get mongodbopsrequests -n demo
```

After the ops request is Successful, Let's verify the configuration is applied from inside the database pod and mongo shell,

```shell
# Connect to the shell of one mongodb pod
$ kubectl exec -n demo mg-rs-0 -it -- bash
# You'll be connected to the pod shell
$ cat /data/configdb/mongod.conf
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD --tls --tlsCAFile /var/run/mongodb/tls/ca.crt --tlsCertificateKeyFile /var/run/mongodb/tls/client.pem

# You'll be connected to the mongo shell
$ db.adminCommand({getCmdLineOpts: 1})
```

## Reconfiguration of TLS

Now, we are going to reconfigure the TLS of our database. We'll remove the tls from our

First, Let's connect to the database using tls.
```shell
$ kubectl exec -n demo mg-rs-0 -it -- bash
# You'll be connected to the pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD --tls --tlsCAFile /var/run/mongodb/tls/ca.crt --tlsCertificateKeyFile /var/run/mongodb/tls/client.pem
```

Now, we will remove the TLS from this database. We need to apply an ops request of type `ReconfigureTLS`.

```shell
$ kubectl apply -f ./ch2/yamls/6_reconfigure_tls.yaml
```

Watch the database pods to see them restart one by one.

```shell
$ watch kubectl get pods -n demo --label-columns kubedb.com/role
```

Wait for the ops request to be Successful.
```shell
$ watch kubectl get mongodbopsrequests -n demo
```

After the ops request is Successful, Let's verify by connecting to the database without TLS.

```shell
$ kubectl exec -n demo mg-rs-2 -it -- bash
# You'll be connected to the pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD
# Check the documents
$ show dbs
$ use kubecon
$ show collections
$ db.workshop.find()
```

So, we've successfully removed the TLS from our database.

## Volume Expansion (optional)

Volume expansion requires a real kubernetes cluster with a storageclass that has `ALLOWEDVOLUMEEXPANSION` field true.

Let's check the current PVCs.

```shell
$ kubectl get pvc -n demo
```

So, all the pvc has 10Gi of storage. Now we will expand it to make it 20Gi. To do that, we have to apply an ops request of type `VolumeExpansion`

```shell
$ kubectl apply -f ./ch2/yamls/7_volume_expansion.yaml
```

Now, we can watch the database pods to see them restart one by one. Also, we can watch the PVCs to see them expand at the same time.

```shell
$ watch kubectl get pods -n demo --label-columns kubedb.com/role

# In another terminal
$ watch kubectl get pvc -n demo
```

Wait for the ops request to be Successful.
```shell
$ watch kubectl get mongodbopsrequests -n demo
```

After the ops request is Successful, Let's verify the size of all pvcs.

```shell
$ kubectl get pvc -n demo
```
