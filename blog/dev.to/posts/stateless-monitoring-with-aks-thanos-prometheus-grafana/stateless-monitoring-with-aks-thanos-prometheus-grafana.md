---
title: Stateless, Secretless OSS Monitoring in Azure Kubernetes Service with Thanos, Prometheus and Azure Managed Grafana 
published: false
description: A simple test article
tags: 'thanos, azure, prometheus, grafana, aks'
cover_image: ./assets/cat.jpg
canonical_url: null
---

Some random text with a [link](https://code.visualstudio.com).

## Stateless, Secretless OSS Monitoring in Azure Kubernetes Service with Thanos, Prometheus and Azure Managed Grafana 

### Introduction 

Observability is 


### Prerequisites

- An 1.23 or 1.24 AKS cluster with either a user-managed identity assigned to the kubelet identity or system-assigned identity
- Ability to assign roles on Azure resources (User Access Administrator role)
- A storage account
- (Recommended) A public DNS zone in Azure
- Azure CLI
- Helm CLI

### Cluster-wide services

For Thanos receive and query components to be available outside the cluster and secured with TLS, we will need [ingress-nginx](https://github.com/kubernetes/ingress-nginx) and [cert-manager](https://cert-manager.io/). For ingress, deploy the Helm chart using the following command, to account for this [issue](link to issue):

```
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
  --set controller.service.externalTrafficPolicy=Local \
  --namespace ingress-nginx --create-namespace
```

Notice the extra annotations and the `externalTrafficPolicy` set to `Local`.

Next, we need `cert-manager` to automatically provision SSL certificates from Let's Encrypt; we will just need a valid email address for the ClusterIssuer:

```
helm upgrade -i cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true \
  --set ingressShim.defaultIssuerName=letsencrypt-prod \
  --set ingressShim.defaultIssuerKind=ClusterIssuer \
  jetstack/cert-manager

kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: alessandro.vozza@microsoft.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```

Last but not least, we will add add DNS record for our ingress Loadbalancer IP, so it will be seamless to get public FQDNs for our endpoints for Thanos receive and Thanos Query.

```
az network dns record-set a add-record  -n "*.thanos" -g dns -z cookingwithazure.com --ipv4-address $(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
```

Note how we use `kubectl` with `jsonpath` type output to get the ingress public IP.

### Storage account preparation

Because we do not want to store any secret or service principal in-cluster, we will leverage the Managed Identities assigned to the cluster and assign the relevant Azure Roles to the storage account.

Once you have created or identified the storage account to use and created a container within it, to store the Thanos metrics, assign the roles using the `azure cli`; first, determine the clientID of the managed identity:

```
clientid=$(az aks show -g <rg> -n <cluster_name> -o json --query identityProfile.kubeletidentity.clientId)
```

Now, assign the role of `Reader and Data Access` to the **Storage account** (you need this so the cloud controller can generate access keys for the containers) and the `Storage Blob Data Contributor` role **to the container only** (there's no need to give this permission at the storage account level, because it will enable writing to *every* container, which we don't need. Always remember to apply the [principles of least priviliges](https://www.cisa.gov/uscert/bsi/articles/knowledge/principles/least-privilege)!)


```
az role assignment create --role "Reader and data access" --assignee $clientid --scope /subscriptions/<subID>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account_name>

az role assignment create --role "Storage Blob Data Contributor" --assignee $clientid --scope /subscriptions/<subID>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/<account_name>/containers/<container_name>
```

### Create basic auth credentials

Ok, we kinda cheated in the title: you **do** need one credential at least for this setup, and it's the one to access the Prometheus API exposed by Thanos from Azure Managed Grafana. We will use the same credentials (but feel free to generate a different one) to push metrics from Prometheus to Thanos using `remote-write` via the ingress controller. You'll need a strong password stored into a file called `pass` locally:

```
htpasswd -c  -i auth thanos < pass

#Create the namespaces
kubectl create ns thanos
kubectl create ns prometheus

#for Thanos Query and Receive
kubectl create secret generic -n thanos basic-auth --from-file=auth

#for Prometheus remote write
kubectl create secret generic -n prometheus remotewrite-secret --from-literal=user=thanos --from-literal=password=$(cat pass)
```

### Deploying Thanos

We will use the [Bitnami chart](https://github.com/bitnami/charts/tree/master/bitnami/thanos/) to deploy the Thanos components we need.

```
helm upgrade -i thanos -n monitoring --create-namespace --values thanos-values.yaml bitnami/thanos
```

Let's go thru the relevant sections of the [values file](blog/dev.to/posts/assets/stateless-monitoring-with-aks-thanos-prometheus-grafana/files/thanos-values.yaml):

```
objstoreConfig: |-
  type: AZURE
  config:
    storage_account: "thanostore"
    container: "thanostore"
    endpoint: "blob.core.windows.net"
    max_retries: 0
    user_assigned_id: "5c424851-e907-4cb0-acb5-3ea42fc56082"
```

(replace the `user_assigned_id` with the object id of your kubeletIdentity, for more information about AKS identities, check out [this article](https://docs.microsoft.com/en-us/azure/aks/use-managed-identity#use-a-pre-created-kubelet-managed-identity)) This section instructs the Thanos Store Gateway and Compactor to use an Azure Blob store, and to use the kubelet identity to access it. Next, we enable the ruler and the query components:

```
ruler:
  enabled: true
  
query:
  enabled: true
...
queryFrontend:
  enabled: true
```

We also enable autoscaling for the stateless query components (the `query` and the `query-frontend`; the latter helps aggregating read queries), and we enable simple authentication for the Query frontend service using `ingress-nginx` annotations:

```
queryFrontend:
...
  ingress:
    enabled: true
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/auth-type: basic
      nginx.ingress.kubernetes.io/auth-secret: basic-auth
      nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required - thanos'
    hostname: query.thanos.cookingwithazure.com
    ingressClassName: nginx
    tls: true
```

The annotation references the `basic-auth` secret we created before from the `htpasswd` credentials. Note that the same annotations are also under the `receive` section, as we're using the exact same secret for pushing metrics *into* Thanos (although with a different `hostname`).

# Prometheus remote-write 
