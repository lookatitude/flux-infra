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
  - 1.1 - Existing cluster
  - 1.2 - Setup local cluster with Kind
- 2 - Install flux CLI
- 3 - Export github credentials
- 4 - bootstrap flux on your system
- 5 - Check all is running correctly
- 6 - review the file structure and how to organize your clusters


## 0 - Clone this repo

Clone thsi repo with ```git clone https://github.com/lookatitude/flux-infra.git```.
Navigate to the ```flux-infra``` directory.

## 1 - Kubernetes Cluster

In order to follow this manual you'll need to setup a local kuberntes cluster.

### 1.1 - Existing cluster

If you have a local cluster already just change your context to that cluster and move to step 2

### 1.2 - Setup local cluster with Kind

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

## 5 - Check all is running correctly

#### Check flux is ok
```flux check```this command will perform a check on all flux components runing on the cluster.

```bash
➜ flux check
► checking prerequisites
✔ Kubernetes 1.21.1 >=1.19.0-0
► checking controllers
✔ helm-controller: deployment ready
► ghcr.io/fluxcd/helm-controller:v0.14.1
✔ kustomize-controller: deployment ready
► ghcr.io/fluxcd/kustomize-controller:v0.18.2
✔ notification-controller: deployment ready
► ghcr.io/fluxcd/notification-controller:v0.19.0
✔ source-controller: deployment ready
► ghcr.io/fluxcd/source-controller:v0.19.2
✔ all checks passed
```

```flux get all -A```this command will check all your flux components and their current state.

```bash
NAMESPACE  	NAME                     	READY	MESSAGE                                                        	REVISION                                     	SUSPENDED
flux-system	gitrepository/flux-system	True 	Fetched revision: main/f0f510041d7678da25630983571a8762ac13a6ac	main/f0f510041d7678da25630983571a8762ac13a6ac	False

NAMESPACE  	NAME                        	READY	MESSAGE                                                        	REVISION                                     	SUSPENDED
flux-system	kustomization/apps          	True 	Applied revision: main/f0f510041d7678da25630983571a8762ac13a6ac	main/f0f510041d7678da25630983571a8762ac13a6ac	False
flux-system	kustomization/flux-system   	True 	Applied revision: main/f0f510041d7678da25630983571a8762ac13a6ac	main/f0f510041d7678da25630983571a8762ac13a6ac	False
flux-system	kustomization/infrastructure	True 	Applied revision: main/f0f510041d7678da25630983571a8762ac13a6ac	main/f0f510041d7678da25630983571a8762ac13a6ac	False
```


## 6 - review the file structure and how to organize your clusters

Your folder structure should look something like this:
```bash
├── README.md
├── apps
│   ├── base
│   └── local
│       └── kustomization..yaml
├── clusters
│   └── local
│       ├── apps.yaml
│       ├── flux-system
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       └── infrastructure.yaml
├── infrastructure
│   ├── common
│   ├── local
│   │   └── kustomization..yaml
│   └── sources
│       └── kustomization..yaml
├── kind.yaml
└── notes.txt
```

#### cluster
For every cluster you add you can create a new folder inside the clusters folder and add the specific configs for that cluster. 
Inside each cluster folder there is a flux-system folder created and updated by the flux bootstrap command, and 2 yaml files ```apps.yaml``` and ```ìnfrastructure.yaml``` this files point to where the infrastructure code and your apps code resides. 

#### Infrastructure
The infrastructure folder should define the softwares you want to run on your clusters. you will have 3 main folders, ```sources```, ```common``` and ```local```or the name of your cluster.
The common folder should have all the software you want to run on every cluster in a generic form, example an ingress controller deploy, the specific configs for the local environment can be added on the local folder.
The sources folder will hold the repos where your software resides, a helm registry info, or a git repository, 

#### apps
This folder contains the ```base``` folder were you add the base deploy of your apps. Add the specific configs for an environment inside that environment folder (cluster name) in this case ```local```.


## 7 - Infrastructure example install ingress-nginx

In this example we will install ingress nginx to test and understand our folder organization.

### 7.1 - Add a source
Adding the source repository where flux should grab the ingress-nginx helm chart open ```infrastructure/sources/kustomization.yaml``` and add the following content:
```bash 
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: flux-system
resources:
  - ingress-nginx-helm.yml
```

Now add the ingress-nginx repo file ```infrastructure/sources/ingress-nginx-helm.yaml``` and add the following content:
```bash
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: ingress-nginx
spec:
  interval: 1h
  url: https://kubernetes.github.io/ingress-nginx
```

note: As a convention and since the sources folder can hold several types of repositories and/or registries I personally add the type of source to the file name, in this case ```-helm```  to represent a helm repo.

So ```infrastructure/sources/ingress-nginx-helm.yaml``` describes where the repo is located, you can add and reference secrets to the repo if it is a private repo. for more information check (flux helm repositories examples)[https://fluxcd.io/docs/components/source/helmrepositories/#spec-examples]
The file  ```infrastructure/sources/kustomization.yaml``` Describes to flux what sources should be included in the cluster.

### 7.2 - Add ingress nginx components to be deployed

create a folder ```infrastructure/common/ingress-nginx``` this will hold the default for your ingress nginx deployment instructions.

#### namespace
Lets create a namespace to run our ingress in its own namespace, create a file in ```infrastructure/common/ingress-nginx/namespace.yaml``` and add the following content:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
```
This is just a simple namespace manifest from kubernetes.

#### Add a configmap 
Use this config map for the base settings of your ingress create a file in  ```infrastructure/common/ingress-nginx/configmap.yaml``` and add the following content:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
data:
  disable-access-log: "true"
  ssl-protocols: "TLSv1.2 TLSv1.3"
  server-tokens: "false"
```

#### Add a Helm release 
The helm release file describes how you want to install update and release your application. 
create a release file in ```infrastructure/```and add the following content:
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: nginx-ingress
spec:
  releaseName: nginx-ingress-controller
  interval: 1h0m0s
  chart:
    spec:
      chart: ingress-nginx
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
      version: "4.0.9"
  install:
    remediation:
      retries: 3
  values:
    controller:
      # Configures the controller container name
      containerName: controller
      # Configures the ports the nginx-controller listens on
      containerPort:
        http: 80
        https: 443
    kind: Deployment
    metrics:
      enabled: true
      port: 10254
    
    labels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/component: controller

    admissionWebhooks:
      annotations: {}
      enabled: true
      failurePolicy: Fail
      # timeoutSeconds: 10
      port: 8443

```

Now lets look at this file with more detail
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: nginx-ingress
spec:
  releaseName: nginx-ingress-controller
  interval: 1h0m0s
```
This section defines our helm release and give it a name, and settinga a release name we also telm how often we want to check this helm release for new versions in this case once evry hour.

The chart configuration 
```yaml
  chart:
    spec:
      chart: ingress-nginx
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
      version: "4.0.9"
```
here we define the chart we want to use, reference your source (we created this source in the previous step) the sourceRef.name value should be the exact same you called your source on the previous step. We are also defining the version (use semver rules here) in this case we are pointing to a specific version but you can set renges like ```4.x``` and it will grab the latest version starting with 4. in our case we want this specific version "4.0.9" so we have control over what exact version of the helm chart we have intalled.

Installation instructions
```yaml
  install:
    remediation:
      retries: 3
```
What happens if the installation fails? it will try again for a maximum of 3 times other wise it fails the installation.

The values
```yaml
  values:
    controller:
      # Configures the controller container name
      containerName: controller
      # Configures the ports the nginx-controller listens on
      containerPort:
        http: 80
        https: 443
    kind: Deployment
    metrics:
      enabled: true
      port: 10254
    
    labels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/component: controller

    admissionWebhooks:
      annotations: {}
      enabled: true
      failurePolicy: Fail
      # timeoutSeconds: 10
      port: 8443
```
In this case we want to pass our helm chart this values, in case you are dealing with a different chart refer to the values file of your chart to know what values you can add to the values section.

#### Let Kustomize know what files to use
Lets tell kustomize what files we want to run and in what order.
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ingress-nginx
resources:
  - namespace.yaml
  - configmap.yaml
  - release.yaml
```

### 7.3 - Add the specific configurations for your cluster
Now that we added the source for the helm chart and the default values for releasing an ingress-nginx we will use that base and add the modifications we need to our specific cluster, this is where you change the values for example to run on your local or production cluster.

Create a folder ``` infrastructure/local/ingress```
Create a file for your kustomizations ``` infrastructure/local/kustomization.yaml``` and add the following content:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../sources
  - ingress
```
So we are telling it to include 1st all the sources and then the contents in the ingress folder.

#### the ingress 

Create a file for your kustomizations ``` infrastructure/local/ingress/kustomization.yaml``` and add the following content:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: ingress-nginx
resources:
  - ../../common/ingress-nginx
patchesStrategicMerge:
 - values.yaml
```

create a file for your values ``` infrastructure/local/ingress/values.yaml``` and add the following content:
```yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: nginx-ingress
spec:
  releaseName: nginx-ingress-controller
  values:
    controller:
      updateStrategy:
        type: "RollingUpdate"
        rollingUpdate:
          maxUnavailable: 1
      hostPort:
        enabled: true
      terminationGracePeriodSeconds: 0
      service:
        type: "NodePort"
      watchingIngressWithoutClass: true
      nodeSelector:
        ingress-ready: "true"
      tolerations:
        - key: "node-role.kubernetes.io/master"
          operator: "Equal"
          effect: "NoSchedule"
      publishService:
        enabled: false
      extraArgs:
        publish-status-address: "localhost"
    metrics:
      enabled: true
      port: 10254
    
    labels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/instance: ingress-nginx
      app.kubernetes.io/component: controller

    admissionWebhooks:
      annotations: {}
      enabled: true
      failurePolicy: Fail
      # timeoutSeconds: 10
      port: 8443

```
This contains several nginx configurations so it plays well with kind as for the local environment we want to use our localhost and local IP so the services are set to "NodePort" on a external cluster this would be "LoadBalancer". for more details on nginx configuartions check the (helm chart documentation)[https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx#configuration]

### 7.4 - Confirm all is running

to check how is the state of you flux deployments use the command ```flux get all -A``` basically saying give me all flux resources in all namespaces ```-A```. You should see a result like this:
```bash
NAMESPACE  	NAME                     	READY	MESSAGE                                                        	REVISION                                     	SUSPENDED
flux-system	gitrepository/flux-system	True 	Fetched revision: main/59fb83bf0246c1fe6f42f747819404bd49c33a76	main/59fb83bf0246c1fe6f42f747819404bd49c33a76	False

NAMESPACE  	NAME                        	READY	MESSAGE                                                                           	REVISION                                                        	SUSPENDED
flux-system	helmrepository/ingress-nginx	True 	Fetched revision: b67e10d002be11c3bd121f8f5fe2d1d50213460f70fc708c11247cd995a818d6	b67e10d002be11c3bd121f8f5fe2d1d50213460f70fc708c11247cd995a818d6	False

NAMESPACE  	NAME                                 	READY	MESSAGE                                           	REVISION	SUSPENDED
flux-system	helmchart/ingress-nginx-nginx-ingress	True 	Pulled 'ingress-nginx' chart with version '4.0.9'.	4.0.9   	False

NAMESPACE    	NAME                     	READY	MESSAGE                         	REVISION	SUSPENDED
ingress-nginx	helmrelease/nginx-ingress	True 	Release reconciliation succeeded	4.0.9   	False

NAMESPACE  	NAME                        	READY	MESSAGE                                                        	REVISION                                     	SUSPENDED
flux-system	kustomization/apps          	True 	Applied revision: main/59fb83bf0246c1fe6f42f747819404bd49c33a76	main/59fb83bf0246c1fe6f42f747819404bd49c33a76	False
flux-system	kustomization/flux-system   	True 	Applied revision: main/59fb83bf0246c1fe6f42f747819404bd49c33a76	main/59fb83bf0246c1fe6f42f747819404bd49c33a76	False
flux-system	kustomization/infrastructure	True 	Applied revision: main/59fb83bf0246c1fe6f42f747819404bd49c33a76	main/59fb83bf0246c1fe6f42f747819404bd49c33a76	False
```

You cal also check using ```kubectl get all -n ingress-nginx``` and you should see something similar to:
```bash
NAME                                                                  READY   STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-ingress-nginx-controller-bcfb9d64jz4fq   1/1     Running   0          13m

NAME                                                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/nginx-ingress-controller-ingress-nginx-controller             NodePort    10.96.117.141   <none>        80:31744/TCP,443:30982/TCP   59m
service/nginx-ingress-controller-ingress-nginx-controller-admission   ClusterIP   10.96.126.41    <none>        443/TCP                      59m

NAME                                                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nginx-ingress-controller-ingress-nginx-controller   1/1     1            1           59m

NAME                                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/nginx-ingress-controller-ingress-nginx-controller-8477cccf69   0         0         0       59m
replicaset.apps/nginx-ingress-controller-ingress-nginx-controller-bcfb9d64     1         1         1       13m
```