# Software Containerization Project
This document will describe how to install the project application for the software containerization course. The project application is a simple CRUD style application that consists of a database for persistant storage, a REST API as to interact with the database and a web application which serves as the user-facing front end of the application.

The application is deployed in GKE and makes use of a single helm chart containing multiple sub charts and dependencies.


## Objective

The following table lists all the requirements as specified by the project on the canvas page:

| TASKS | SOLUTION |
| ------ | ------ |
| Create a Deployment/Statefulset, a Service and a Persistent Volume, Persistent Volume Claim for a database of your choice. You can use a pre-built docker image for this database, or a pre-built Helm chart that you package as a sub.chart in your chart. Ensure that the Service exposed by the database is such that it can be accessed by the REST API, but not by users outside of the cluster. Ensure that the configuration of the database makes use of ConfigMaps and Secrets appropriately. | [DB Chart](./charts/postgresql-10.2.6.tgz) |
| Create a deployment for a REST API of your choice, that uses the database. It is expected that you create your own Dockerfile for this image. Ensure that the application can scale horizontally. Create an appropriate type of Service such that the REST API can be accessed by the Web Application. Depending on where the Web Application rune, this might require the Web API to be exposed outside of the cluster. | [REST API Chart](./charts/api) |
| Create a Deployment for a Web application of your choice, that uses the REST API. It is expected that you create your own Dockerfile for this image. Ensure that the application can scale horizontally. Create an appropriate type of Service/Ingress/API Gateway, such that the application can be accessed by users outside of the cluster.| [Web Application Chart](./charts/frontend) |
| Configure TLS for the web application. You can use self-signed certificates or certificates signed by a self-made certificate authority. You can use cert-manager and/or openssl to generate the certificates. Ensure that if the REST API is accessible outside of the Cluster, it is running on https as well. Exposing http endpoints that redirect to https is fine.| [cert-manager](installing-cert-manager) [Cert](./templates/certificate.yaml) [Issuer](./templates/issuer.yaml) |
| Configure ServiceMesh (Istio). If you use the Istio Ingress Gateway, it's ok not to configure Kubernetes NGINX Ingress as it replaces it.| [Install Istio](installing-istio) |
| Configure application monitoring with Prometheus. In the presentation, you need to show how you query Prometheus. | [Install](#installing-prometheus) [Query](#querying-in-grafana) |
| Run the Application on Google Cloud Platform. | [Create GKE Cluster](#creating-the-gke-cluster) |


## Requirements
[gcp account](https://console.cloud.google.com/?hl=nl)
[gcloud cli](https://cloud.google.com/sdk/gcloud)
[helm](https://helm.sh/)

### Helm repositories

Below are the custom Helm repositories that are used for this project and which charts are used from that repository.

| REPO | CHART |
| ------ | ------ |
| https://charts.jetstack.io | cert-manager |
| https://charts.bitnami.com/bitnami | postgresql |
| https://prometheus-community.github.io/helm-charts | kube-prometheus-stack |

```sh
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo add https://charts.bitnami.com/bitnami
$ helm repo add https://prometheus-community.github.io/helm-charts
```


### Creating the GKE Cluster

```sh
$ export PROJECT=project-name
$ gcloud beta container \
    --project $PROJECT \
    clusters create "project" \
    --zone "europe-west4-a" \
    --no-enable-basic-auth \
    --cluster-version "1.18.12-gke.1205" \
    --release-channel "regular" \
    --machine-type "e2-medium" \
    --image-type "COS" \
    --disk-type "pd-standard" \
    --disk-size "20" \
    --metadata disable-legacy-endpoints=true \
    --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
    --preemptible \
    --num-nodes "3" \
    --enable-stackdriver-kubernetes \
    --enable-ip-alias \
    --network "projects/software-containerisation/global/networks/default" \
    --subnetwork "projects/software-containerisation/regions/europe-west4/subnetworks/default" \
    --default-max-pods-per-node "110" \
    --enable-autoscaling \
    --min-nodes "0" \
    --max-nodes "4" \
    --no-enable-master-authorized-networks \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver \
    --enable-autoupgrade \
    --enable-autorepair \
    --max-surge-upgrade 1 \
    --max-unavailable-upgrade 0 \
    --enable-shielded-nodes
```

## Installing Istio
Create a namespace istio-system for Istio components:
```sh
$ kubectl create namespace istio-system
```

Install the Istio base chart which contains cluster-wide resources used by the Istio control plane:
```sh
$ helm install -n istio-system istio-base manifests/charts/base
```

Install the Istio discovery chart which deploys the istiod service:
```sh
$ helm install --namespace istio-system istiod manifests/charts/istio-control/istio-discovery \
    --set global.hub="docker.io/istio" --set global.tag="1.8.2"
```

Install the Istio ingress gateway chart which contains the ingress gateway components:
```sh
$ helm install --namespace istio-system istio-ingress manifests/charts/gateways/istio-ingress \
    --set global.hub="docker.io/istio" --set global.tag="1.8.2"
```

Verify installation:
```sh
$ kubectl get pods -n istio-system
```

## Installing Prometheus
Add the required Helm repository
```sh
$ helm repo add https://prometheus-community.github.io/helm-charts
```

Installing prometheus in a seperate namespace:
```sh
$ helm install prometheus \
  --create-namespace \
  --namespace=prometheus \
  prometheus-community/kube-prometheus-stack
```

Verify installation:
```sh
kubectl --namespace prometheus get pods -l "release=prometheus"
```

### Accessing Grafana
You can access the Grafana dashboard by portforwarding to the service:
```sh
$ kubectl -n prometheus port-forward svc/prometheus-grafana 8000:80
```
The default credentials are:
| Username | Password |
| ------ | ------ |
| admin | prom-operator |


## Installing Cert-Manager
Add the required helm repository:
```sh
$ helm repo add https://charts.jetstack.io
```

Install cert manager:
```sh
$ helm install \
  cert-manager jetstack/cert-manager \
  --create-namespace \
  --namespace cert-manager \
  --version v1.1.0 \
  --set installCRDs=true
```

Verify installation:
```sh
kubectl --namespace prometheus get pods -l "release=prometheus"
```

# Installing Application

Create a namespace for the application:
```sh
$ export RELEASE=sc
$ export NAMESPACE=project
$ kubectl create namespace $NAMESPACE
```

Allow Istio sidecar injection for this namespace:
```sh
$ kubectl label namespace $NAMESPACE istio-injection=enabled
```

Install the application:
```sh
helm install $RELEASE --namespace=$NAMESPACE $PWD
```

The following table lists all the requirements as specified by the project on the canvas page:

| TASKS | SOLUTION |
| ------ | ------ |
| Create a Deployment/Statefulset, a Service and a Persistent Volume, Persistent Volume Claim for a database of your choice. You can use a pre-built docker image for this database, or a pre-built Helm chart that you package as a sub.chart in your chart. Ensure that the Service exposed by the database is such that it can be accessed by the REST API, but not by users outside of the cluster. Ensure that the configuration of the database makes use of ConfigMaps and Secrets appropriately. | NIL |
| Create a deployment for a REST API of your choice, that uses the database. It is expected that you create your own Dockerfile for this image. Ensure that the application can scale horizontally. Create an appropriate type of Service such that the REST API can be accessed by the Web Application. Depending on where the Web Application rune, this might require the Web API to be exposed outside of the cluster. | NIL |

## Requirements
[gcp account](https://console.cloud.google.com/?hl=nl)
[gcloud cli](https://cloud.google.com/sdk/gcloud)
[helm](https://helm.sh/)

### Helm repositories

Below are the custom Helm repositories that are used for this project and which charts are used from that repository.

| REPO | CHART |
| ------ | ------ |
| https://charts.jetstack.io | cert-manager |
| https://charts.bitnami.com/bitnami | postgresql |
| https://prometheus-community.github.io/helm-charts | kube-prometheus-stack |

```sh
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo add https://charts.bitnami.com/bitnami
$ helm repo add https://prometheus-community.github.io/helm-charts
```


### Creating the GKE Cluster

```sh
$ export PROJECT=project-name
$ gcloud beta container \
    --project $PROJECT \
    clusters create "project" \
    --zone "europe-west4-a" \
    --no-enable-basic-auth \
    --cluster-version "1.18.12-gke.1205" \
    --release-channel "regular" \
    --machine-type "e2-medium" \
    --image-type "COS" \
    --disk-type "pd-standard" \
    --disk-size "20" \
    --metadata disable-legacy-endpoints=true \
    --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
    --preemptible \
    --num-nodes "3" \
    --enable-stackdriver-kubernetes \
    --enable-ip-alias \
    --network "projects/software-containerisation/global/networks/default" \
    --subnetwork "projects/software-containerisation/regions/europe-west4/subnetworks/default" \
    --default-max-pods-per-node "110" \
    --enable-autoscaling \
    --min-nodes "0" \
    --max-nodes "4" \
    --no-enable-master-authorized-networks \
    --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver \
    --enable-autoupgrade \
    --enable-autorepair \
    --max-surge-upgrade 1 \
    --max-unavailable-upgrade 0 \
    --enable-shielded-nodes
```

## Installing Istio
Create a namespace istio-system for Istio components:
```sh
$ kubectl create namespace istio-system
```

Install the Istio base chart which contains cluster-wide resources used by the Istio control plane:
```sh
$ helm install -n istio-system istio-base manifests/charts/base
```

Install the Istio discovery chart which deploys the istiod service:
```sh
$ helm install --namespace istio-system istiod manifests/charts/istio-control/istio-discovery \
    --set global.hub="docker.io/istio" --set global.tag="1.8.2"
```

Install the Istio ingress gateway chart which contains the ingress gateway components:
```sh
$ helm install --namespace istio-system istio-ingress manifests/charts/gateways/istio-ingress \
    --set global.hub="docker.io/istio" --set global.tag="1.8.2"
```

Verify installation:
```sh
$ kubectl get pods -n istio-system
```

## Installing Prometheus
Add the required Helm repository
```sh
$ helm repo add https://prometheus-community.github.io/helm-charts
```

Installing prometheus in a seperate namespace:
```sh
$ helm install prometheus \
  --create-namespace \
  --namespace=prometheus \
  prometheus-community/kube-prometheus-stack
```

Verify installation:
```sh
kubectl --namespace prometheus get pods -l "release=prometheus"
```

### Accessing Grafana
You can access the Grafana dashboard by portforwarding to the service:
```sh
$ kubectl -n prometheus port-forward svc/prometheus-grafana 8000:80
```
The default credentials are:
| Username | Password |
| ------ | ------ |
| admin | prom-operator |

### Querying in Grafana
Follow the steps above to get into Grafana and proceed to go to the "Explore" section. From here you can run queries and access Prometheus metrics such as ```prometheus_http_requests_total```.

![Query Example](./images/query.png?raw=true)



## Installing Cert-Manager
Add the required helm repository:
```sh
$ helm repo add https://charts.jetstack.io
```

Install cert manager:
```sh
$ helm install \
  cert-manager jetstack/cert-manager \
  --create-namespace \
  --namespace cert-manager \
  --version v1.1.0 \
  --set installCRDs=true
```

Verify installation:
```sh
kubectl --namespace prometheus get pods -l "release=prometheus"
```

# Installing Application

Create a namespace for the application:
```sh
$ export RELEASE=sc
$ export NAMESPACE=project
$ kubectl create namespace $NAMESPACE
```

Allow Istio sidecar injection for this namespace:
```sh
$ kubectl label namespace $NAMESPACE istio-injection=enabled
```

Install the application:
```sh
helm install $RELEASE --namespace=$NAMESPACE $PWD
```