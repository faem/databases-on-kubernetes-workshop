# Deploy MongoDB Database

## Deploy a TLS Secured ReplicaSet Database

First, Create a namespace `demo`

```shell
$ kubectl create ns demo
```

Then, Create a CA certificate secret.

```shell
$ kubectl create secret -n demo tls mongo-ca \
           --cert=./ch1/yamls/replicaset/ca.crt \
           --key=./ch1/yamls/replicaset/ca.key
```
Then, create Issuer and check if `Ready`.

```shell
$ kubectl apply -f ./ch1/yamls/replicaset/issuer.yaml
$ kubectl get issuer -n demo
```

Create config secret with `mongod.conf` and `replicaset.json`.

```shell
$ kubectl create secret generic -n demo custom-config --from-file=./ch1/yamls/replicaset/mongod.conf --from-file=./ch1/yamls/replicaset/replicaset.json
```

Create configmap with init script
```shell
$ kubectl create configmap -n demo init-script --from-file=./ch1/yamls/replicaset/init.js
```

Now, apply the mongodb yaml and wait for the DB to get `Ready`

```shell
$ kubectl apply -f ./ch1/yamls/replicaset/mg-rs.yaml
$ watch kubectl get mg -n demo
$ watch kubectl get pods -n demo --label-columns kubedb.com/role
```

After the DB is Ready, Lets connect to the primary of the replicaSet.

```shell
$ kubectl exec -n demo mg-rs-0 -it -- bash

# You'll be connected pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD --tls --tlsCAFile /var/run/mongodb/tls/ca.crt --tlsCertificateKeyFile /var/run/mongodb/tls/client.pem

# You'll be connected to mongo shell
$ show dbs
$ use kubecon
$ show collections
$ db.workshop.find()
```

Now, Let's check the configuration that we have provided via config secret.

```shell
# In the mongo shell
$ db.adminCommand({getCmdLineOpts: 1})
```

Let's check the status of the replicaset.

```shell
# In the mongo shell
$ rs.status()
```

Let's check the configuration we provided via `replicaset.json`.

```shell
$ rs.conf().settings
```

Let's insert some documents in a collection in the `kubecon` database.

```shell
$ use kubecon
$ db.database.insert({"db1": "mongodb"})
$ db.database.insert({"db2": "mysql"})
$ db.database.insert({"db3": "postgres"})
$ db.database.find()
```

Let's exit from the mongo shell and pod shell by issuing the `exit` command on both shell.

Now, As we have enabled monitoring for our database. We can check the grafana dashboards for our database.

Let's port-forward the grafana service.

```shell
$ kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

Now, we can visit grafana dashboard via http://localhost:3000 and login to the grafana dashboard.

Default username is `admin` and password is `prom-operator`. You can view the username and password using the following command too.

```shell
$ kubectl get secret -n monitoring prometheus-grafana -o go-template='{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'
```

After logging in to the grafana dashboard, we are going to add some grafana dashboard to monitor our database by following this [link](https://github.com/appscode/grafana-dashboards/tree/master/mongodb#import-grafana-dashboard)

Let's try to delete the database.

```shell
$ kubectl delete mg -n demo mg-rs
# Denied by webhook because of terminationPolicy
$ kubectl patch mg -n demo mg-rs -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
$ kubectl delete mg -n demo mg-rs
```

## Deploy a Sharded Cluster

First, we'll create a config secret with `mongod.conf` for our shard.

```shell
$ kubectl create secret generic -n demo shard-config --from-file=./ch1/yamls/shard/mongod.conf
```

Now, apply the mongodb yaml and wait for the DB to get `Ready`

```shell
$ kubectl apply -f ./ch1/yamls/shard/mg-sh.yaml
$ watch kubectl get mg -n demo
$ watch kubectl get pods -n demo --label-columns kubedb.com/role
```

After the DB is Ready, Lets connect to the mongos of the cluster and check the shard status.

```shell
$ kubectl exec -n demo mg-sh-mongos-0 -it -- bash

# You'll be connected pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD

# You'll be connected to mongo shell
$ sh.status()
```

Let's insert some documents in a collection in the `kubecon` database.

```shell
$ use kubecon
$ db.database.insert({"db1": "mongodb"})
$ db.database.insert({"db2": "mysql"})
$ db.database.insert({"db3": "postgres"})
$ db.database.find()
$ sh.status()
```

Let's exit from the mongo shell and pod shell by issuing the `exit` command on both shell.

Now, Lets connect to the primary of the shard replicaSet and check the replicaSet status.

```shell
$ kubectl exec -n demo mg-sh-shard0-0 -it -- bash

# You'll be connected pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD

# You'll be connected to mongo shell
$ rs.status()
```

Now, Let's check the configuration that we have provided via config secret.

```shell
# In the mongo shell
$ db.adminCommand({getCmdLineOpts: 1})
```

Let's exit from the mongo shell and pod shell by issuing the `exit` command on both shell.

Now, Lets connect to the primary of the configServer replicaSet and check the replicaSet status.

```shell
$ kubectl exec -n demo mg-sh-configsvr-0 -it -- bash

# You'll be connected pod shell
$ mongo -u root -p $MONGO_INITDB_ROOT_PASSWORD

# You'll be connected to mongo shell
$ rs.status()
```

Let's exit from the mongo shell and pod shell by issuing the `exit` command on both shell.

Let's delete the database.

```shell
$ kubectl delete mg -n demo mg-sh
```
