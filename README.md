# Kubernetes Demo with MongoDB, Prometheus, and Grafana

This demo is for a local setup only, running K8S on a Mac, Docker Desktop.  It shows you how to monitor a third party application, like MongoDB, in your Kubernetes cluster using Prometheus monitoring with Grafana as the visualization tool.

## Visuals

![Prometheus screen shot](https://hedgehoghs.com/wp-content/uploads/2023/11/prometheus_2023-11-15.png?raw=true)
![Grafana screen shot](https://hedgehoghs.com/wp-content/uploads/2023/11/grafana_2023-11-15.png?raw=true)
![Mongo Express screen shot](https://hedgehoghs.com/wp-content/uploads/2023/11/mongo-express_2023-11-15.png?raw=true)

## Installation

Create a base64 encoded username and password for MongoDB:
```shell
echo -n 'username' | base64
echo -n 'password' | base64
```

Use something other than 'username' and 'password' of course.

Create a secret file: `mongo-secret.yaml`

Put the base64 encoded values into a `mongo-secret.yaml` file in the root directory:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  mongo-root-username: 
  mongo-root-password: 
```

Run the apply commands for mongo setup:
```shell
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo.yaml
kubectl apply -f mongo-configmap.yaml 
kubectl apply -f mongo-express.yaml
```

Create a `monitoring` namespace if you do not already have one:
```shell
kubectl create namespace monitoring
```

Install [Helm](https://helm.sh/docs/intro/install/)

Setup Prometheus and Grafana with Helm:
```shell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -f kube-prometheus-stack-values.yaml
kubectl --namespace monitoring get pods
```

You may need this patch if running Docker Desktop on a Mac:
```shell
kubectl patch daemonset.apps/kube-prometheus-stack-prometheus-node-exporter --type "json" -p '[{"op": "remove", "path" : "/spec/template/spec/containers/0/volumeMounts/2/mountPropagation"}]'
```

Port forward Prometheus to access it as a service
```shell
kubectl port-forward service/kube-prometheus-stack-prometheus 9090
```

Port forward Grafana to access it from localhost:
```shell
kubectl port-forward deployment/kube-prometheus-stack-grafana 3000
```

Port forward mongo express to access it from localhost:
```shell
kubectl port-forward service/mongo-express-service 8081
```

Now install and port forward the MongoDB exporter to collect metrics from mongodb:

```shell
helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter -f mongo-exporter-values.yaml
```

Port forward if you want to check the /metrics endpoint:
```shell
kubectl port-forward service/mongodb-exporter-prometheus-mongodb-exporter 9216
```

## Usage
Navigate to the port forwarded URL for grafana: http://127.0.0.1:3000

Default login for grafana:
Username: admin
Password: prom-operator

Navigate to Prometheus and check targets: http://127.0.0.1:9090/targets

Navigate to Mongo Express and make some changes: http://127.0.0.1:8081/

Of course, this demo is not stateful, so there is no backend persistent disk for MongoDB.  As such, any MongoDB changes will be gone if the cluster or pod restarts.

## Tear Down
Tear down prometheus and grafana:
```shell
helm uninstall mongodb-exporter 
helm uninstall kube-prometheus-stack
```