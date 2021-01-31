# Software Containerization Project
This document will describe how to install the project application for the software containerization course. The project application is a simple CRUD style application that consists of a database for persistant storage, a REST API as to interact with the database and a web application which serves as the user-facing front end of the application.

The application is deployed in GKE and makes use of helm charts, TLS is done with a domain n

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


kubectl create namespace istio-system

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
export NAMESPACE=sc
kubectl label namespace $NAMESPACE istio-injection=enabled
helm install $RELEASE_NAME --create-namespace --namespace=$NAMESPACE $PWD