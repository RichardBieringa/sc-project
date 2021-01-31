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

# Installing Prometheus
helm install sc-prometheus prometheus-community/kube-prometheus-stack --namespace=prometheus

# Installing Cert-Manager
helm install \
  cert-manager jetstack/cert-manager \
  --create-namespace \
  --namespace cert-manager \
  --version v1.1.0 \
  --set installCRDs=true

# Installing Application
export NAMESPACE=sc
kubectl label namespace $NAMESPACE istio-injection=enabled
helm install $RELEASE_NAME --create-namespace --namespace=$NAMESPACE $PWD