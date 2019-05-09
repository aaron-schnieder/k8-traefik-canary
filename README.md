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

helm install --name traefik-hello-kube --namespace kube-system --set dashboard.enabled=true,dashboard.domain=traefik-ui.acs stable/traefik
```

In prod, just install the traefik ingress controller but don't enable the dashboard

```

helm install stable/traefik --name traefik-hello-kube --namespace kube-system
```

Get the local resource name of the traefik helm chart that was installed

```
helm list
```

Now view the status of traefik on the k8 cluster using the local resource name

```
helm status traefik-hello-kube
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

Deploy ingress and ensure it is running

```
kubectl apply -f deployment/ingress.yaml
kubectl get ingress
```

Deploy service and ensure it is running

```
kubectl apply -f deployment/service.yaml
kubectl get svc
```

Deploy pods and ensure they are running

```
kubectl apply -f deployment/deployment-stable.yaml
kubectl apply -f deployment/deployment-canary.yaml
kubectl get pods
kubectl describe pod <podname from get pods output>
```

Get the external IP for the traefik ingress controller

```
kubectl describe service traefik-hello-kube -n kube-system | grep Ingress
<copy the LoadBalancer Ingress: value to get the external IP>
```


Execute a bunch of HTTP GET requests to view requests splitting between canary and stable versions

```

for ((i=1;i<=30;i++)); do curl -s "http://<loadbalancer ingress ip>/" | grep version: ; done
```

Clean up resources in cluster
```

kubectl delete -f service.yaml
kubectl delete -f deployment-canary.yaml
kubectl delete -f deployment-stable.yaml
helm delete traefik-hello-kube
```

Portworx config and deployment
- K8s storage volumes are temporary storage per pod
- K8s persistent volumes exist beyond the lifecyle of a pod and can be claimed by another pod
- K8s uses etcd as a distributed key/value store by default in clusters
- etcdctl is the CLI used to interact with etcd
- kubectl apply -f deployment/portworx-spec.yaml
- kubectl get pods,svc -n kube-system

Debugging
- kubecel logs -n kube-system <pod-name>
- https://docs.portworx.com/portworx-install-with-kubernetes/operate-and-maintain-on-kubernetes/troubleshooting/troubleshoot-and-get-support/#useful-commands
- kubectl describe pod/px-lighthouse-d7949d765-9fln9 -n kube-system
- Shell into running pod:
- kubectl exec -it <pod-name> -- /bin/bash
- k exec -it px-lighthouse-6b66ddfd98-lg46b -c config-init -n kube-system /bin/bash
- kubectl describe pod/etcd-docker-for-desktop -n kube-system
- For Portworx pods, you need to specify a container with -c [px-lighthouse config-sync stork-connector] or one of the init containers: [config-init]
- kubectl logs -n kube-system px-lighthouse-6b66ddfd98-lg46b -c px-lighthouse

Azure Setup
- Create SP
- az ad sp create-for-rbac --skip-assignment
```
{
  "appId": "9ac5c520-4e64-445a-a038-237bb0956696",
  "displayName": "azure-cli-2019-05-08-03-44-09",
  "name": "http://azure-cli-2019-05-08-03-44-09",
  "password": "b772a418-80a3-4579-839d-404d0372a592",
  "tenant": "72f988bf-86f1-41af-91ab-2d7cd011db47"
}
```
- Create resource group
- az group create -l westus2 -n aaros-aks-portworx
- Create AKS cluster
- az aks create \
    --resource-group aaros-aks-portworx \
    --name aarosAksCluster \
    --node-count 3 \
    --service-principal "9ac5c520-4e64-445a-a038-237bb0956696" \
    --client-secret "b772a418-80a3-4579-839d-404d0372a592" \
    --generate-ssh-keys
- Connect to AKS cluster and get credentials
- az aks get-credentials --resource-group aaros-aks-portworx --name aarosAksCluster
- kubectl get nodes
- Create a static Azure Disk (single Pod access only) in the same resource group as the AKS cluster
- https://docs.microsoft.com/en-us/azure/aks/azure-disk-volume
- az disk create \
  --resource-group aaros-aks-portworx \
  --name aarosPwDataDisk  \
  --size-gb 20 \
  --sku Premium_LRS \
  --query id --output tsv
- /subscriptions/2a6b40d9-16cb-4676-8609-d5a1df110803/resourceGroups/aaros-aks-portworx/providers/Microsoft.Compute/disks/aarosPwDataDisk
```
- Make sure that the appropriate storage class has been deployed to the AKS cluster for your Azure Disk type
- kubectl get sc
- *If the storage class does not exist on the AKS cluster*, edit azure-storage-class.yaml and apply the Azure storage class to the AKS cluster
- kubectl apply -f deployment/azure-storage-class.yaml
```
- Apply the persistent volume claim to the storage class
- kubectl apply -f deployment/azure-pv-claim.yaml
- persistentvolumeclaim/azure-managed-disk
- Get K8s version
- kubectl version --short | awk -Fv '/Server Version: / {print $3}'

https://docs.microsoft.com/en-us/azure/aks/ssh
az vm user update \
  --resource-group MC_aaros-aks-portworx_aarosAksCluster_westus2 \
  --name aks-nodepool1-17636774-0 \
  --username azureuser \
  --ssh-key-value ~/.ssh/id_rsa.pub
  az vm list-ip-addresses --resource-group MC_aaros-aks-portworx_aarosAksCluster_westus2 -o table
  kubectl run -it --rm aks-ssh --image=debian
  apt-get update && apt-get install openssh-client -y
  kubectl get pods
  kubectl cp ~/.ssh/id_rsa aks-ssh-589f4659c5-jdcqr:/id_rsa
  chmod 0600 id_rsa
  ssh -i id_rsa azureuser@10.240.0.4

https://docs.microsoft.com/en-us/azure/virtual-machines/linux/attach-disk-portal?toc=%2Fazure%2Fvirtual-machines%2Flinux%2Ftoc.json
dmesg | grep SCSI
sudo fdisk /dev/sdc
n
p
p
w
sudo mkfs -t ext4 /dev/sdc1
sudo mkdir /portworxdrive
sudo mount /dev/sdc1 /portworxdrive
sudo -i blkid
sudo vi /etc/fstab
UUID=d0bfbc13-b24a-436b-a5f1-b7c91ae885a0   /portworxdrive   ext4   defaults,nofail   1   2
