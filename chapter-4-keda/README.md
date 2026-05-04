# Chapter 4: KEDA - Kubernetes Event-Driven Autoscaling

In this chapter, you'll learn how to use KEDA (Kubernetes Event Driven Autoscaling) to scale applications based on event sources and custom metrics beyond CPU and memory.

## 🎯 Learning Objectives

- Understand KEDA architecture and how it extends HPA
- Install KEDA in your AKS cluster
- Scale applications based on Azure Service Bus queue length
- Scale applications based on HTTP requests using KEDA HTTP Add-on
- Configure scale-to-zero scenarios
- Compare KEDA with standard HPA
- Understand KEDA best practices

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Completed [Chapter 3: Horizontal Pod Autoscaler (HPA)](../chapter-3-hpa/README.md)
- AKS cluster with workload identity enabled
- Azure Service Bus namespace created (from Chapter 0)

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./lab-config.sh

# Verify cluster access
kubectl get nodes

# Verify Service Bus namespace exists
az servicebus namespace show --name $SERVICE_BUS_NAMESPACE --resource-group $RESOURCE_GROUP --query name -o tsv
```

## 📖 Understanding KEDA

### What is KEDA?

KEDA (Kubernetes Event Driven Autoscaling) is a Kubernetes-based event-driven autoscaler that extends HPA functionality.

**Key Features**:
- Scales based on external event sources (queues, databases, HTTP, etc.)
- Supports scale-to-zero for cost optimization
- Works alongside standard HPA
- 50+ built-in scalers

### KEDA Architecture

1. **KEDA Operator**: Manages ScaledObjects and ScaledJobs
2. **Metrics Server**: Exposes external metrics to HPA
3. **Admission Webhooks**: Validates KEDA resources
4. **Scalers**: Connectors to external event sources

### KEDA vs Standard HPA

| Feature            | Standard HPA   | KEDA                 |
| ------------------ | -------------- | -------------------- |
| **Metric Sources** | CPU, Memory    | 50+ external sources |
| **Scale to Zero**  | No (min 1 pod) | Yes                  |
| **Custom Metrics** | Complex setup  | Built-in scalers     |
| **Event-Driven**   | No             | Yes                  |

## 🏗️ Step 1: Install KEDA

```bash
# Add KEDA Helm repository
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# Install KEDA
helm install keda kedacore/keda --namespace keda --create-namespace

echo "⏳ Waiting for KEDA to be ready..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=keda-operator -n keda --timeout=120s

echo "✅ KEDA installed"
```

## ✅ Step 2: Verify KEDA Installation

```bash
# Check KEDA components
kubectl get pods -n keda

# Check KEDA version
helm list -n keda

# Verify KEDA CRDs are installed
kubectl get crd | grep keda

# Check KEDA operator logs
kubectl logs -n keda -l app.kubernetes.io/name=keda-operator --tail=20
```

Expected: You should see keda-operator and keda-metrics-apiserver pods running.

## 🏗️ Step 3: Create Namespace

```bash
# Create namespace for KEDA demos
kubectl create namespace keda-demo

echo "✅ Namespace created"
```

## 🚌 Step 4: Configure Azure Service Bus Scaler

Let's create an application that scales based on Azure Service Bus queue length.

First, create a workload identity for Service Bus access:

```bash
# Create managed identity
IDENTITY_NAME="id-keda-servicebus-$STUDENT_INITIALS"

az identity create \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --location $LOCATION

echo "✅ Managed identity created"

# Get identity details
IDENTITY_CLIENT_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query clientId -o tsv)

IDENTITY_PRINCIPAL_ID=$(az identity show \
  --resource-group $RESOURCE_GROUP \
  --name $IDENTITY_NAME \
  --query principalId -o tsv)

echo "Identity Client ID: $IDENTITY_CLIENT_ID"

# Get Service Bus resource ID
SERVICE_BUS_RESOURCE_ID=$(az servicebus namespace show \
  --name $SERVICE_BUS_NAMESPACE \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

# Grant Service Bus Data Receiver role
az role assignment create \
  --role "Azure Service Bus Data Receiver" \
  --assignee-object-id $IDENTITY_PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --scope $SERVICE_BUS_RESOURCE_ID

echo "✅ Service Bus access granted"

# Wait for role assignment propagation
echo "⏳ Waiting 30 seconds for role assignment to propagate..."
sleep 30
```

## 🔗 Step 5: Create Federated Identity Credential

```bash
# Verify all identity variables are set
echo "🔍 Verifying identity variables..."
echo "Identity Name: $IDENTITY_NAME"
echo "Client ID: $IDENTITY_CLIENT_ID"
echo "Principal ID: $IDENTITY_PRINCIPAL_ID"

if [ -z "$IDENTITY_NAME" ] || [ -z "$IDENTITY_CLIENT_ID" ] || [ -z "$IDENTITY_PRINCIPAL_ID" ]; then
  echo "❌ Identity variables not set. Re-fetching from Azure..."
  
  export IDENTITY_NAME="keda-servicebus-identity-$STUDENT_INITIALS"
  
  # Get identity details
  IDENTITY_INFO=$(az identity show \
    --name $IDENTITY_NAME \
    --resource-group $RESOURCE_GROUP \
    --query '{clientId:clientId, principalId:principalId}' -o json)
  
  export IDENTITY_CLIENT_ID=$(echo $IDENTITY_INFO | jq -r '.clientId')
  export IDENTITY_PRINCIPAL_ID=$(echo $IDENTITY_INFO | jq -r '.principalId')
  
  echo "✅ Identity variables loaded:"
  echo "  Client ID: $IDENTITY_CLIENT_ID"
  echo "  Principal ID: $IDENTITY_PRINCIPAL_ID"
fi

# Get AKS OIDC Issuer
if [ -z "$AKS_OIDC_ISSUER" ]; then
  export AKS_OIDC_ISSUER=$(az aks show \
    --resource-group $RESOURCE_GROUP \
    --name $CLUSTER_NAME \
    --query 'oidcIssuerProfile.issuerUrl' -o tsv)
fi

echo "AKS OIDC Issuer: $AKS_OIDC_ISSUER"

# Delete existing federated credential if it exists
az identity federated-credential delete \
  --name "fed-keda-servicebus" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --yes 2>/dev/null || true

# Create federated credential
az identity federated-credential create \
  --name "fed-keda-servicebus" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --issuer $AKS_OIDC_ISSUER \
  --subject "system:serviceaccount:keda-demo:servicebus-processor-sa" \
  --audience api://AzureADTokenExchange

echo "✅ Federated credential created"
```

## 🔧 Step 5.5: Grant KEDA Access to Service Bus

KEDA's operator needs to query Service Bus to get queue metrics. We'll grant the managed identity to KEDA:

```bash
# Get KEDA operator service account
KEDA_SA_NAME="keda-operator"
KEDA_NAMESPACE="keda"

# Create federated credential for KEDA operator
az identity federated-credential delete \
  --name "fed-keda-operator" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --yes 2>/dev/null || true

az identity federated-credential create \
  --name "fed-keda-operator" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --issuer $AKS_OIDC_ISSUER \
  --subject "system:serviceaccount:${KEDA_NAMESPACE}:${KEDA_SA_NAME}" \
  --audience api://AzureADTokenExchange

echo "✅ Federated credential created for KEDA operator"

# Annotate KEDA operator service account with the managed identity
kubectl annotate serviceaccount $KEDA_SA_NAME -n $KEDA_NAMESPACE \
  azure.workload.identity/client-id=$IDENTITY_CLIENT_ID \
  --overwrite

echo "✅ KEDA operator service account annotated"

# Patch KEDA operator deployment to add workload identity label to pods
kubectl patch deployment keda-operator -n $KEDA_NAMESPACE --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/metadata/labels/azure.workload.identity~1use",
    "value": "true"
  }
]'

echo "✅ KEDA operator deployment patched with workload identity label"

# Wait for rollout to complete
echo "⏳ Waiting for KEDA operator to restart with new configuration..."
kubectl rollout status deployment keda-operator -n keda --timeout=120s

# Verify workload identity injection
echo -e "\n🔍 Verifying workload identity injection:"
kubectl get pods -n keda -l app=keda-operator -o jsonpath='{.items[0].metadata.labels.azure\.workload\.identity/use}' && echo ""
kubectl get pods -n keda -l app=keda-operator -o yaml | grep -A 3 "AZURE_" | head -10

echo "✅ KEDA operator configured with workload identity"
```

## 📝 Step 6: Create Service Account

```bash
# Create service account with workload identity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: servicebus-processor-sa
  namespace: keda-demo
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID
EOF

echo "✅ Service account created"
```

## 📦 Step 7: Deploy Service Bus Message Processor

```bash
# Deploy application that actually processes Service Bus messages
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: servicebus-processor
  namespace: keda-demo
spec:
  replicas: 0  # Start with 0 replicas (KEDA will scale up)
  selector:
    matchLabels:
      app: servicebus-processor
  template:
    metadata:
      labels:
        app: servicebus-processor
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: servicebus-processor-sa
      containers:
      - name: processor
        image: python:3.11-slim
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
        env:
        - name: SERVICE_BUS_NAMESPACE
          value: "${SERVICE_BUS_NAMESPACE}.servicebus.windows.net"
        - name: QUEUE_NAME
          value: "orders"
        command:
        - "/bin/sh"
        - "-c"
        - |
          echo "Installing Azure Service Bus SDK..."
          pip install azure-servicebus azure-identity --quiet
          
          echo "Starting Service Bus message processor..."
          python3 << 'PYTHON_CODE'
          import os
          import sys
          import time
          from azure.servicebus import ServiceBusClient
          from azure.identity import DefaultAzureCredential

          namespace = os.environ.get('SERVICE_BUS_NAMESPACE')
          queue_name = os.environ.get('QUEUE_NAME')

          print(f"Connecting to Service Bus: {namespace}")
          print(f"Queue: {queue_name}")
          print(f"Using Workload Identity authentication")

          # Use DefaultAzureCredential which will use workload identity
          credential = DefaultAzureCredential()
          
          try:
              client = ServiceBusClient(
                  fully_qualified_namespace=namespace,
                  credential=credential
              )
              
              with client:
                  receiver = client.get_queue_receiver(queue_name=queue_name)
                  
                  print("✅ Connected to Service Bus queue")
                  print("⏳ Waiting for messages...")
                  
                  with receiver:
                      while True:
                          messages = receiver.receive_messages(max_message_count=10, max_wait_time=5)
                          
                          if messages:
                              for message in messages:
                                  print(f"📨 Received message: {str(message)[:100]}")
                                  # Complete the message (remove from queue)
                                  receiver.complete_message(message)
                                  print(f"✅ Message completed and removed from queue")
                                  # Simulate processing time
                                  time.sleep(2)
                          else:
                              print(f"[{time.strftime('%H:%M:%S')}] No messages, waiting...")
                              time.sleep(5)
                              
          except Exception as e:
              print(f"❌ Error: {str(e)}")
              sys.exit(1)
          PYTHON_CODE
EOF

echo "✅ Service Bus processor deployed with 0 replicas"
echo "📝 This container will actually consume and complete messages from the Service Bus queue"
```

## 📊 Step 8: Create KEDA ScaledObject for Service Bus

```bash
# Verify identity variables are set
if [ -z "$IDENTITY_CLIENT_ID" ]; then
  echo "❌ IDENTITY_CLIENT_ID not set. Please run Step 5 first."
  exit 1
fi

# Get Service Bus namespace fully qualified name
SERVICE_BUS_FQDN="${SERVICE_BUS_NAMESPACE}.servicebus.windows.net"

echo "Creating KEDA ScaledObject with:"
echo "  Service Bus Namespace: $SERVICE_BUS_NAMESPACE"
echo "  Identity Client ID: $IDENTITY_CLIENT_ID"

# Create ScaledObject
cat <<EOF | kubectl apply -f -
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: servicebus-scaler
  namespace: keda-demo
spec:
  scaleTargetRef:
    name: servicebus-processor
  minReplicaCount: 0   # Scale to zero when no messages
  maxReplicaCount: 10  # Maximum 10 replicas
  cooldownPeriod: 60   # Wait 60 seconds before scaling to zero
  pollingInterval: 10  # Check queue every 10 seconds
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      namespace: ${SERVICE_BUS_NAMESPACE}
      messageCount: "5"  # Scale up when more than 5 messages per pod
    authenticationRef:
      name: azure-servicebus-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-servicebus-auth
  namespace: keda-demo
spec:
  podIdentity:
    provider: azure-workload
EOF

echo "✅ KEDA ScaledObject created"

# Wait a moment for KEDA to process
echo "⏳ Waiting 10 seconds for KEDA to process the ScaledObject..."
sleep 10

# Check status immediately
echo -e "\n📊 Initial ScaledObject Status:"
kubectl get scaledobject servicebus-scaler -n keda-demo
```

**Important**: The configuration uses:
- `namespace`: The short Service Bus namespace name (e.g., "sb-aks-lab-xyz") - **required when using pod identity**
- `azure-workload` provider: KEDA operator uses workload identity for authentication (configured in Step 5.5)
- No `identityId` specified: KEDA will use its own service account's identity

### ScaledObject Configuration Explained

- **minReplicaCount: 0**: Enable scale-to-zero
- **maxReplicaCount: 10**: Maximum pods
- **cooldownPeriod: 60**: Wait 60s after last trigger before scaling to zero
- **pollingInterval: 10**: Check queue every 10 seconds
- **messageCount: "5"**: Target 5 messages per pod

## 🔍 Step 9: Verify KEDA ScaledObject

```bash
# Check ScaledObject status
kubectl get scaledobject -n keda-demo

# Detailed view
kubectl describe scaledobject servicebus-scaler -n keda-demo

# Check current pod count (should be 0)
kubectl get pods -n keda-demo -l app=servicebus-processor

# Check the HPA created by KEDA
kubectl get hpa -n keda-demo

# Check KEDA operator logs for any errors
echo -e "\n📋 KEDA Operator Logs (last 20 lines):"
kubectl logs -n keda -l app=keda-operator --tail=20

# Check if workload identity is properly configured
echo -e "\n🔍 Verifying workload identity webhook:"
kubectl get pods -n keda -l app=keda-operator -o yaml | grep -A 5 "azure.workload.identity"
```

**Troubleshooting**: If ScaledObject shows `READY: False`, check:
1. KEDA operator logs for authentication errors
2. Workload identity webhook is installed
3. Role assignments have propagated (wait 2-3 minutes)
4. Federated credential matches the service account namespace and name

Expected: ScaledObject should be in "Ready" state with 0 replicas since the queue is empty.

## 🔧 Step 9.5: Troubleshoot KEDA Authentication (If READY: False)

If your ScaledObject shows `READY: False`, run these troubleshooting steps:

```bash
echo "🔍 Troubleshooting KEDA authentication..."

# First, verify identity variables are set
if [ -z "$IDENTITY_NAME" ]; then
  export IDENTITY_NAME="keda-servicebus-identity-$STUDENT_INITIALS"
fi

# Check KEDA operator logs for errors
echo -e "\n📋 Recent KEDA errors:"
kubectl logs -n keda -l app=keda-operator --tail=50 | grep -i "error\|failed\|denied" | tail -10

# Verify the managed identity exists and has correct client ID
echo -e "\n🆔 Managed Identity Info:"
echo "Identity Name: $IDENTITY_NAME"
if [ -z "$IDENTITY_CLIENT_ID" ]; then
  echo "⚠️  Client ID not set, fetching..."
  export IDENTITY_CLIENT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query clientId -o tsv)
  export IDENTITY_PRINCIPAL_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query principalId -o tsv)
fi
echo "Client ID: $IDENTITY_CLIENT_ID"
echo "Principal ID: $IDENTITY_PRINCIPAL_ID"

# Verify role assignment
echo -e "\n📝 Checking role assignments:"
az role assignment list \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ServiceBus/namespaces/$SERVICE_BUS_NAMESPACE" \
  --query "[?principalId=='$IDENTITY_PRINCIPAL_ID'].{Role:roleDefinitionName, Principal:principalId}" \
  -o table

# Verify federated credential
echo -e "\n🔗 Federated Credential:"
az identity federated-credential show \
  --name "fed-keda-servicebus" \
  --identity-name $IDENTITY_NAME \
  --resource-group $RESOURCE_GROUP \
  --query '{Name:name, Subject:subject, Issuer:issuer, Audiences:audiences}' \
  -o table 2>/dev/null || echo "❌ Federated credential not found or error"

# Verify service account annotation
echo -e "\n📦 Service Account Configuration:"
kubectl get sa servicebus-processor-sa -n keda-demo -o jsonpath='{.metadata.annotations.azure\.workload\.identity/client-id}' && echo ""

# Check TriggerAuthentication
echo -e "\n🔐 TriggerAuthentication Configuration:"
kubectl get triggerauthentication azure-servicebus-auth -n keda-demo -o yaml | grep -A 5 "spec:"

echo -e "\n⏰ Common fixes:"
echo "1. If identity variables are empty, re-run Step 5 to reload them"
echo "2. If role assignments are missing, re-run Step 4"
echo "3. If federated credential shows wrong subject, re-run Step 5"
echo "4. Wait 2-3 minutes for Azure RBAC propagation after role assignment"
echo ""
echo "After fixing, recreate the ScaledObject by re-running Step 8"
```

## 🔧 Step 9.6: Fix ScaledObject Configuration (If Still Not Working)

If after troubleshooting the ScaledObject still shows errors, delete and recreate it:

```bash
echo "🔧 Fixing ScaledObject configuration..."

# Verify identity variables are loaded
if [ -z "$IDENTITY_CLIENT_ID" ]; then
  export IDENTITY_NAME="keda-servicebus-identity-$STUDENT_INITIALS"
  export IDENTITY_CLIENT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RESOURCE_GROUP --query clientId -o tsv)
  echo "Loaded Client ID: $IDENTITY_CLIENT_ID"
fi

# Delete existing ScaledObject
echo "Deleting existing ScaledObject..."
kubectl delete scaledobject servicebus-scaler -n keda-demo 2>/dev/null || true
kubectl delete triggerauthentication azure-servicebus-auth -n keda-demo 2>/dev/null || true

echo "Waiting 5 seconds..."
sleep 5

# Recreate with correct configuration
SERVICE_BUS_FQDN="${SERVICE_BUS_NAMESPACE}.servicebus.windows.net"

echo "Creating ScaledObject with:"
echo "  Namespace (short): $SERVICE_BUS_NAMESPACE"
echo "  Client ID: $IDENTITY_CLIENT_ID"

cat <<EOF | kubectl apply -f -
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-servicebus-auth
  namespace: keda-demo
spec:
  podIdentity:
    provider: azure-workload
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: servicebus-scaler
  namespace: keda-demo
spec:
  scaleTargetRef:
    name: servicebus-processor
  minReplicaCount: 0
  maxReplicaCount: 10
  cooldownPeriod: 60
  pollingInterval: 10
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      namespace: ${SERVICE_BUS_NAMESPACE}
      messageCount: "5"
    authenticationRef:
      name: azure-servicebus-auth
EOF

echo "✅ ScaledObject recreated"

# Wait and check status
echo "⏳ Waiting 15 seconds for KEDA to process..."
sleep 15

echo -e "\n📊 ScaledObject Status:"
kubectl get scaledobject servicebus-scaler -n keda-demo

echo -e "\n📋 Recent KEDA logs:"
kubectl logs -n keda -l app=keda-operator --tail=10 | grep "servicebus-scaler" || echo "No recent logs"

echo -e "\n🔍 If still showing errors, check:"
echo "1. KEDA operator has workload identity enabled: kubectl get pods -n keda -o yaml | grep azure.workload.identity"
echo "2. Wait 2-3 minutes for Azure RBAC to fully propagate"
echo "3. Verify Service Bus has Entra ID authentication enabled (not disabled)"
```

## 📨 Step 10: Send Messages to Service Bus Queue

## 📨 Step 10: Send Messages to Service Bus Queue

We'll use Entra ID (Azure AD) authentication to send messages securely:

```bash
# Get current user's object ID
CURRENT_USER_ID=$(az ad signed-in-user show --query id -o tsv)

echo "📋 Granting Azure Service Bus Data Sender role to current user..."

# Assign Azure Service Bus Data Sender role to the current user
az role assignment create \
  --role "Azure Service Bus Data Sender" \
  --assignee $CURRENT_USER_ID \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ServiceBus/namespaces/$SERVICE_BUS_NAMESPACE"

echo "✅ Role assigned (may take 1-2 minutes to propagate)"
echo "⏳ Waiting 30 seconds for role assignment to propagate..."
sleep 30

# Get an access token for Service Bus data plane
echo "🔐 Getting Entra ID access token..."

# Check if running in Cloud Shell (which requires interactive login for Service Bus)
if [ ! -z "$ACC_CLOUD" ]; then
  echo "⚠️  Detected Cloud Shell environment"
  echo "Cloud Shell MSI doesn't support Service Bus tokens - interactive login required"
  echo ""
  echo "Running interactive login for Service Bus..."
  az logout
  az login --scope https://servicebus.azure.net/.default
  echo ""
fi

# Get access token
ACCESS_TOKEN=$(az account get-access-token --scope https://servicebus.azure.net/.default --query accessToken -o tsv 2>&1)

# Check if token retrieval failed
if [[ "$ACCESS_TOKEN" == *"not a supported MSI token audience"* ]] || [ -z "$ACCESS_TOKEN" ]; then
  echo ""
  echo "⚠️  Token retrieval requires interactive authentication."
  echo "Please run these commands manually:"
  echo ""
  echo "  az logout"
  echo "  az login --scope https://servicebus.azure.net/.default"
  echo ""
  read -p "Press Enter after you've completed the login, then we'll continue..."
  
  # Try getting token again after interactive login
  ACCESS_TOKEN=$(az account get-access-token --scope https://servicebus.azure.net/.default --query accessToken -o tsv)
  
  if [ -z "$ACCESS_TOKEN" ]; then
    echo "❌ Still unable to get access token. Please contact your instructor."
    exit 1
  fi
fi

# Service Bus details
SB_FQDN="$SERVICE_BUS_NAMESPACE.servicebus.windows.net"
QUEUE_NAME="orders"

echo "📨 Sending 20 messages to Service Bus queue using Entra ID authentication..."

# Send messages using REST API with Entra ID token
for i in {1..20}; do
  MESSAGE_BODY="Order message $i - $(date +%T)"
  
  # Send message using Service Bus REST API with Bearer token
  HTTP_CODE=$(curl -X POST "https://$SB_FQDN/$QUEUE_NAME/messages" \
    -H "Authorization: Bearer $ACCESS_TOKEN" \
    -H "Content-Type: application/json" \
    -H "BrokerProperties: {}" \
    -d "$MESSAGE_BODY" \
    --silent --write-out "%{http_code}" --output /dev/null)
  
  if [ "$HTTP_CODE" -eq 201 ]; then
    echo "✅ Sent message $i"
  else
    echo "⚠️  Failed to send message $i (HTTP $HTTP_CODE)"
  fi
done

echo "✅ Message sending complete"
```

**Note**: This approach uses Entra ID authentication (no connection strings or SAS tokens required), which is more secure and works when local authentication is disabled on Service Bus.

## 📊 Step 11: Check Queue Message Count

Before observing scaling, verify messages are in the queue:

```bash
# Check queue message count
echo "📊 Checking queue message count..."
QUEUE_COUNT=$(az servicebus queue show \
  --resource-group $RESOURCE_GROUP \
  --namespace-name $SERVICE_BUS_NAMESPACE \
  --name orders \
  --query messageCount \
  -o tsv)

echo "Messages in queue: $QUEUE_COUNT"

if [ "$QUEUE_COUNT" -eq 0 ]; then
  echo ""
  echo "⚠️  Queue is empty! Please add messages using Azure Portal (see Step 10 instructions)"
  echo "Once you've added messages, run this step again to verify."
else
  echo "✅ Queue has messages - ready to observe KEDA scaling!"
fi
```

## 📊 Step 12: Observe KEDA Scaling Up

```bash
# Monitor scaling behavior and message processing
echo "📊 Monitoring KEDA scaling (checking every 10 seconds for 2 minutes):"

for i in {1..12}; do
  echo "=== Check $i/12 ($(date +%T)) ==="
  
  # Check ScaledObject status
  kubectl get scaledobject servicebus-scaler -n keda-demo
  
  # Check pod count
  POD_COUNT=$(kubectl get pods -n keda-demo -l app=servicebus-processor --no-headers 2>/dev/null | wc -l)
  echo "Pod count: $POD_COUNT"
  
  # Check queue length
  QUEUE_LENGTH=$(az servicebus queue show \
    --resource-group $RESOURCE_GROUP \
    --namespace-name $SERVICE_BUS_NAMESPACE \
    --name orders \
    --query messageCount -o tsv)
  echo "Queue length: $QUEUE_LENGTH"
  
  echo -e "\n---"
  sleep 10
done

# Show recent processor logs
echo -e "\n📝 Recent message processing logs:"
kubectl logs -n keda-demo -l app=servicebus-processor --tail=10 2>/dev/null || echo "No pods running yet"
```

Expected output:
- Pods should scale up based on queue length (approximately 1 pod per 5 messages)
- Queue length should decrease as messages are processed and completed
- Pods will show logs of receiving and completing messages

## 🔍 Step 13: View Scaling Events

```bash
# Check KEDA events
echo "📋 KEDA Scaling Events:"
kubectl get events -n keda-demo --sort-by='.lastTimestamp' | grep -i "scaled\|keda"

# Check deployment status
kubectl get deployment servicebus-processor -n keda-demo

# Check HPA created by KEDA
kubectl describe hpa -n keda-demo

# Verify current state
echo -e "\n📊 Current State:"
echo "Pods running: $(kubectl get pods -n keda-demo -l app=servicebus-processor --no-headers 2>/dev/null | wc -l)"
echo "Queue messages: $(az servicebus queue show --resource-group $RESOURCE_GROUP --namespace-name $SERVICE_BUS_NAMESPACE --name orders --query messageCount -o tsv)"
```

## ⏳ Step 14: Monitor Scale-to-Zero

```bash
# Monitor scale down as queue empties
echo "⏳ Monitoring scale-to-zero as messages are consumed (checking every 20 seconds for 3 minutes):"

for i in {1..9}; do
  echo "=== Check $i/9 ($(date +%T)) ==="
  
  kubectl get scaledobject servicebus-scaler -n keda-demo
  POD_COUNT=$(kubectl get pods -n keda-demo -l app=servicebus-processor --no-headers 2>/dev/null | wc -l)
  echo "Pod count: $POD_COUNT"
  
  # Check queue length
  QUEUE_LENGTH=$(az servicebus queue show \
    --resource-group $RESOURCE_GROUP \
    --namespace-name $SERVICE_BUS_NAMESPACE \
    --name orders \
    --query messageCount -o tsv)
  echo "Queue length: $QUEUE_LENGTH"
  
  echo -e "\n---"
  sleep 20
done

echo "📊 Final pod count: $(kubectl get pods -n keda-demo -l app=servicebus-processor --no-headers 2>/dev/null | wc -l)"
```

Expected behavior:
- As the queue empties (messages processed), KEDA detects fewer messages
- After cooldown period (60s) with an empty queue, pods scale back to 0
- You can view pod logs to see message processing: `kubectl logs -n keda-demo -l app=servicebus-processor --tail=20`

## 🌐 Step 16: Install KEDA HTTP Add-on

Now let's explore HTTP-based scaling:

```bash
# Add KEDA Helm repository if not already added
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# Install KEDA HTTP Add-on
helm install http-add-on kedacore/keda-add-ons-http \
  --namespace keda \
  --set interceptor.replicas.min=1 \
  --set interceptor.replicas.max=10

echo "⏳ Waiting for HTTP Add-on to be ready..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/component=interceptor -n keda --timeout=120s

echo "✅ KEDA HTTP Add-on installed"
```

## 📦 Step 17: Deploy HTTP Application

```bash
# Deploy a simple HTTP application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-app
  namespace: keda-demo
spec:
  replicas: 0  # Start with 0 replicas
  selector:
    matchLabels:
      app: http-app
  template:
    metadata:
      labels:
        app: http-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: http-app
  namespace: keda-demo
spec:
  selector:
    app: http-app
  ports:
  - port: 80
    targetPort: 80
EOF

echo "✅ HTTP application deployed"
```

## 📊 Step 18: Create HTTPScaledObject

```bash
# Create HTTPScaledObject
cat <<EOF | kubectl apply -f -
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: http-app-scaler
  namespace: keda-demo
spec:
  hosts:
  - keda-demo-http.local
  pathPrefixes:
  - /
  scaleTargetRef:
    name: http-app
    kind: Deployment
    apiVersion: apps/v1
    service: http-app
    port: 80
  targetPendingRequests: 5
  replicas:
    min: 0
    max: 10
EOF

echo "✅ HTTPScaledObject created"
echo ""
echo "⏳ Waiting for interceptor proxy to be configured..."
sleep 10

# Verify the HTTPScaledObject
kubectl get httpscaledobject -n keda-demo
kubectl describe httpscaledobject http-app-scaler -n keda-demo
```

## 🔍 Step 19: Test HTTP Scaling

```bash
# Check HTTPScaledObject status
kubectl get httpscaledobject -n keda-demo

# Get the interceptor proxy service endpoint
echo "🔍 Getting interceptor proxy service..."
INTERCEPTOR_SVC=$(kubectl get svc -n keda -l app.kubernetes.io/component=interceptor -o jsonpath='{.items[0].metadata.name}')
echo "Interceptor service: $INTERCEPTOR_SVC"

# Generate HTTP traffic through the interceptor
echo "📨 Generating HTTP traffic..."
echo "Traffic must go through interceptor proxy with Host header matching HTTPScaledObject host"

# Run load generator in background
kubectl run http-load-generator \
  --image=curlimages/curl:latest \
  --namespace=keda-demo \
  --restart=Never \
  -- sh -c 'while true; do curl -s -H "Host: keda-demo-http.local" http://keda-add-ons-http-interceptor-proxy.keda.svc.cluster.local:8080/; sleep 0.1; done' &

echo "✅ Load generator started in background"
echo ""
echo "⏳ Monitoring HTTP scaling (press Ctrl+C to stop)..."

# Monitor scaling
for i in {1..20}; do
  clear
  echo "=== HTTP Scaling Monitor - Check $i/20 ($(date +%T)) ==="
  echo ""
  kubectl get httpscaledobject http-app-scaler -n keda-demo
  echo ""
  echo "Pod count: $(kubectl get pods -n keda-demo -l app=http-app --no-headers 2>/dev/null | wc -l)"
  echo ""
  kubectl get pods -n keda-demo -l app=http-app
  echo ""
  echo "Press Ctrl+C to stop monitoring"
  sleep 5
done

# Stop load generator
echo "🛑 Stopping load generator..."
kubectl delete pod http-load-generator -n keda-demo --ignore-not-found=true
```

**Note:** HTTP scaling requires traffic to go through the KEDA interceptor proxy with the correct Host header. The interceptor tracks pending requests and scales accordingly.

## 📊 Step 20: Compare KEDA with Standard HPA

Let's create a comparison:

```bash
# Deploy standard HPA application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: standard-hpa-app
  namespace: keda-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: standard-hpa-app
  template:
    metadata:
      labels:
        app: standard-hpa-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: standard-hpa
  namespace: keda-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: standard-hpa-app
  minReplicas: 1  # Cannot be 0
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
EOF

echo "✅ Standard HPA application deployed"

# Compare
echo -e "\n📊 Comparison:"
echo "Standard HPA - Min Replicas: 1 (always running)"
echo "KEDA ScaledObject - Min Replicas: 0 (scale-to-zero)"
echo ""
kubectl get deployment -n keda-demo
```

## 📊 Step 21: View All KEDA Resources

```bash
# List all ScaledObjects
echo "📊 ScaledObjects:"
kubectl get scaledobject -n keda-demo

# List all HTTPScaledObjects
echo -e "\n📊 HTTPScaledObjects:"
kubectl get httpscaledobject -n keda-demo

# List all TriggerAuthentications
echo -e "\n📊 TriggerAuthentications:"
kubectl get triggerauthentication -n keda-demo

# Check all HPAs (including KEDA-managed)
echo -e "\n📊 All HPAs:"
kubectl get hpa -n keda-demo

# View KEDA operator logs
echo -e "\n📋 Recent KEDA Operator Logs:"
kubectl logs -n keda -l app.kubernetes.io/name=keda-operator --tail=20
```

## 🎓 Summary

You have successfully:
- ✅ Installed KEDA and KEDA HTTP Add-on
- ✅ Created ScaledObject for Azure Service Bus queue
- ✅ Configured workload identity for Service Bus access
- ✅ Tested scale-to-zero based on queue length
- ✅ Deployed HTTPScaledObject for HTTP request-based scaling
- ✅ Compared KEDA with standard HPA
- ✅ Observed event-driven autoscaling behavior

## 🔑 Key Concepts Learned

1. **KEDA Architecture**:
   - Extends HPA with custom metrics
   - Supports 50+ event sources (scalers)
   - Enables scale-to-zero functionality

2. **ScaledObject**:
   - Defines scaling behavior and triggers
   - Creates underlying HPA automatically
   - Supports multiple triggers with different metrics

3. **Scale-to-Zero**:
   - Reduces costs when no work is pending
   - Requires external event source to trigger scale-up
   - Cooldown period prevents rapid scaling

4. **Event Sources**:
   - Azure Service Bus: Queue/topic-based scaling
   - HTTP: Request count-based scaling
   - Many others: Kafka, RabbitMQ, Redis, Prometheus, etc.

## 📝 Best Practices

- ✅ Use workload identity instead of connection strings
- ✅ Set appropriate cooldown periods (30-300s)
- ✅ Configure reasonable polling intervals (5-30s)
- ✅ Set maxReplicaCount to prevent runaway scaling
- ✅ Use scale-to-zero for batch/event-driven workloads
- ✅ Test scaling behavior under realistic load
- ✅ Monitor KEDA metrics and events
- ✅ Use KEDA for event-driven, HPA for resource-based scaling
- ✅ Consider cold start time when using scale-to-zero

## 🧹 Cleanup

```bash
# Delete load generators
kubectl delete pod http-load-generator -n keda-demo --ignore-not-found=true

# Delete namespace
kubectl delete namespace keda-demo

# Optionally uninstall KEDA (if not needed for other chapters)
# helm uninstall keda -n keda
# helm uninstall http-add-on -n keda
# kubectl delete namespace keda

echo "✅ Cleanup complete"
```

## 🔍 Troubleshooting

**Issue**: ScaledObject shows "Unknown" state
- Check KEDA operator logs: `kubectl logs -n keda -l app.kubernetes.io/name=keda-operator`
- Verify TriggerAuthentication is configured correctly
- Check workload identity federated credential exists

**Issue**: Pods not scaling up despite messages in queue
- Verify role assignment for Service Bus access
- Check polling interval isn't too long
- Review ScaledObject events: `kubectl describe scaledobject <name>`

**Issue**: Pods not scaling to zero
- Wait for cooldown period to pass
- Check if queue/event source is truly empty
- Verify minReplicaCount is set to 0

**Issue**: HTTP scaling not working
- Verify HTTP Add-on is installed and running
- Check interceptor service is accessible
- Review HTTPScaledObject configuration

**Issue**: Workload identity authentication failing
- Verify federated credential subject matches service account
- Check identity has proper role assignments
- Wait for role propagation (30-60 seconds)

## 🚀 Next Steps

Proceed to:
- **[Chapter 5: Pod Disruption Budgets](../chapter-5-pdb/README.md)**

In the next chapter, you'll learn how to maintain application availability during voluntary disruptions using Pod Disruption Budgets.
