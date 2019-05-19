# bench_multiregion

Benchmark of M3 in a multi-region environment, currently only supported for GKE.

## GKE Guide

### 1. Provision Kubernetes clusters in multiple regions

Provision a regional Kubernetes cluster in each region you will run the benchmark.

To achieve best performance, set the following attributes for nodes (lower resources work fine, they will not perform as many writes per second however):
- Machine type `n1-highmem-32` (32 vCPUs, 208 GB memory)
- SSD disk

Set your GCP project context:
```bash
gcloud config set project $PROJECT_NAME
```

Retrieve the credentials for each cluster:
```bash
gcloud container clusters get-credentials --region $REGION $CLUSTER_NAME
```

### 2. Create M3DB cluster in each region

Set the context for the region:
```bash
# List contexts
    kubectl config get-contexts
# Set context
kubectl config use-context $CONTEXT_NAME
```

For GKE we must set a cluster admin binding first for the installation:
```bash
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$EMAIL
```

Install the operator:
```bash
kubectl apply -f https://raw.githubusercontent.com/m3db/m3db-operator/master/bundle.yaml
```

Create an etcd cluster:
```bash
kubectl apply -f https://raw.githubusercontent.com/m3db/m3db-operator/master/example/etcd/etcd-basic.yaml
```

Let's create a storage class to use named something like `m3-cluster-storage-class.yml`:
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: topology-aware-ssd
provisioner: kubernetes.io/gce-pd
# WaitForFirstConsumer required with k8s 1.12 
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: pd-ssd
```

Now apply:
```bash
kubectl apply -f $STORAGE_CLASS_YAML_FILE
```

Let's also apply the `sysctl-setter` daemon set to make sure the correct system values are set:
```bash
kubectl apply -f https://raw.githubusercontent.com/m3db/m3/master/kube/sysctl-daemonset.yaml
```

Create a spec for cluster (see `examples` for an example):
```yaml
apiVersion: operator.m3db.io/v1alpha1
kind: M3DBCluster
metadata:
  name: m3-cluster
spec:
  image: quay.io/m3db/m3dbnode:latest
  replicationFactor: 3
  numberOfShards: 1024
  etcdEndpoints:
  - http://etcd-0.etcd:2379
  - http://etcd-1.etcd:2379
  - http://etcd-2.etcd:2379
  isolationGroups:
  - name: group1
    numInstances: 10
    nodeAffinityTerms:
    - key: failure-domain.beta.kubernetes.io/zone
      values:
      - <zone-a>
  - name: group2
    numInstances: 10
    nodeAffinityTerms:
    - key: failure-domain.beta.kubernetes.io/zone
      values:
      - <zone-b>
  - name: group3
    numInstances: 10
    nodeAffinityTerms:
    - key: failure-domain.beta.kubernetes.io/zone
      values:
      - <zone-c>
  podIdentityConfig:
    sources: []
  namespaces:
  - name: bench-10s:14d
    options:
      bootstrapEnabled: true
      flushEnabled: true
      writesToCommitLog: true
      cleanupEnabled: true
      snapshotEnabled: true
      repairEnabled: false
      retentionOptions:
        retentionPeriod: 336h
        blockSize: 2h
        bufferFuture: 10m
        bufferPast: 10m
        blockDataExpiry: true
        blockDataExpiryAfterNotAccessPeriod: 10m
      indexOptions:
        enabled: true
        blockSize: 2h
  dataDirVolumeClaimTemplate:
    metadata:
      name: m3db-data
    spec:
      # Note: using the storage class we created
      storageClassName: topology-aware-ssd
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 500Gi
```

Apply the manifest:
```bash
kubectl apply -f $SPEC_YAML_FILE
```

Ensure nodes are provisioning:
```bash
watch -n 1 -- kubectl get po -l operator.m3db.io/app=m3db
```

## 3. Install dedicated coordinator in each region

Now we will create a m3coordinator dedicated deployment, so that writes and reads are issued from a process isolated from the DB nodes.

Create values for the config template in a file named something like `m3coordinator-deployment-values.yaml`:
```yaml
cluster_namespace: m3db
cluster_name: bench-cluster-m3coordinator
num_coordinators: 30
namespaces:
- name: bench-10s:14d
  type: unaggregated
  retention: 336h
etcd_endpoints:
- http://etcd-0.etcd:2379
- http://etcd-1.etcd:2379
- http://etcd-2.etcd:2379
```

Now apply the config using helm to template and execute the resulting manifests:
```bash
# Set repo path to where your repo is locally
export REPO_OPERATOR_PATH=$GOPATH/src/github.com/m3db/m3db-operator
# Set the values file to the file you just created
export VALUES_YAML_FILE=m3coordinator-deployment-values.yaml
# First check the template looks sane
helm template -f $VALUES_YAML_FILE ${REPO_OPERATOR_PATH}/helm/m3coordinator | less
# Now apply the config
helm template -f $VALUES_YAML_FILE ${REPO_OPERATOR_PATH}/helm/m3coordinator | kubectl apply -f -
```

## 4. Monitor the clusters with Prometheus in each region

Let's get some visibility on how the clusters are doing. We will setup a Prometheus in each region.

Install the Prometheus operator:
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/prometheus-operator/master/bundle.yaml
```

Create the manifests to start prometheus, grafana, node-exporter, etc:
```bash
kubectl create -f ./examples/kube-prometheus-manifests
```

Create a service monitor to start scraping the M3DB nodes:
```bash
kubectl apply -f ./examples/prometheus-servicemonitor.yaml
```

Port forward to access Prometheus and Grafana:
```bash
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090
kubectl --namespace monitoring port-forward svc/grafana 3000
```

Add the following dashboard to Grafana:
[https://grafana.com/dashboards/8126](https://grafana.com/dashboards/8126). 

## 5. Add load from promremotebench

Create a manifest to launch Prometheus remote write benchmarkers. They simulate load from node exporter nodes.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: promremotebench
spec:
  replicas: 10
  selector:
    matchLabels:
      k8s-app: promremotebench
  template:
    metadata:
      labels:
        k8s-app: promremotebench
    spec:
      containers:
      - name: promremotebench
        image: quay.io/m3db/promremotebench
        env:
        - name: PROMREMOTEBENCH_TARGET
          value: "http://m3coordinator-dedicated-bench-cluster:7201/api/v1/prom/remote/write"
        - name: PROMREMOTEBENCH_NUM_HOSTS
          value: "1000"
        - name: PROMREMOTEBENCH_INTERVAL
          value: "10"
        - name: PROMREMOTEBENCH_BATCH
          value: "128"
```

Now apply the manifest:
```bash
kubectl apply -f $PROMREMOTEBENCH_DEPLOYMENT_YAML_FILE
```

Now the rough rule of thumb is that every replica of the benchmarkers is ballpark around 1,000 writes per second load each. To get to desired writes per second, step up the writes in increments of 50,000 or so (i.e. adding 50 replicas each step). You should monitor your cluster to see how it's doing it at each increment.
