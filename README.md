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
kubectl get pods -n kube-system

# Output
NAME                                     READY   STATUS      RESTARTS   AGE
local-path-provisioner-957fdf8bc-rr5x2   1/1     Running     0          31s
coredns-77ccd57875-nn4sq                 1/1     Running     0          31s
helm-install-traefik-crd-p82qm           0/1     Completed   0          31s
svclb-traefik-ad900558-j4ql4             2/2     Running     0          22s
```

### Find all nodes
![alt text](https://kubernetes.io/images/docs/components-of-kubernetes.svg)

Kubernetes has 2 sorts of nodes.
- [**control plane**](https://kubernetes.io/docs/concepts/overview/components/) nodes is the brain of k8s orchestration. It's contains:
  - etcd - high availability database to store configuration
  - API Server - for clients to interact with k8s
  - Scheduler - to ensure pods have a home on a _worker node_
  - Controller Manager - does reconsiliation of all resources to ensure declarative state is respected
  - Cloud controller manager - to talk with Cloud provider
- [**worker nodes**](https://kubernetes.io/docs/concepts/overview/components/#node-components).
  - container runtime - responsible to run your containers (applicaiton code). Defaults to _containerd_ since 1.26
  - kubelet - agent on node which ensures containers runs in a pod
  - kube-proxy - helps with networking and `Service` to do it's job

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
kubectl describe configmap team-red-cm -n team-red
```

### NEVER use `kubectl edit` to edit contents of a resource

We loose declarative tracking of resources via `kubectl.kubernetes.io/last-applied-configuration`

```bash
# Avoid doing this, unless it's absolutely necessary
kubectl edit configmap team-red-cm -n team-red
```

### Update value using `kubectl apply -f` instead

Make change to `configmap.yaml` by changing `player_initial_lives` from 3 to 4.

```bash
kubectl apply -f ./configmap.yaml -n team-red
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
kubectl get pods -n team-red

# Output
NAME                            READY   STATUS    RESTARTS   AGE
team-red-app-798f464c7f-rxzcp   1/1     Running   0          10s
team-red-app-798f464c7f-4r876   1/1     Running   0          10s
```

### Updating a deployment
Update the image version in _deployment.yaml_

- from: `image: mendhak/http-https-echo:29`
- to: `image: mendhak/http-https-echo:30`

You will need 2 terminals.

_terminal 1_

```bash
# Watch for pods updating
kubectl get pods -n team-red --watch
```

_terminal 2_
```bash
kubectl apply -f ./deployment.yaml -n team-red
```

You will notice a gradual roll-over of old pods with older image version to new pod with newer image version. This is a the default rollout strategy known as _Rolling Update_. Other [strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) are supported.

### Previous version of your deployments via ReplicaSets

To see the previous version deployed, find all corresponding replica sets.

```bash
kubectl get replicaset -n team-red

# Output
NAME                      DESIRED   CURRENT   READY   AGE
team-red-app-798f464c7f   0         0         0       12h
team-red-app-676dc9f797   2         2         2       15m
```

You can all use [Rollouts](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment) to control deployments for example to rollback a deployment. However this is discouraged from git-ops principles as we want git to be the source of truth. To rollback, the best thing to do would be to update the `deployment.yaml` with previous version of the image and run `kubectl apply -f ./deployment.yaml -n team-red` instead.


## Create a Service

A [Service](https://kubernetes.io/docs/concepts/services-networking/service/) helps expose one or more pods via a singular ip, load balancing request to pods that back it.

```bash
kubectl apply -f ./service.yaml -n team-red

# verify it's created
kubectl get services -n team-red

# Output
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
team-red-app   ClusterIP   10.43.207.82   <none>        8080/TCP   8s
```

### `port-forward` - connect to you app Pods via Service

Use [port-forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) command to communicate to your pods.

```
kubectl port-forward svc/team-red-app 8080:8080 -n team-red

# Output
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

In another terminal instance run

```
curl localhost:8080

# output
{
  "path": "/",
  "headers": {
    "host": "localhost:8080",
    "user-agent": "curl/8.1.2",
    "accept": "*/*"
  },
...
```
