## Demo 1

Cilium with AzureCNI=none and Helm

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm upgrade -i cilium cilium/cilium --set hubble.relay.enabled=true --set hubble.ui.enabled=true --namespace kube-system
```

```bash
#restart all pods
kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```


## Demo 2

Azure CNI with Cilium

```bash
RG=aks
LOCATION="West Central US"
VNET_NAME="Cilium"
PREFIX="10.0.0.0/8"
SUBNET_PREFIX="10.240.0.0/16"
CLUSTER_NAME=cilium
SUB=12c7e9d6-967e-40c8-8b3e-4659a4ada3ef


az network vnet create -g $RG --location $LOCATION --name $VNET_NAME --address-prefixes $PREFIX -o none 
az network vnet subnet create -g $RG --vnet-name $VNET_NAME --name nodesubnet --address-prefixes $SUBNET_PREFIX -o none 

az aks create -n $CLUSTER_NAME -g $RG -l$LOCATION \
  --max-pods 250 \
  --node-count 2 \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16 \
  --vnet-subnet-id /subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/nodesubnet \
  --enable-cilium-dataplane
```

## Demo

https://github.com/cilium/star-wars-demo

```bash
git clone git@github.com:cilium/star-wars-demo.git ; cd star-wars-demo
kubectl create -f 01-deathstar.yaml -f 02-xwing.yaml
```

