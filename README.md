# A Banking Workflow AKS Cluster using DAPR [WIP]

This sample creates an AKS Cluster and uses DAPR for service mesh. The project is built using [Azure Developer CLI](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/make-azd-compatible?pivots=azd-create) conventions.

## How it works?

This sample implements a simple banking workflow:

1. Public API endpoint receives new money transfer request. [TRANSFER(Sender: A, Amount: 100, Receiver:B)]
1. Request is published to Redis (pub/sub)
1. Deposit workflow starts
    1. Fraud service checks the legitimacy of the operation and triggers [VALIDATED(Sender: A, Amount: 100, Receiver:B)]
    1. Account service checks if `Sender` has enough funds and triggers [APPROVED(Sender: A, Amount: 100, Receiver: B)]
    1. [WIP] Notification services notifies both `Sender` and `Receiver`.
1. Public API can be used to check if there is a confirmation of the money transfer request.

![Workflow](/docs/flow.drawio.png)

## Services

The project contains the following services:

- [Public API](/src/public-api-service) - Public API endpoint that receives new money transfer requests, starts workflow and checks customer notifications.
- [Fraud Service](/src/fraud-service) - Fraud service that checks the legitimacy of the operation.
- [Account Service](/src/account-service) - Account service that checks if `Sender` has enough funds.
- [Notification Service](/src/notification-service) - Notification service that notifies both `Sender` and `Receiver` by updating/inserting transfer requests in the Public APIs state store.

## Prerequisites

Following technologies and CLIs are used for the development. Follow the links to install them:

- [Azure Developer CLI](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/make-azd-compatible?pivots=azd-create)
- [Dapr CLI](https://docs.dapr.io/getting-started/install-dapr-cli/)
- [Docker](https://docs.docker.com/get-docker/)
- [Kubernetes](https://kubernetes.io/docs/tasks/tools/)
- [Helm](https://helm.sh/docs/intro/install/)
- [Redis](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/)
- [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Spring Boot](https://spring.io/projects/spring-boot)

**Alternatively** you can use [DevContainers](https://code.visualstudio.com/docs/remote/containers) and [VS Code](https://code.visualstudio.com/) for local development. Opening the project with VS Code will automatically install all the required tools and extensions using DevContainers.

## Getting Started

We use [Make](https://www.gnu.org/software/make/manual/make.html) to automate the build and deployment process. You can run the following command to see the available commands:

```bash
make help
```

The following commands are available:

```bash
help                 💬 This help message :)
all-azure            🏃‍♀️ Run all the things in Azure
start-local          🧹 Setup local Kind Cluster
deploy-local         🚀 Deploy application resources locally
run-local            💿 Run app locally
port-forward-local   ⏩ Forward the local port
test                 🧪 Run tests, used for local development
clean                🧹 Clean up local files
dapr-dashboard       🔬 Open the Dapr Dashboard
dapr-components      🏗️  List the Dapr Components
deploy-azure         🚀 Deploy application resources in Azure
test-azure           🧪 Run tests in Azure
```

## Local Dev Environment Setup

Local environment is setup using [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/). Kind is a tool for running local Kubernetes clusters using Docker container "nodes". Kind was primarily designed for testing Kubernetes itself, but can be used for local development or CI.

### 1. Local Environment Setup

Running following command to setup your local development environment using `Docker` and `Kind`:

```bash
make start-local
```

This script runs the followings:

1. Creates a local Docker registry called `kind-registry` running locally on port 9999.
1. Creates a Kind cluster called `azd-aks` using config file from [kind-cluster-config.yaml](/local/kind-cluster-config.yaml).
1. Connects the registry to the cluster network if not already connected so deployments can access the local registry.
1. Maps the local registry to the cluster
1. Deploys [Redis](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/) to the cluster using [Helm](https://helm.sh/docs/intro/quickstart/) [chart](https://bitnami.com/stack/redis/helm) to use for different Dapr components (pub/sub, state store, etc).
1. Deploys [Dapr](https://docs.dapr.io/operations/hosting/kubernetes/kubernetes-deploy/) on your local cluster.
1. Deploys [Pub/Sub Broker](https://docs.dapr.io/developing-applications/building-blocks/pubsub/pubsub-overview/) using Redis as the message broker using [redis.yaml](./local/components/redis.yaml) Dapr component.

Your local cluster will be laid out as follows:

![Local](/docs/local.drawio.png)

### 2. Dapr Dashboard & Components

You can validate that the setup finished successfully by navigating to <http://localhost:9000>. This will open the [Dapr dashboard](/docs/dapr-dashboard.png) in your default browser. This assumes you installed [Dapr CLI](https://docs.dapr.io/getting-started/install-dapr-cli/) in your local machine.

```bash
make dapr-dashboard
```

To verify the installation of pub/sub broker and other components:

```bash
make dapr-components
```

This will output something similar to the following:

```bash
NAMESPACE  NAME                   TYPE          VERSION  SCOPES  CREATED              AGE
default    money-transfer-pubsub  pubsub.redis  v1               2023-05-10 11:25.24  10m
default    money-transfer-state   state.redis   v1               2023-05-10 11:25.24  10m
```

### 3. Deploy Services to Cluster

By convention, every service under `/src` folder has 2 files:

1. [Dockerfile](/src/public-api-service/Dockerfile) - Dockerfile for building the service image.
1. [local-deploy.sh](/src/public-api-service/local-deploy.sh) - Script file to build, publish and deployment the latest code to local cluster as Docker image.

To deploy all services to the cluster, run the following command under `scripts` folder:

```bash
make deploy
```

The deployment script will do the following:

1. Build and publish the Docker image for each service.
1. Deploy the image to the local cluster.
1. Create a Kubernetes service for each service.
1. Register the service with Dapr.

The output of the local deployment of each service will be similar to following:

```bash
🤖  Starting local deployments...


🎖️  Deploying Public API Service


🛖  Releasing version: 2023.05.10.11.38.59


☢️  Attempting to delete existing deployment public-api-service

Error from server (NotFound): deployments.apps "public-api-service" not found

🏗️  Building docker image

[+] Building 0.0s (19/19) FINISHED
 => [internal] load 
 
 ............
 
 => exporting to image                                                                                                                0.0s
 => => exporting layers                                                                                                               0.0s
 => => writing image sha256:0ee9a3ad60209d914d772661a07d64b49c95780b4e43f58457ea86018632d8cd                                          0.0s
 => => naming to localhost:5001/public-api-service:2023.05.10.11.38.59                                                                0.0s

🚚  Pushing docker image to local registry

The push refers to repository [localhost:5001/public-api-service]
.......
0f706090ed95: Layer already exists
a8dd5239cafe: Layer already exists
2023.05.10.11.38.59: digest: sha256:562162f9bfac78b6f234c372748ef20ce2457288a7c625bcb50b2ba5ed751798 size: 1993

🚀  Deploying to cluster

service/public-api-service created
deployment.apps/public-api-service created

🎉  Deployment complete
```

You can check the deployment status of the services:

```bash
kubectl get pods -A
```

You should get an output similar to:

```bash
NAMESPACE            NAME                                            READY   STATUS    RESTARTS       AGE
dapr-system          dapr-dashboard-575df59d4c-mp262                 1/1     Running   0              172m
dapr-system          dapr-operator-676b7df68d-xwzjw                  1/1     Running   0              172m
dapr-system          dapr-placement-server-0                         1/1     Running   1 (171m ago)   172m
dapr-system          dapr-sentry-5f44fd7c9d-gjjcl                    1/1     Running   0              172m
dapr-system          dapr-sidecar-injector-c66df4c49-h645k           1/1     Running   0              172m
default              account-service-655448db67-vdc6n                2/2     Running   0              14m
default              custody-service-5b8656d84c-lgb8j                2/2     Running   0              13m
default              fraud-service-7dfcd56d86-hs7s6                  2/2     Running   0              15m
default              notification-service-7dfbb47b4f-sthln           2/2     Running   0              12m
default              public-api-service-5df9f84648-gb9t2             2/2     Running   0              15m
default              redis-master-0                                  1/1     Running   0              172m
default              redis-replicas-0                                1/1     Running   0              172m
default              redis-replicas-1                                1/1     Running   0              171m
default              redis-replicas-2                                1/1     Running   0              170m
kube-system          coredns-565d847f94-k5nc4                        1/1     Running   0              172m
kube-system          coredns-565d847f94-qx2tn                        1/1     Running   0              172m
kube-system          etcd-azd-aks-control-plane                      1/1     Running   0              172m
kube-system          kindnet-7rhdn                                   1/1     Running   0              172m
kube-system          kindnet-bbk74                                   1/1     Running   0              172m
kube-system          kindnet-nfgrd                                   1/1     Running   0              172m
kube-system          kube-apiserver-azd-aks-control-plane            1/1     Running   0              172m
kube-system          kube-controller-manager-azd-aks-control-plane   1/1     Running   0              172m
kube-system          kube-proxy-6s47w                                1/1     Running   0              172m
kube-system          kube-proxy-7mnzh                                1/1     Running   0              172m
kube-system          kube-proxy-kj8pn                                1/1     Running   0              172m
kube-system          kube-scheduler-azd-aks-control-plane            1/1     Running   0              172m
local-path-storage   local-path-provisioner-684f458cdd-wtmqh         1/1     Running   0              172m
```

### 4. Connecting to Public API Service

[Public API](/src/public-api-service) is deployed as `Loadbalancer` service type. In local cluster setup, it will not able to get public IP to connect.
Instead, you can access this service locally using the Kubectl proxy tool.

```bash
make port-forward-local
```

While this command is running, you can access the service at <http://localhost:8080>.

An example request to start a new transfer workflow is:

```curl
curl -X POST \
  http://localhost:8080/transfers \
  -H 'Content-Type: application/json' \
  -d '{
    "sender": "A",
    "receiver": "B",
    "amount": 100
}'
```

You can query the status of a transfer:

```curl
curl -X GET \
  http://localhost:8080/transfers/{transferId} \
  -H 'Content-Type: application/json'
```

### 5. Delete Local Cluster

If you'd like to create a clean state and start over, you can run the following command to delete the local cluster and all the resources associated with it.

```bash
make clean
```

## Implementation Status

- [X] Public API endpoint receives new money transfer request. [TRANSFER(Sender: A, Amount: 100, Receiver:B)]
- [X] Request is published to Redis (pub/sub)
- [X] Deposit workflow starts
  - [X] Fraud service checks the legitimacy of the operation and triggers [VALIDATED(Sender: A, Amount: 100, Receiver:B)]
  - [X] Account service checks if `Sender` has enough funds and triggers [APPROVED(Sender: A, Amount: 100, Receiver: B)]
  - [ ] Notification services notifies both `Sender` and `Receiver`.
- [X] Public API can be used to check if there is a confirmation of the money transfer request.

## How To

Following are practical operator guides for common tasks.

### How to Operate Redis

To connect to Redis, you can use the following command:

```bash
kubectl exec -it redis-master-0 -- redis-cli
```

To get Redis password:

```bash
kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 --decode
```

To AUTH with Redis instance for further operations

```bash
AUTH <password>
```

Get all keys in the state store:

```bash
keys *
```

## Azure Deployment [WIP]

It's possible to deploy this application through Azure Developer (azd) CLI. azd is responsible for resource provision and application deployment requiring minimal interaction with azure in a developer point of view.

**The option for deploying everything with `azd up` is still a work in progress.** Currently a manual flow with `azd provision` and `azd deploy` is still required, as DAPR and Redis have to be installed in the cluster prior to the services to be deployed.

For more information of azd CLI: [Make your project azd-compatible](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/make-azd-compatible?pivots=azd-create)

### Pre-requisites

The following are available in the devContainer environment:

- Azure Developer CLI
- Azure CLI with K8s-extension enabled

The following have to be registered in the azure subscription after logining in

- AKS-ExtensionManager and AKS-Dapr features
- Kubernetes and ContainerService resource providers

First, make sure you have signed in to Azure CLI:

```bash
az login
```

Then, set the subscription you want to use:

```bash
az account set --subscription <subscription-id>
```

When your subscription is set, you can check the current registration status of the features and resource providers:

```bash
az feature list --query "[?name=='Microsoft.ContainerService/AKS-Dapr' || name=='Microsoft.ContainerService/AKS-ExtensionManager'].{Name:name,State:properties.state}" --output table
```

Then, register the features and resource providers:

```bash
# register AKS-ExtensionManager and AKS-Dapr resource provider
az feature register --namespace "Microsoft.ContainerService" --name "AKS-ExtensionManager"
az feature register --namespace "Microsoft.ContainerService" --name "AKS-Dapr"

# invoke the resource providers for Kubernetes and ContainerService to propagate the registration
az provider register --namespace Microsoft.KubernetesConfiguration
az provider register --namespace Microsoft.ContainerService
```

### 1. Initiate azd environment

```bash
azd init
```

When issuing the command above, azd asks for an azure subscription, environment name and geo-location. It also adds folders to your project to store the environment variables it creates along the deployment process

```txt
├── .azure                     [ For storing Azure configurations]
│   └── <your environment>     [ For storing all environment-related configurations]
│      ├── .env                [ Contains environment variables ]
│      └── config.json         [ Contains environment configuration ]
```

### 2. Provision of Resources

To deploy all the resources needed for this sample, run the following

```bash
azd provision
```

This will set up a dedicated resource group in your subscription with a clean AKS cluster, a Container Registry for the cluster to pull images from, and a Keyvault for secrets' storage (needed for the AKS deployed through azd).

### 3. Install Dapr and Redis on the Cluster

The variables used on the following commands are generated by azd and stored in `.azure/<your_env_name>/.env`

To use these environment variables, you can run the following command:

```bash
source <(azd env get-values)
```

```bash
# Install DAPR on AKS Cluster
az k8s-extension create --cluster-type managedClusters \
  --cluster-name $AZURE_AKS_CLUSTER_NAME \
  --resource-group $AZURE_RESOURCE_GROUP_NAME \
  --name myDaprExtension \
  --extension-type Microsoft.Dapr
```

To connect to the cluster and run other steps, update your current context of `kubectl` by running the following command:

```bash
az aks get-credentials --resource-group $AZURE_RESOURCE_GROUP_NAME --name $AZURE_AKS_CLUSTER_NAME
```

To check if Dapr is installed, run the following command:

```bash
kubectl get pods -n dapr-system
```

This will output something like this:

```bash
NAME                                    READY   STATUS    RESTARTS   AGE
dapr-dashboard-7798c78c74-4ddw5         1/1     Running   0          5m9s
dapr-operator-8955455bf-6qs79           1/1     Running   0          5m9s
dapr-placement-server-0                 1/1     Running   0          5m9s
dapr-sentry-69c8464dc9-6rtkf            1/1     Running   0          5m9s
dapr-sidecar-injector-fcbc4b979-wcjmj   1/1     Running   0          5m9s
```

We will deploy the application to its own namespace. Run following command to create a namespace for the application:

```bash
kubectl create namespace $AZURE_ENV_NAME
```

Check if the namespace is created:

```bash
kubectl get namespace
```

This will output something like this:

```bash
NAME                                  STATUS   AGE
dapr-system                           Active   6m48s
default                               Active   122m
gatekeeper-system                     Active   121m
java-banking-pubsub-dapr-sample-dev   Active   25s
kube-node-lease                       Active   122m
kube-public                           Active   122m
kube-system                           Active   122m
```

We are using an in-cluster Redis instance for the pub-sub and state store components. To deploy Redis, run the following command:

```bash
kubectl apply -f ./infra/redis.yaml --namespace $AZURE_ENV_NAME --wait=true
```

### 4. Deploy PubSub and State Store components

```bash
# Deploy pub-sub broker component backed by Redis
kubectl apply -f ./local/components/pubsub.yaml --wait=true --namespace $AZURE_ENV_NAME

# Deploy state store component backed Redis
kubectl apply -f ./local/components/state.yaml --wait=true --namespace $AZURE_ENV_NAME
```

### 5. Final step: Deploy micro-services

```bash
azd deploy
```

This command deploys all services defined in `azure.yaml` files in the namespace defined by the environment name, based on their individual deployment manifests.

At this point, you have everything configured in your Azure environment. From here, you can start interacting with the application.
