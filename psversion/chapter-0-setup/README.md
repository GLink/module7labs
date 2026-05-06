# Chapter 0: Setup and Prerequisites

In this chapter, you'll set up your environment and create the AKS cluster that will be used throughout all subsequent chapters.

## 📋 Prerequisites

Before starting this lab, ensure you have:

### Required Tools

1. **Azure CLI** (version 2.50.0 or later)
   - [Installation guide](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
   - Verify: `az --version`

2. **kubectl** (Kubernetes command-line tool)
   - [Installation guide](https://kubernetes.io/docs/tasks/tools/)
   - Verify: `kubectl version --client`

3. **Helm** (version 3.x)
   - [Installation guide](https://helm.sh/docs/intro/install/)
   - Verify: `helm version`

4. **jq** (JSON processor - optional but helpful)
   - [Installation guide](https://stedolan.github.io/jq/download/)
   - Verify: `jq --version`

### Azure Requirements

- Active Azure subscription
- Permissions to create:
  - Resource Groups
  - AKS Clusters
  - Azure Service Bus Namespaces (for KEDA chapter)
  - Role Assignments

## 🎯 Learning Objectives

- Set up student-specific naming conventions
- Create a resource group for lab resources
- Deploy an AKS cluster with required features
- Configure kubectl access to the cluster
- Verify cluster functionality
- Create Azure Service Bus for KEDA exercises

## 📝 Step 1: Set Your Student Initials

To avoid naming conflicts in a shared subscription, you'll use your initials for resource naming.

```powershell
# Set your initials (use 2-4 lowercase letters)
$STUDENT_INITIALS = "abc"  # CHANGE THIS to your initials

# Verify it's set
Write-Host "Your initials: $STUDENT_INITIALS"
```

## 📦 Step 2: Define Resource Names

```powershell
# Set variables for resource names
$LOCATION = "swedencentral"
$RESOURCE_GROUP = "rg-aks-lab-$STUDENT_INITIALS"
$CLUSTER_NAME = "aks-lab-$STUDENT_INITIALS"
$SERVICE_BUS_NAMESPACE = "sb-aks-lab-$STUDENT_INITIALS"

# Display your configuration
Write-Host "Resource Configuration:"
Write-Host "  Location: $LOCATION"
Write-Host "  Resource Group: $RESOURCE_GROUP"
Write-Host "  Cluster Name: $CLUSTER_NAME"
Write-Host "  Service Bus Namespace: $SERVICE_BUS_NAMESPACE"
```

## 🔐 Step 3: Login to Azure

```powershell
# Login to Azure
az login

# Set the subscription (if you have multiple subscriptions)
# az account set --subscription "YOUR-SUBSCRIPTION-ID"

# Verify your subscription
az account show --query "{Name:name, SubscriptionId:id, TenantId:tenantId}" --output table
```

## 🏗️ Step 4: Create Resource Group

```powershell
# Create the resource group
az group create `
  --name $RESOURCE_GROUP `
  --location $LOCATION

Write-Host "✅ Resource group created: $RESOURCE_GROUP"
```

## ☸️ Step 5: Create AKS Cluster

We'll create an AKS cluster with features needed for all lab chapters:

```powershell
# Create AKS cluster with required features
az aks create `
  --resource-group $RESOURCE_GROUP `
  --name $CLUSTER_NAME `
  --location $LOCATION `
  --node-count 3 `
  --node-vm-size Standard_D4s_v5 `
  --zones 1 2 3 `
  --enable-managed-identity `
  --enable-oidc-issuer `
  --enable-workload-identity `
  --enable-aad `
  --enable-azure-rbac `
  --network-plugin azure `
  --max-pods 110 `
  --generate-ssh-keys

Write-Host "⏳ AKS cluster creation in progress (this takes 5-10 minutes)..."
```

### Cluster Configuration Explained

- **--node-count 3**: Three nodes for testing autoscaling and disruptions
- **--zones 1 2 3**: Spread nodes across availability zones for resilience
- **--enable-managed-identity**: Use managed identity for Azure integrations
- **--enable-oidc-issuer**: Required for Workload Identity (needed for KEDA with Azure Service Bus)
- **--enable-workload-identity**: Enable workload identity federation
- **--network-plugin azure**: Use Azure CNI for advanced networking

## 🔌 Step 6: Configure kubectl Access

```powershell
# Get AKS credentials
az aks get-credentials `
  --resource-group $RESOURCE_GROUP `
  --name $CLUSTER_NAME `
  --overwrite-existing

Write-Host "✅ kubectl configured for cluster: $CLUSTER_NAME"
```

## ✅ Step 7: Verify Cluster

```powershell
# Check cluster connection
kubectl cluster-info

# List nodes
kubectl get nodes -o wide

# Verify nodes are in different zones
kubectl get nodes -o custom-columns=NAME:.metadata.name,ZONE:.metadata.labels."topology\.kubernetes\.io/zone"

# Check system pods
kubectl get pods -n kube-system

# Check node resources
kubectl top nodes
```

Expected output:
- 3 nodes in "Ready" state
- Nodes distributed across zones 1, 2, and 3
- System pods running in kube-system namespace

**Note**: Metrics server is installed by default in AKS, so metrics should be available after the cluster is ready.

## 📊 Step 8: Verify Metrics Server

AKS clusters come with metrics-server pre-installed. Let's verify it's working:

```powershell
# Check metrics-server pods
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Verify metrics are available (may take 1-2 minutes after cluster creation)
kubectl top nodes
```

**Note**: If metrics aren't immediately available, wait a minute and try again. The metrics-server needs time to collect data after cluster creation.

## 🚌 Step 9: Create Azure Service Bus (for KEDA Chapter)

We'll create an Azure Service Bus namespace for the KEDA exercises:

```powershell
# Create Service Bus namespace
az servicebus namespace create `
  --name $SERVICE_BUS_NAMESPACE `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --sku Standard

Write-Host "✅ Service Bus namespace created: $SERVICE_BUS_NAMESPACE"

# Create a queue
az servicebus queue create `
  --resource-group $RESOURCE_GROUP `
  --namespace-name $SERVICE_BUS_NAMESPACE `
  --name orders

Write-Host "✅ Service Bus queue 'orders' created"

# Get connection string for later use
$SERVICE_BUS_CONNECTION_STRING = $(az servicebus namespace authorization-rule keys list `
  --resource-group $RESOURCE_GROUP `
  --namespace-name $SERVICE_BUS_NAMESPACE `
  --name RootManageSharedAccessKey `
  --query primaryConnectionString `
  --output tsv)

Write-Host "✅ Service Bus connection string retrieved"
```

## 📊 Step 10: Save Your Configuration

Create a file to save your environment variables for future sessions:

```powershell
# Create a configuration file
@"
# AKS Lab Configuration for student: $STUDENT_INITIALS
`$STUDENT_INITIALS = "$STUDENT_INITIALS"
`$LOCATION = "$LOCATION"
`$RESOURCE_GROUP = "$RESOURCE_GROUP"
`$CLUSTER_NAME = "$CLUSTER_NAME"
`$SERVICE_BUS_NAMESPACE = "$SERVICE_BUS_NAMESPACE"

# Derived values
`$SUBSCRIPTION_ID = `$(az account show --query id -o tsv)
`$TENANT_ID = `$(az account show --query tenantId -o tsv)
`$AKS_OIDC_ISSUER = `$(az aks show --resource-group `$RESOURCE_GROUP --name `$CLUSTER_NAME --query 'oidcIssuerProfile.issuerUrl' -o tsv)
`$SERVICE_BUS_CONNECTION_STRING = `$(az servicebus namespace authorization-rule keys list --resource-group `$RESOURCE_GROUP --namespace-name `$SERVICE_BUS_NAMESPACE --name RootManageSharedAccessKey --query primaryConnectionString --output tsv)

Write-Host "Configuration loaded for student: `$STUDENT_INITIALS"
"@ | Set-Content -Path .\lab-config.ps1

Write-Host "✅ Configuration saved to lab-config.ps1"
Write-Host "   Load it in future sessions with: . .\lab-config.ps1"
```

## 🎓 Summary

You have successfully:
- ✅ Installed required tools
- ✅ Set up student-specific naming
- ✅ Created a resource group
- ✅ Deployed an AKS cluster with advanced features
- ✅ Configured kubectl access
- ✅ Installed metrics-server for monitoring
- ✅ Created Azure Service Bus for KEDA exercises
- ✅ Verified cluster functionality
- ✅ Saved your configuration

## 📝 Important Notes

- **Keep your cluster running** - You'll use it for all subsequent chapters
- **Save your lab-config.ps1 file** - You'll need it to restore variables
- **Note your resource names** - You'll reference them throughout the lab
- **Metrics collection** - It may take 1-2 minutes for metrics to appear after pod deployment

## 🔄 Loading Configuration in Future Sessions

When you start a new terminal session:

```powershell
# Navigate to lab directory
Set-Location C:\path\to\module7labs

# Load your configuration
. .\lab-config.ps1

# Re-authenticate if needed
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME
```

## 🧹 Cleanup (Only at End of ALL Chapters)

**⚠️ WARNING: Only run this after completing ALL chapters!**

```powershell
# Delete the entire resource group (removes all resources)
az group delete `
  --name $RESOURCE_GROUP `
  --yes `
  --no-wait

Write-Host "🗑️ Resource group deletion initiated"
```

## 🚀 Next Steps

Now that your environment is set up, proceed to:
- **[Chapter 1: Resource Requests and Limits](../chapter-1-requests-limits/README.md)**

---

**Questions or Issues?**
- Verify all commands completed successfully
- Check that all pods in kube-system are running
- Ensure metrics-server is collecting data: `kubectl top nodes`
- Ensure you have proper Azure permissions
- Ask your instructor for assistance
