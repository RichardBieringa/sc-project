HELM REPOS:
helm repo add jetstack https://charts.jetstack.io
helm repo add https://charts.bitnami.com/bitnami
helm repo add https://prometheus-community.github.io/helm-charts

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
