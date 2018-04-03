# hello-kube
k8 canary deployment example using the traefik ingress controller.

## Pre-requisites

1. brew install kubernetes-helm
2. Azure CLI


## Working with Azure ACS cluster

```
az login
az account set -s <subscription id>
az acs kubernetes get-credentials --resource-group=my-resource-group --name=my-k8-cluster
```

Ensure you kubectl is connected to the corrrect k8 cluster

```
kubectl config view
```

Init helm, which will deploy the tiller agent pod that allows helm charts to be deployed to the cluster

```
helm init --upgrade
```

Ensure helm's tiller pod is deployed to your k8 cluster and running

```
kubectl get pods --namespace kube-system
```

## Install traefik ingress controller

Use the traefik helm chart to deploy traefik to your cluster

In dev / test, enable the traefik ingress controller dashboard to view routing
```

helm install --namespace kube-system --set dashboard.enabled=true,dashboard.domain=traefik-ui.acs stable/traefik
```

In prod, just install the traefik ingress controller but don't enable the dashboard

```

helm install --namespace kube-system stable/traefik
```

Get the local resource name of the traefik helm chart that was installed

```
helm list
```

Now view the status of traefik on the k8 cluster using the local resource name

```
helm status newbie-jaguar
```

You can also view the service and pods deployed via

```
kubectl get svc,pods --namespace kube-system
```

## Deployment

Build docker image for stable version and push to dockerhub

```
edit index.html and change version to stable
docker build -t schnieds/hello-kube:stable .
docker push schnieds/hello-kube:stable
```

Build docker image for canary version and push to dockerhub

```
edit index.html and change version canary
docker build -t schnieds/hello-kube:canary .
docker push schnieds/hello-kube:canary
```

Deploy service and ensure it is running

```
kubectl apply -f service.yaml
kubectl get svc
```

Deploy pods and ensure they are running

```
kubectl apply -f deployment-stable.yaml
kubectl apply -f deployment-canary.yaml
kubectl get pods
kubectl describe pod <podname from get pods output>
```

Get the external IP for the service load balancer

```
kubectl describe svc service-hello-kube
<copy the LoadBalancer Ingress: value to get the external IP>
```


Execute a bunch of HTTP GET requests to view requests splitting between canary and stable versions

```

for ((i=1;i<=30;i++)); do curl -s "http://13.91.242.211:8080/" | grep version: ; done
```

Clean up resources in cluster
```

kubectl delete -f service.yaml
kubectl delete -f deployment-canary.yaml
kubectl delete -f deployment-stable.yaml
```