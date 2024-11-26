---
page_type: sample
languages:
- bash
- terraform
- yaml
- json
products:
- azure
- azure-resource-manager
- azure-kubernetes-service
- azure-container-registry
- azure-storage
- azure-blob-storage
- azure-storage-accounts
- azure-monitor
- azure-log-analytics
- azure-virtual-machines
name:  Building a Platform Engineering Environment on Azure Kubernetes Service (AKS)
description: This project demonstrates the process of implementing a consistent and solid platform engineering strategy on the Azure platform using Azure Kubernetes Service (AKS), ArgoCD, and Crossplane or Cluster API (CAPZ).
urlFragment: aks-platform-engineering
---

# Building a Platform Engineering Environment on Azure Kubernetes Service (AKS)

At its core, platform engineering is about constructing a solid and adaptable groundwork that simplifies and accelerates the development, deployment, and operation of software applications.  The goal is to abstract the complexity inherent in managing infrastructure and operational concerns, enabling dev teams to focus on crafting code that adds direct value. This environment is based on GitOps principles and includes a set of best practices and tools to manage the lifecycle of the applications and the underlying infrastructure. Many platform teams use multiple clusters to separate concerns and provide isolation between different environments, such as development, staging, and production. This guide provides a reference architecture and sample to build a platform engineering environment on Azure Kubernetes Service (AKS).

This sample will illustrate an end-to-end workflow that Platform Engineering and Development teams need to deploy multi-cluster environments on AKS:

- Platform Engineering team deploys a control plane cluster with core infrastructure services and tools to support Day 2 Operations using Terraform and ArgoCD.
- When a new development team is on boarded, the Platform Engineering team provisions new clusters dedicated to that team.  These new clusters will automatically have common required infrastructure tools installed via ArgoCD and have ArgoCD installed automatically.
- The development team optionally installs additional infrastructure tools and customizes the Kubernetes configuration as desired with potential limits enforced by policies from the Platform Engineering team.
- The development team deploys applications using GitOps principles and ArgoCD.

## Architecture

This sample leverages the [GitOps Bridge Pattern](https://github.com/gitops-bridge-dev/gitops-bridge?tab=readme-ov-file).  The following diagram shows the high-level architecture of the solution:  
![Platform Engineering on AKS Architecture Diagram](./images/AKS-platform-engineering-architecture.png)

The control plane cluster will be configured with addons via ArgoCD using Terraform and then bootstrapped with tools needed for Day Two operations.  

Choose Crossplane **or** Cluster API provider for Azure (CAPZ) to support deploying and managing clusters and Azure infrastructure for the application teams by changing the Terraform `infrastructure_provider` variable to either `crossplane` or `capz`.  [See this document](./docs/capz-or-crossplane.md) for further information on the comparison.  The default is `capz` if no value is specified.

## Prerequisites

- An active Azure subscription. If you don't have one, create a free Azure account before you begin.
- Azure CLI version 2.60.0 or later installed. To install or upgrade, see Install Azure CLI.
- Terraform v1.8.3 or later [configured for authentication](https://learn.microsoft.com/azure/developer/terraform/authenticate-to-azure?tabs=bash) where the user account has permissions to create resource groups and user managed identities on the subscription to setup workload identity for the AKS cluster (capz option).
- kubectl version 1.28.9 or later installed. To install or upgrade, see Install kubectl.

## Getting Started

### Configure the Access Node
Follow the steps 1-4 as described in https://github.com/DOME-Marketplace/access-node#how-to-deploy 

### Provisioning the Control Plane Cluster

- Fork the repo
- Only if the repo is desired to be private, ArgoCD will need a ssh deploy key to access this repo. Follow these steps to enable: 
    - Create a [read-only deploy ssh key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys) on the fork
    - Place the corresponding private key named `private_ssh_deploy_key` in the `terraform` directory
    - Change the `gitops_addons_org` variable to `git@github.com:Azure-Samples` replacing Azure-Samples with your fork org/username versus the existing `https://` format
    - Uncomment line 218 of the `main.tf` file: `# sshPrivateKey = file(pathexpand(var.git_private_ssh_key))`

Login into Azure and select a subscription:

```bash
az login
```

Run Terraform:

```bash
cd terraform
terraform init -upgrade
```

Choose the `infrastructure_provider` variable to be `capz` (default) or `crossplane`.

> [!Important] 
> Change `azure-samples` to your fork organization or GitHub user name in the commands below.

Alternatively, consider changing the example `tvars` file to match your desired configuration

```bash
terraform apply --auto-approve
```

> Note: You can ignore the warnings related to deprecated attributes and invalid kubeconfig path.

Terraform completed installing the AKS cluster, installing ArgoCD, and configuring ArgoCD to install applications under the `gitops/bootstrap/control-plane/addons` directory from the git repo.

### Configure Cert-Manager with Azure DNS Zone ()

https://cert-manager.io/docs/configuration/acme/dns01/azuredns/#service-principal

Configuring the AzureDNS DNS01 Challenge for a Kubernetes cluster requires creating a service principal in Azure.

Remember to switch to the right Azure Subscription that contains the Azure DNS Zone using ``az login`` command.

To create the service principal you can use the following script (requires azure-cli and jq):

```shell
#AZURE_CERT_MANAGER_NEW_SP_NAME=sp-MyServicePrincipalName
AZURE_CERT_MANAGER_NEW_SP_NAME=sp-in2-dome-aks-dev-04
#AZURE_DNS_ZONE_RESOURCE_GROUP=AZURE_DNS_ZONE_RESOURCE_GROUP
AZURE_DNS_ZONE_RESOURCE_GROUP=rg-in2-dns-dev
#AZURE_DNS_ZONE=mydomain.com
AZURE_DNS_ZONE=dev.in2.es

DNS_SP=$(az ad sp create-for-rbac --name $AZURE_CERT_MANAGER_NEW_SP_NAME --output json)
AZURE_CERT_MANAGER_SP_APP_ID=$(echo $DNS_SP | jq -r '.appId')
AZURE_CERT_MANAGER_SP_PASSWORD=$(echo $DNS_SP | jq -r '.password')
AZURE_TENANT_ID=$(echo $DNS_SP | jq -r '.tenant')
AZURE_SUBSCRIPTION_ID=$(az account show --output json | jq -r '.id')
```

For security purposes, it is appropriate to utilize RBAC to ensure that you properly maintain access control to your resources in Azure. The service principal that is generated by this tutorial has fine-grained access to ONLY the DNS Zone in the specific resource group specified. It requires this permission so that it can read/write the _acme_challenge TXT records to the zone.

Lower the Permissions of the service principal.

```shell
az role assignment delete --assignee $AZURE_CERT_MANAGER_SP_APP_ID --role Contributor
```

Give Access to DNS Zone.

```shell
DNS_ID=$(az network dns zone show --name $AZURE_DNS_ZONE --resource-group $AZURE_DNS_ZONE_RESOURCE_GROUP --query "id" --output tsv)
az role assignment create --assignee $AZURE_CERT_MANAGER_SP_APP_ID --role "DNS Zone Contributor" --scope $DNS_ID
```

Check Permissions. As the result of the following command, we would like to see just one object in the permissions array with "DNS Zone Contributor" role.

```shell
az role assignment list --all --assignee $AZURE_CERT_MANAGER_SP_APP_ID
```

A secret containing service principal password should be created on Kubernetes to facilitate presenting the challenge to Azure DNS. 

Be sure you get the current AKS credentials
```shell
az aks get-credentials --name aks-in2-dome-dev-04 --resource-group rg-in2-dome-dev-04
```

You can create the secret with the following command:

```shell
kubectl create secret generic azuredns-config --from-literal=client-secret=$AZURE_CERT_MANAGER_SP_PASSWORD --namespace=cert-manager
```

Get the variables for configuring the issuer.

```shell
echo "AZURE_CERT_MANAGER_SP_APP_ID: $AZURE_CERT_MANAGER_SP_APP_ID"
echo "AZURE_CERT_MANAGER_SP_PASSWORD: $AZURE_CERT_MANAGER_SP_PASSWORD"
echo "AZURE_SUBSCRIPTION_ID: $AZURE_SUBSCRIPTION_ID"
echo "AZURE_TENANT_ID: $AZURE_TENANT_ID"
echo "AZURE_DNS_ZONE: $AZURE_DNS_ZONE"
echo "AZURE_DNS_ZONE_RESOURCE_GROUP: $AZURE_DNS_ZONE_RESOURCE_GROUP"
```

To configure the issuer, substitute the capital cased variables with the values from the previous script. You can get the subscription id from the Azure portal.

```shell
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    ...
    solvers:
    - dns01:
        azureDNS:
          clientID: AZURE_CERT_MANAGER_SP_APP_ID
          clientSecretSecretRef:
          # The following is the secret we created in Kubernetes. Issuer will use this to present challenge to Azure DNS.
            name: azuredns-config
            key: client-secret
          subscriptionID: AZURE_SUBSCRIPTION_ID
          tenantID: AZURE_TENANT_ID
          resourceGroupName: AZURE_DNS_ZONE_RESOURCE_GROUP
          hostedZoneName: AZURE_DNS_ZONE
          # Azure Cloud Environment, default to AzurePublicCloud
          environment: AzurePublicCloud
```

### Accessing the Control Plane Cluster and ArgoCD UI

Getting the credentials for the Control Plane Cluster

```shell
export KUBECONFIG=/workspaces/aks-platform-engineering/terraform/kubeconfig
echo $KUBECONFIG
```

```shell
# Get the initial admin password and the IP address of the ArgoCD web interface.
kubectl get secrets argocd-initial-admin-secret -n argocd --template="{{index .data.password | base64decode}}"
kubectl get svc -n argocd argo-cd-argocd-server
```

It may take a few minutes for the LoadBalancer to create a public IP for the ArgoCD UI after the Terraform apply. In case something goes wrong and you don't find a public IP, connect to the ArgoCD server doing a port forward with kubectl and access the UI on https://localhost:8080.

```shell
kubectl port-forward svc/argo-cd-argocd-server -n argocd 8080:443
```

The username for the ArgoCD UI login is `admin`.

### Deploy DOME Apps using ArgoCD

```shell
kubectl apply -f gitops/apps/namespaces.yaml -n argocd
kubectl apply -f gitops/apps/sealed-secrets.yaml -n argocd
kubectl apply -f gitops/apps/cert-manager.yaml -n argocd
kubectl apply -f gitops/apps/ingress.yaml -n argocd
kubectl apply -f gitops/apps/marketplace/mysql.yaml -n argocd
kubectl apply -f gitops/apps/marketplace/mongodb.yaml -n argocd
kubectl apply -f gitops/apps/marketplace/credentials-config-service.yaml -n argocd
kubectl apply -f gitops/apps/marketplace/trusted-issuers-list.yaml -n argocd

```

### Enable Application Gateway Ingress Controller add-on for the AKS

```shell
AGW_RG_NAME=rg-in2-dome-dev-04
AGW_LOCATION=westeurope
az group create --name $AGW_RG_NAME --location $AGW_LOCATION

az network public-ip create --name pip-in2-dome-dev-04 --resource-group $AGW_RG_NAME --allocation-method Static --sku Standard --location $AGW_LOCATION

az network vnet create --name vnet-in2-dome-dev-04 --resource-group $AGW_RG_NAME --address-prefix 10.2.0.0/16 --subnet-name ApplicationGatewaySubnet --subnet-prefix 10.2.0.0/24 --location $AGW_LOCATION

az network application-gateway create --name agw-in2-dome-agw-dev-04 --resource-group $AGW_RG_NAME --sku Standard_v2 --public-ip-address pip-in2-dome-agw-dev-04 --vnet-name vnet-in2-dome-agw-dev-04 --subnet ApplicationGatewaySubnet --priority 100 --location $AGW_LOCATION
```

Then enable
https://learn.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-existing#enable-the-agic-add-on-in-existing-aks-cluster-through-azure-cli

If you do it using the Az Portal you will get de peering included.
I fnot, create a vnet peering between the two networks

```shell
AKS_NODES_RG=$(az aks show --name aks-in2-dome-dev-04 --resource-group rg-in2-dome-dev-04 -o tsv --query "nodeResourceGroup")
AKS_VNET=vnet1

aksVnetId=$(az network vnet show --name $aksVnetName --resource-group $nodeResourceGroup -o tsv --query "id")

az network vnet peering create --name AppGWtoAKSVnetPeering --resource-group myResourceGroup --vnet-name myVnet --remote-vnet $aksVnetId --allow-vnet-access

appGWVnetId=$(az network vnet show --name myVnet --resource-group myResourceGroup -o tsv --query "id")

az network vnet peering create --name AKStoAppGWVnetPeering --resource-group $nodeResourceGroup --vnet-name $aksVnetName --remote-vnet $appGWVnetId --allow-vnet-access
```

FIXME Move to terraform an denable AGIC by default in the same RG

### Summary

1. Terraform created an AKS control plane / management cluster and downloaded the kubeconfig file in the `terraform` directory.
1. Terraform installed ArgoCD via the Terraform Kubernetes provider to that cluster
1. Terraform did a `kubectl apply` an ArgoCD ApplicationSet to the cluster which syncs the bootstrap folder [gitops/bootstrap/control-plane/addons](https://github.com/Azure-Samples/aks-platform-engineering/tree/main/gitops/bootstrap/control-plane/addons). That ApplicationSet utilizes the [ArgoCD App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern) and ArgoCD applies all of the applications under that folder in git which match the [labels specified in Terraform](https://github.com/Azure-Samples/aks-platform-engineering/blob/main/terraform/main.tf#L20-L38).

### Delete the Control Plane Cluster

Run Terraform:

```bash
terraform destroy --auto-approve
```

## Next Steps

Learn how to define your own cluster, infrastructure, and hand off to the development team the access to the AKS cluster and ArgoCD deployment UI in [this article](./docs/Onboard-New-Dev-Team.md).

## Trademarks

Trademarks This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft trademarks or logos is subject to and must follow Microsoft’s Trademark & Brand Guidelines. Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship. Any use of third-party trademarks or logos are subject to those third-party’s policies.