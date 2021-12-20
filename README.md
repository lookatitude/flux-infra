# GitOps the easy way

This repo describes an easy aproach to get up and running with GitOps and Kubernetes. 
This is an opinionated aproach there are many alternatives to what we are doing here, once you understand the concepts you can change the tools as you please.

### Pre Requisites
- GitHub account 
- Access to a kubernetes cluster
  - this can work with any kubernetes cluster
  - I provide a simple script to create a local cluster in your machine. For the local cluster you'll need:
    - Docker running in your machine
    - Kind so you can spin up a multi-node cluster based on containers 
- CLI for flux v2
- ```kubectl```

## Contents
- 0 - Clone this repo
- 1 - Kubernetes cluster
  - existing cluster
  - local cluster
- 2 - install flux cli


## 0 - Clone this repo

Clone thsi repo with ```git clone https://github.com/lookatitude/flux-infra.git```.
Navigate to the ```flux-infra``` directory.

## 1 - Kubernetes Cluster

In order to follow this manual you'll need to setup a local kuberntes cluster.

### 1.1 - Existing cluster

If you have a local cluster already just change your context to that cluster and move to step 2

### 1.2 Setup local cluster with Kind

Follow the steps on (Kind)[https://kind.sigs.k8s.io/docs/user/quick-start/] to install Kind in your system.

After the kind cli is installed just spin your new cluster:
```
kind create cluster --name kind --config=kind.yaml
```
This will use the configuration on the ```kind.yaml``` file to spin up a new cluster.
kind:yaml
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
```
This conflig will generate a cluster with 1 control-plane and 2 workes you can adjust this file to your preference in order to fit your needs.

## 2 - Install flux CLI

In order to sync a repo you will need to install the (flux cli)[https://fluxcd.io/docs/installation/#install-the-flux-cli] follow the instructions to your system.

## 3 - Export github credentials

The flux tool will use your credentials to create a repo in case it does not exist and connect to that repo to add the flux system configuration and syncronise your cluster configuration.
```
export GITHUB_TOKEN=<your token>
export GITHUB_USER=<your username>
export GITHUB_REPO=<your repo name> # ex: flux-infra
```
Get your token from github personal tokens area under settings -> developer settings -> personal access tokens

## 4 - bootstrap flux on your system

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=./clusters/local \
  --personal \
  --private
```
```--repository=flux-infra``` name your repo (will be created if it does not exist and your token has permissions to create repos on your account).

```--branch=main``` branch you want to sync to your cluster

```--path=./clusters/local``` path in the repo were the flux config will sit.

```--personal``` personal github account 

```--private``` make the repo private (if you remove this the repo will be created as public)


There are other options for this tool like linking it with an Enterprise Github account, or other git rpoviders check (flux bootstrap)[https://fluxcd.io/docs/installation/#bootstrap]

Remove the reference to the currect repo and connect it to yours:

```git remote rm origin``` to remove the current remote reference

```git remote add origin git@github.com:$GITHUB_USER/GITHUB_REPO.git``` to remove the current remote reference


pull the changes and merge them with your current repo.

```git pull origin main``` get the config for your local repo.

Merge the repo and the file structure, commit it to your repo and push your changes.

