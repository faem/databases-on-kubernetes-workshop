# Autoscaling of MongoDB Database

## Prerequisite
- A mongodb replicaSet deployed in the cluster. If you don't have one, Please follow the first chapter to deploy one. 
- We need Metrics server installed on the cluster. To install we can just apply the following command
    ```shell
    $ helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
    $ helm repo update
    $ helm install metrics-server metrics-server/metrics-server -n kube-system --set=args={--kubelet-insecure-tls}
    ```

## Deploy MongoDB Autoscaler
We will autoscale the resources of our database using MongoDBAutoscaler. 

Let's check the current cpu and memory.

```shell
$ kubectl get mg mg-rs -n demo -o=jsonpath='{.spec.podTemplate.spec.resources}{"\n"}'
$ kubectl get sts mg-rs -n demo -o jsonpath='{.spec.template.spec.containers[0].resources}'
$ kubectl get pods mg-rs-0 -n demo -o=jsonpath='{.spec.containers[0].resources}{"\n"}'
```

Now, We will apply the MongoDBAutoscaler yaml.

```shell
$ kubectl apply -f ./ch4/yamls/mongodbautoscaler.yaml
```

Now, we'll watch the mongodbopsrequests. When the autoscaler generates a recommendation, it'll create a mongodbopsrequests.

```shell
$ watch kubectl get mongodbopsrequests -n demo
```

When an ops request is created, we can watch the database pods to see them restart one by one.

```shell
$ watch kubectl get pods -n demo --label-columns kubedb.com/role
```

After the ops request is Successful, we can check the cpu and memory this database currently has from the MongoDB object, resources the pods and the statefulSet has,

```shell
$ kubectl get mg mg-rs -n demo -o=jsonpath='{.spec.podTemplate.spec.resources}{"\n"}'
$ kubectl get sts mg-rs -n demo -o jsonpath='{.spec.template.spec.containers[0].resources}'
$ kubectl get pods mg-rs-0 -n demo -o=jsonpath='{.spec.containers[0].resources}{"\n"}'
```

So, we can see that the database is autoscaled successfully.

