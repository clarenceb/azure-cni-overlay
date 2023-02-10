Azure CNI Overlay demo
======================

During preview:

```sh
az extensions install --name aks-preview
az extensions update --name aks-preview

az feature register --namespace "Microsoft.ContainerService" --name "AzureOverlayPreview"
az feature show --namespace "Microsoft.ContainerService" --name "AzureOverlayPreview"
# Wait until "Registered"
az provider register --namespace Microsoft.ContainerService
``

```sh
clusterName="aks-cni-overlay"
resourceGroup="aks-demos"
location="australiaeast"
vnet="aksvnet-overlay"

# Create the resource group
az group create --name $resourceGroup --location $location
```

Set up the VNET and subnets for 2 node pools and a shared pod subnet for all pods in the cluster:

```sh
# Create our vnet  (10.0.0.0/8)
az network vnet create \
    -g $resourceGroup \
    --location $location \
    --name $vnet \
    --address-prefixes 10.0.0.0/8 \
    -o none

# Create our linux node pool (will also be the system node pool - you can split this out from a linux user pool)
# A /24 will support 251 nodes (256 - 5 azure reserved addresses)
az network vnet subnet create \
    -g $resourceGroup \
    --vnet-name $vnet \
    --name linuxnp1 \
    --address-prefixes 10.0.0.0/24 \
    -o none 

# Create a windows nodepool
# A /26 will support 59 nodes (64 - 5 azure reserved addresses)
az network vnet subnet create \
    -g $resourceGroup \
    --vnet-name $vnet \
    --name windowsnp1 \
    --address-prefixes 10.0.1.0/26 \
    -o none
```

Create the cluster with the iniital node pool using subnet 1 and the shared pod subnet:

```sh
subscription=$(az account show --query id -o tsv)
aksVersion=$(az aks get-versions -l $location --query orchestrators[*].orchestratorVersion -o tsv | sort -nr | head -n1)

az aks create -n $clusterName -g $resourceGroup -l $location \
    --max-pods 250 \
    --node-count 2 \
    --network-plugin azure \
    --vnet-subnet-id /subscriptions/$subscription/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet/subnets/linuxnp1 \
    -k $aksVersion \
    --service-cidr 10.0.1.64/26 \
    --dns-service-ip 10.0.1.74 \
    --generate-ssh-keys \
    --network-plugin-mode overlay \
    --pod-cidr 192.0.0.0/14

# Note: Pod CIDR cannot overlap the AKS cluster VNET CIDR
```

Add the Windows node pool using subnet 2 and the shared pod subnet:

```sh
az aks nodepool add --cluster-name $clusterName -g $resourceGroup  -n newnodepool \
    --max-pods 250 \
    --node-count 2 \
    --vnet-subnet-id /subscriptions/$subscription/resourceGroups/$resourceGroup/providers/Microsoft.Network/virtualNetworks/$vnet/subnets/windowsnp1 \
    -k $aksVersion \
    --os-type Windows \
    --os-sku Windows2022 \
    --name npwin
```

Check the IPs allocated to the nodes and pods and other network configuration:

```sh
az aks get-credentials -n $clusterName -g $resourceGroup
kubectl config use-context aks-cni-overlay

kubectl get node -o wide
# 10.0.0.x  -> Linux nodes - linuxnp1 subnet (10.0.0.0/24)
# 10.0.1.x  -> Windows nodes - windowsnp1 subnet (10.0.1.0/26)

kubectl get pod -o wide -n kube-system
# 10.0.x.x  -> pods on linux hosts with hostNetwork=true (e.g. daemonsets) - linuxnp1 subnet (10.0.0.0/24)
# 192.0.x.x  -> pods on linux hosts - pod subnet (192.0.0.0/14)

az aks show -n $clusterName -g $resourceGroup --query networkProfile.serviceCidr -o tsv
# 10.0.1.64/26 -> service CIDR

az aks show -n $clusterName -g $resourceGroup --query networkProfile.dnsServiceIp -o tsv
# 10.0.1.74    -> DNS service IP

az aks show -n $clusterName -g $resourceGroup --query networkProfile.podCidr -o tsv
# 192.0.0.0/14 -> Pod CIDR

az aks nodepool show --cluster-name $clusterName -g $resourceGroup -n nodepool1 --query maxPods
# 250

az aks nodepool show --cluster-name $clusterName -g $resourceGroup -n npwin --query maxPods
# 250
```

Get counts of available IPs in the subnets:

```sh
subnetAvailableIps() {
    subnet_name=$1
    vnet=$2
    netmask=$3
    resourceGroup=$4

    max_ip=$((2 ** (32-$netmask)-5))
    used_ip=$(az network vnet subnet show -g $resourceGroup -n $subnet_name --vnet-name $vnet -o json | jq  ".ipConfigurations[].id" | wc -l)
    available_ip=$(($max_ip - $used_ip))

    echo "There are $available_ip IPs available in subnet: $subnet_name"
}

subnetAvailableIps "linuxnp1" "$vnet" "24" "$resourceGroup"
subnetAvailableIps "windowsnp1" "$vnet" "26" "$resourceGroup"
```

Run a Linux container and a Windows container, test connectivity:

```sh
kubectl run --image ubuntu ubuntu  --overrides='{"spec": { "nodeSelector": {"kubernetes.io/os": "linux"}}}' -- sleep 1000000
kubectl run --image mcr.microsoft.com/dotnet/framework/samples:aspnetapp aspnetapp --overrides='{"spec": { "nodeSelector": {"kubernetes.io/os": "windows"}}}'

watch -n 2 kubectl get pod -o wide

kubectl port-forward pod/aspnetapp 8080:80

# Test http://localhost:8080

kubectl get pod -o wide
# Note the windows pod IP

kubectl exec -ti ubuntu -- bash
apt update
apt install -y curl

curl -i http://<ip-addr-of-windows-container>
# HTTP/1.1 200 OK
# ...
exit

kubectl get pod -o wide -A | grep "192.0." | wc -l
# 15
```

Test connectivity from a node to a pod:

```sh
# See: https://krew.sigs.k8s.io/docs/user-guide/quickstart/
kubectl krew install ssh-jump
kubectl get node -o wide
kubectl ssh-jump <choose-one-of-the-linux-nodes> -i ~/.ssh/id_rsa -u azureuser

curl --connect-timeout 3 -i http://<ip-addr-of-windows-container>
# curl: (28) Connection timeout after 3001 ms
# ...
exit
```

Hmm, we can't connect to the pod from the node.

Try creating a loadbalancer service for the pod and testing it from outside the cluster:

```sh
kubectl expose pod aspnetapp --type=LoadBalancer --port=80 --target-port=80

kubectl get svc
#NAME         TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
#aspnetapp    LoadBalancer   10.0.1.107   x.x.x.x         80:31216/TCP   30s

curl --connect-timeout 3 -i http://<external-svc-ip>
#HTTP/1.1 200 OK
#...
```

Cleanup:

```sh
kubectl delete pod aspnetapp
kubectl delete pod ubuntu
kubectl delete service aspnetapp
kubectl delete pod sshjump
```
