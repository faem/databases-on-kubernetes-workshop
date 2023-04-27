# Running Databases on Kubernetes Workshop

This tutorial goes over how to run a MongoDB database in Kubernetes using the Operator based approach. We will be using KubeDB as the database operator.

## Pre-requisites
- An active Kubernetes Cluster.
  - a `kind` or `minikube` cluster works just fine for most of the part. You can follow [these instructions](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries) to install `kind`
- A working `helm` installation.
  - You can install `helm` from [here](https://helm.sh/)
- Clone this repo and change the current directory to this repo.
  ```shell
    $ git clone https://github.com/faem/databases-on-kubernetes-workshop.git
    $ cd databases-on-kubernetes-workshop
  ```
## Create a kind cluster
Create a Kind cluster by running the following command
```shell
  $ kind create cluster
```

## Install supporting tools
- Install KubeDB
  - Get a trial license:
    - At first, go to [AppsCode License Server](https://license-issuer.appscode.com/?p=kubedb-enterprise)
    - Provide your name and email address.
    - Then, select KubeDB Enterprise Edition in the product field.
    Now, provide your cluster ID. You can get your cluster ID easily by running this command: `kubectl get ns kube-system -o=jsonpath='{.metadata.uid}'`
    - Then, you have to agree with the terms and conditions.
    - Now, you can submit the form. After you submit the form, the AppsCode License server will email to the provided email address with a link to your license file.
    - Navigate to the provided link and save the license into a file. Here, we save the license to a `license.txt` file.
  - Install KubeDB via helm:
    - KubeDB can be installed via Helm using the chart from AppsCode Charts Repository. To install, follow the steps below:
    ```shell
      $ helm repo add appscode https://charts.appscode.com/stable/
      $ helm repo update
      
      $ helm search repo appscode/kubedb
        NAME                              	CHART VERSION	APP VERSION	DESCRIPTION                                       
        appscode/kubedb                   	v2023.04.10  	v2023.04.10	KubeDB by AppsCode - Production ready databases...
        .....
    
      # Install KubeDB Enterprise edition
      $ helm install kubedb appscode/kubedb \
        --version v2023.04.10 \
        --namespace kubedb --create-namespace \
        --set kubedb-provisioner.enabled=true \
        --set kubedb-ops-manager.enabled=true \
        --set kubedb-autoscaler.enabled=true \
        --set kubedb-dashboard.enabled=true \
        --set kubedb-schema-manager.enabled=true \
        --set-file global.license=./license.txt
    ```
- Install Stash
  - Stash can be installed via Helm using the chart from AppsCode Charts Repository. To install the chart with the release name stash:
    ```shell
      $ helm search repo appscode/stash
      NAME            CHART VERSION  APP VERSION  DESCRIPTION
      appscode/stash  v2023.03.20    v2023.03.20  Stash by AppsCode - Backup your Kubernetes native applications
      
      $ helm install stash appscode/stash    \
        --version v2023.03.20                  \
        --namespace kube-system                 \
        --set features.enterprise=true          \
        --set-file global.license=./license.txt
    ```
- Install Panopticon
  - Stash can be installed via Helm using the chart from AppsCode Charts Repository. To install the chart with the release name panopticon:
  ```shell
    $ helm search repo appscode/panopticon
      NAME               	CHART VERSION	APP VERSION	DESCRIPTION
      appscode/panopticon	v2023.03.23  	v0.0.8     	Kubernetes Panopticon by AppsCode

    $ helm install panopticon appscode/panopticon \
      --version v2023.03.23 \
      --namespace kubeops --create-namespace \
      --set monitoring.serviceMonitor.labels.release=prometheus \
      --set-file license=./license.txt
  ```

- Install KubeDB Metrics
  - Stash can be installed via Helm using the chart from AppsCode Charts Repository. To install the chart with the release name panopticon:
  ```shell
    $ helm search repo appscode/kubedb-metrics
      NAME                   	CHART VERSION	APP VERSION	DESCRIPTION
      appscode/kubedb-metrics	v2023.04.10  	v2023.04.10	KubeDB State Metrics

    $ helm install kubedb-metrics appscode/kubedb-metrics \
      --version v2023.04.10 \
      --namespace kubeops --create-namespace
  ```
  
- Install Prometheus
  - Prometheus can be installed via Helm using the chart from prometheus community chart. To install the chart with the release name prometheus:
  ```shell
    $ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    $ helm repo update
    $ helm install prometheus prometheus-community/kube-prometheus-stack --create-namespace -n monitoring
  ```

- Install Cert-manager
  - Cert-manager can be installed via Helm using the chart from jetstack. To install the chart with the release name cert-manager:
  ```shell
    $ helm repo add jetstack https://charts.jetstack.io 
    $ helm repo update
    
    $ helm search repo jetstack/cert-manager
      NAME                                   	CHART VERSION	APP VERSION	DESCRIPTION                                       
      jetstack/cert-manager                  	v1.11.1      	v1.11.1    	A Helm chart for cert-manager
      ........
    
    $ helm install cert-manager jetstack/cert-manager \
        --namespace cert-manager \
        --create-namespace \
        --version v1.11.1 \
        --set installCRDs=true
  ```

## Tutorials
- [Chapter 1: Deploy MongoDB Database](ch1/)
- [Chapter 2: Day 2 Operations of MongoDB Database](ch2/)
- [Chapter 3: Backup and Restore of MongoDB Database](ch3/)
- [Chapter 4: Autoscaling MongoDB Database](ch4/)
