# Welcome to a demo for KEDA http autoscaling with OpenServiceMesh
Demo KEDA http autoscaling with OSM metrics and bookstore application. This will walk through the steps for doing it with an AKS (Azure Kubernetes Services) cluster and BYO Prometheus. It will also use the Contour Ingress Controller with OSM integration of Envoy.

### Deploy an Azure Kubernetes Cluster with azCLI
Use this [link](https://docs.microsoft.com/en-us/azure/aks/configure-azure-cni#configure-networking---cli)

### Install OpenServiceMesh

```
az aks enable-addons --addons open-service-mesh -g 'resource_group' -n 'aks_cluster-name'

az aks show -g 'resource_group' -n 'aks_cluster-name'  --query 'addonProfiles.openServiceMesh.enabled'
```

### Verify the status of OSM in kube-system namespace

```
kubectl get deployments -n kube-system --selector app.kubernetes.io/name=openservicemesh.io
kubectl get pods -n kube-system --selector app.kubernetes.io/name=openservicemesh.io
kubectl get services -n kube-system --selector app.kubernetes.io/name=openservicemesh.io
```

### Installing Prometheus via helm chart kube-prometheus-stack

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus \
prometheus-community/kube-prometheus-stack -f https://raw.githubusercontent.com/javierromancsa/aks-keda-http-osm-autoscaling/main/byo_values.yaml \
--namespace monitoring \
--create-namespace
```

### Disabled metrics scapping from components that AKS don't expose.

```
helm upgrade prometheus \
prometheus-community/kube-prometheus-stack -f https://raw.githubusercontent.com/javierromancsa/aks-keda-http-osm-autoscaling/main/byo_values.yaml \
--namespace monitoring \
--set kubeEtcd.enabled=false \
--set kubeControllerManager.enabled=false \
--set kubeScheduler.enabled=false

```

### Intalling OSM CLI v0.11.1 

```
OSM_VERSION=v0.11.1
curl -sL "https://github.com/openservicemesh/osm/releases/download/$OSM_VERSION/osm-$OSM_VERSION-linux-amd64.tar.gz" | tar -vxzf -
sudo mv ./linux-amd64/osm /usr/local/bin/osm
sudo chmod +x /usr/local/bin/osm
```

### Deploying the demo application bookstore
Use this link[https://release-v1-0.docs.openservicemesh.io/docs/getting_started/install_apps/]

### Enabling OSM metrics for apps
  *osm metrics enable --namespace "app_namespace, app_namespace"
```
osm metrics enable --namespace "bookstore,bookbuyer,bookwarehouse"
```

### Portforward Prometheus in another new terminal and open http://localhost:9090 :
```
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090
```

### Query Prometheus server to get all http metrics from bookstore web application:
```
envoy_cluster_upstream_rq_xx{envoy_cluster_name="bookstore/bookstore|14001|local"]
```

### Query to obtain the rate of http request in a window of 1 minute from bookstore web application:

```
sum(irate(envoy_cluster_upstream_rq_xx{envoy_cluster_name="bookstore/bookstore|14001|local"}[1m]))
```

### Install KEDA
```
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
kubectl create namespace keda
helm install keda kedacore/keda --namespace keda
```

### Installing Contour in AKS:

link[https://projectcontour.io/getting-started/#option-2-helm]
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install mycontour bitnami/contour --namespace projectcontour --create-namespace

kubectl -n projectcontour get po,svc -w 
```

### Create HTTPProxy and ingressBackend for bookstore appplication
#### Get the public/External IP of the Azure loadbalancer created for the Contour Services
```
dns=".nip.io"
myip="$(kubectl -n projectcontour describe svc -l app.kubernetes.io/component=envoy | grep -w "LoadBalancer Ingress:"| sed 's/\(\([^[:blank:]]*\)[[:blank:]]*\)*/\2/')"
myip_dns=$myip$dns
```
#### Create HTTPProxy and ingressBackend
```
kubectl apply -f - <<EOF
apiVersion: projectcontour.io/v1
kind: HTTPProxy
metadata:
  name: bookstoreproxy
  namespace: bookstore
spec:
  virtualhost:
    fqdn: $myip_dns
  routes:
  - services:
    - name: bookstore
      port: 14001
---
kind: IngressBackend
apiVersion: policy.openservicemesh.io/v1alpha1
metadata:
  name: bookstorebackend
  namespace: bookstore
spec:
  backends:
  - name: bookstore
    port:
      number: 14001 # targetPort of httpbin service
      protocol: http
  sources:
  - kind: Service
    namespace: projectcontour
    name: mycontour-contour-envoy
```

### Create KEDA ScaledObject based on Query

```
kubectl apply -f https://raw.githubusercontent.com/javierromancsa/aks-keda-http-osm-autoscaling/main/keda_bookstore_http.yaml
```
