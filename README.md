# Kubernetes Basic

This tutorial walks you through some basic kubernetes concepts that you can try out locally.


## Prerequsites

- Install [Docker Desktop](https://docs.docker.com/desktop/install/mac-install/).

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"

# Install k9s
# a cli tool to access kubernetes cluster
brew install derailed/k9s/k9s

# Install k3d
# k3d creates kubernetes cluster locally
brew install k3d
```

## Create Kubernetes Cluster locally

```bash
# Create a cluster with 2 worker nodes
k3d cluster create my-first-cluster --agents 2

# update kuberconfig to local cluster
k3d kubeconfig write my-first-cluster

# very cluster is created & connected
kubectl cluster-info
```

Learn more useful [k3d commands](https://k3d.io/v5.6.0/usage/commands/k3d_cluster_list/) when you get a moment
- `k3d cluster list` - lists all your cluster. That's right you can create a cluster for any experiment you want to try in isoloation!
- `k3d cluster stop my-first-cluster` - to stop your cluster
- `k3d cluster start my-first-cluster` - to start your cluster
- `k3d cluster delete my-firs-cluster` - to delete your cluster


## Lab

### Find all pods in `kube-system` namespace

If you see pods running then you are setup correctly!

```bash
kubectl get po -n kube-system

# Output
NAME                                     READY   STATUS      RESTARTS   AGE
local-path-provisioner-957fdf8bc-rr5x2   1/1     Running     0          31s
coredns-77ccd57875-nn4sq                 1/1     Running     0          31s
helm-install-traefik-crd-p82qm           0/1     Completed   0          31s
svclb-traefik-ad900558-j4ql4             2/2     Running     0          22s
```

### Find all nodes

Kubernetes has 2 sorts of nodes.
- [**control plane**](https://kubernetes.io/docs/concepts/overview/components/) nodes is the brain of k8s orchestration. It's responsible for scheduling your nodes via schedular, manage configuratio state via etcd
  - etcd - high availability database to store configuration
  - API Server - for clients to interact with k8s
  - Scheduler - to ensure pods have a home on a _worker node_
  - Controller Manager - does reconsiliation of all resources to ensure declarative state is respected
  - Cloud controller manager - to talk with Cloud provider
- [**worker nodes**](https://kubernetes.io/docs/concepts/overview/components/#node-components).
  - responsible to run your applicaiton code

```bash
# List all nodes
kubectl get nodes

# Output
NAME                            STATUS   ROLES                  AGE   VERSION
k3d-my-first-cluster-server-0   Ready    control-plane,master   11m   v1.27.4+k3s1
k3d-my-first-cluster-agent-1    Ready    <none>                 11m   v1.27.4+k3s1
k3d-my-first-cluster-agent-0    Ready    <none>                 11m   v1.27.4+k3s1
```

You will notice `xxx-server-0` is the _control-plane_ node and `xxx-agent-n` is a worker node.


## Finding pods

```bash
# To find all pods in a namespace
kubectl get pods -n kube-system

# To find all pods in a node - k3d-my-first-cluster-agent-0
kubectl get pods --field-selector spec.nodeName=k3d-my-first-cluster-agent-0 --all-namespaces
```

## Creating a namespace

In Kubernetes, [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) provides a mechanism for isolating groups of resources within a single cluster

You can isolate your pods by putting it in a namespace. Let's create a namespace `team-red` by applying [namepsace.yaml](./namespace.yml)

```base
# This will create kubernetes resources defined in the file
kubectl apply -f ./namespace.yaml

# Let's verify it's created
kubectl get namespace

# Output - notice team-red
➜  kubernetes-basics git:(main) ✗ k get namespace
NAME              STATUS   AGE
kube-system       Active   23m
default           Active   23m
team-red          Active   5s
```

## Creating a configmap

A [configmap](https://kubernetes.io/docs/concepts/configuration/configmap/) object is used to store non-confidential data in key-value pairs.

```bash
# creates resources in configmap.yaml in team-red namespace
kubectl apply -f ./configmap.yaml -n team-red

# Verify it's created
k get configmap -n team-red

# Output
NAME               DATA   AGE
kube-root-ca.crt   1      5m48s
team-red-cm        2      8s
```

### Use `kubectl describe` to view contents of a resource

```bash
kubectl describe cm team-red-cm -n team-red
```

### NEVER use `kubectl edit` to edit contents of a resource

We loose declarative tracking of resources via `kubectl.kubernetes.io/last-applied-configuration`

```bash
# Avoid doing this, unless it's absolutely necessary
kubectl edit cm team-red-cm -n team-red
```

### Update value using `kubectl apply -f` instead

Make change to `configmap.yaml` by changing `player_initial_lives` from 3 to 4.

```bash
kubectl apply -f ./configmap -n team-red
```

### Delete resource using `kubectl delete -f`

```bash
kubectl delete -f ./configmap.yaml -n team-red
```

## Create a Deployment

A [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) provides a way to release your code with high availability with zero downtime (when configured correctly). It's backed by ReplicaSet and Pods.

```bash
kubectl apply -f ./deployment.yaml -n team-red

# Verify it's running
k get pod -n team-red

# Output
NAME                            READY   STATUS    RESTARTS   AGE
team-red-app-798f464c7f-rxzcp   1/1     Running   0          10s
team-red-app-798f464c7f-4r876   1/1     Running   0          10s
```

# Create a Service

A [Service](https://kubernetes.io/docs/concepts/services-networking/service/) helps expose one or more pods via a singular ip, load balancing request to pods that back it.


