# Chapter 2: LimitRange and ResourceQuota

In this chapter, you'll learn how to enforce resource policies at the namespace level using LimitRange and ResourceQuota to manage resources in multi-tenant clusters.

## 🎯 Learning Objectives

- Understand the purpose of LimitRange and ResourceQuota
- Configure default resource requests and limits with LimitRange
- Implement namespace-level resource constraints with ResourceQuota
- Test quota enforcement and resource tracking
- Manage resources in multi-tenant scenarios

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Completed [Chapter 1: Resource Requests and Limits](../chapter-1-requests-limits/README.md)
- AKS cluster running with metrics-server installed

## 🔄 Load Your Configuration

```powershell
# Load your lab configuration
. ./lab-config.sh

# Verify cluster access
kubectl get nodes
```

## 📖 Understanding LimitRange and ResourceQuota

### LimitRange
- **Purpose**: Sets default and maximum/minimum resource values for containers in a namespace
- **Scope**: Applies per pod/container
- **Use Cases**:
  - Enforce default resource requests/limits
  - Prevent excessively large or small resource requests
  - Ensure QoS consistency
  - Control the ratio between limits and requests

#### Understanding maxLimitRequestRatio

The `maxLimitRequestRatio` is a critical LimitRange setting that controls resource overcommitment:

**Formula**: `ratio = limit ÷ request`

**Example**:
```yaml
maxLimitRequestRatio:
  cpu: 4      # Limit can be at most 4x the request
  memory: 2   # Limit can be at most 2x the request
```

**Why This Matters**:

Without ratio limits, you could have configurations like:
- Request: 10m CPU (reserves almost nothing)
- Limit: 8000m CPU (can consume 8 entire CPUs!)
- Ratio: 8000 ÷ 10 = 800 😱

This creates severe overcommitment where pods reserve minimal resources but can burst to consume massive amounts, starving neighboring pods.

With `maxLimitRequestRatio: 4`, you enforce balanced configurations:
- Request: 500m CPU (reserves reasonable baseline)
- Limit: 2000m CPU (can burst to 2 CPUs)
- Ratio: 2000 ÷ 500 = 4 ✅

This ensures pods that want high burst capacity must also reserve a proportional baseline.

### ResourceQuota
- **Purpose**: Limits total resource consumption in a namespace
- **Scope**: Applies to the entire namespace
- **Use Cases**:
  - Multi-tenant resource management
  - Cost control
  - Prevent resource exhaustion

## 🏗️ Step 1: Create Namespaces for Testing

```powershell
# Create namespaces
kubectl create namespace limitrange-demo
kubectl create namespace quota-demo
kubectl create namespace team-a
kubectl create namespace team-b

Write-Host "✅ Namespaces created"
```

## 📏 Step 2: Create a LimitRange

Let's create a LimitRange that sets defaults and constraints:

```powershell
# Create LimitRange
@"
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: limitrange-demo
spec:
  limits:
  # Container-level limits
  - type: Container
    max:
      cpu: "2"
      memory: "1Gi"
    min:
      cpu: "50m"
      memory: "32Mi"
    default:
      cpu: "400m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    maxLimitRequestRatio:
      cpu: 4
      memory: 2
  # Pod-level limits
  - type: Pod
    max:
      cpu: "4"
      memory: "2Gi"
"@ | kubectl apply -f -

Write-Host "✅ LimitRange created"
```

### LimitRange Configuration Explained

- **max**: Maximum resources a container can request
- **min**: Minimum resources a container must request
- **default**: Default limits if not specified
- **defaultRequest**: Default requests if not specified
- **maxLimitRequestRatio**: Maximum ratio between limit and request (prevents excessive overcommit)

## 🔍 Step 3: Verify LimitRange

```powershell
# View the LimitRange
kubectl describe limitrange resource-limits -n limitrange-demo

# View in YAML format
kubectl get limitrange resource-limits -n limitrange-demo -o yaml
```

## 📦 Step 4: Deploy Pod Without Resource Specs

Deploy a pod without specifying resources - it should get defaults from LimitRange:

```powershell
# Create pod without resource specifications
@"
apiVersion: v1
kind: Pod
metadata:
  name: defaulted-pod
  namespace: limitrange-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
"@ | kubectl apply -f -

Write-Host "✅ Pod deployed without resource specs"
```

## 🔍 Step 5: Verify Default Resources Applied

```powershell
# Wait for pod to be ready
kubectl wait --for=condition=Ready pod/defaulted-pod -n limitrange-demo --timeout=60s

# Check the resources that were automatically applied
Write-Host "📊 Automatically Applied Resources:"
kubectl get pod defaulted-pod -n limitrange-demo -o jsonpath='{.spec.containers[0].resources}' | jq

# Detailed view
kubectl describe pod defaulted-pod -n limitrange-demo | Select-String -Pattern "Limits|Requests" -Context 0,10
```

Expected: The pod should have the default requests and limits from the LimitRange.

## ⚠️ Step 6: Test LimitRange Enforcement

Try to create a pod that violates the LimitRange constraints:

```powershell
# Try to create a pod with too much memory
@"
apiVersion: v1
kind: Pod
metadata:
  name: too-large-pod
  namespace: limitrange-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    resources:
      requests:
        memory: "2Gi"
        cpu: "100m"
      limits:
        memory: "2Gi"
        cpu: "400m"
"@ | kubectl apply -f -
```

Expected: This should fail because 2Gi exceeds the max memory (1Gi) defined in the LimitRange.

## 🧪 Step 7: Test Minimum Constraints

Try to create a pod with resources below the minimum:

```powershell
# Try to create a pod with too little CPU
@"
apiVersion: v1
kind: Pod
metadata:
  name: too-small-pod
  namespace: limitrange-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    resources:
      requests:
        memory: "64Mi"
        cpu: "10m"
      limits:
        memory: "128Mi"
        cpu: "100m"
"@ | kubectl apply -f -
```

Expected: This should fail because 10m CPU is below the minimum (50m).

## ✅ Step 8: Create a Valid Pod

Create a pod that fits within the LimitRange constraints:

```powershell
# Create a pod with valid resource specifications
@"
apiVersion: v1
kind: Pod
metadata:
  name: valid-pod
  namespace: limitrange-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "400m"
    ports:
    - containerPort: 80
"@ | kubectl apply -f -

Write-Host "✅ Valid pod created"

# Verify it's running
kubectl wait --for=condition=Ready pod/valid-pod -n limitrange-demo --timeout=60s
kubectl get pod valid-pod -n limitrange-demo
```

## 📊 Step 9: Create a ResourceQuota

Now let's create a ResourceQuota to limit total resources in a namespace:

```powershell
# Create ResourceQuota
@"
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: quota-demo
spec:
  hard:
    # Compute resources
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    
    # Object counts
    pods: "10"
    services: "5"
    persistentvolumeclaims: "4"
    
    # Storage
    requests.storage: "100Gi"
"@ | kubectl apply -f -

Write-Host "✅ ResourceQuota created"
```

## 🔍 Step 10: Verify ResourceQuota

```powershell
# View the ResourceQuota
kubectl describe resourcequota team-quota -n quota-demo

# Get quota status
kubectl get resourcequota team-quota -n quota-demo -o yaml
```

## 📦 Step 11: Deploy Pods to Test Quota

Deploy multiple pods to consume quota:

```powershell
# Create a deployment
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-test
  namespace: quota-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: quota-test
  template:
    metadata:
      labels:
        app: quota-test
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
"@ | kubectl apply -f -

Write-Host "✅ Deployment created"
```

## 📊 Step 12: Monitor Quota Usage

```powershell
# Wait for pods to be created
Start-Sleep -Seconds 10

# Check quota usage
Write-Host "📊 Quota Usage:"
kubectl describe resourcequota team-quota -n quota-demo

# Check pods
kubectl get pods -n quota-demo

# View detailed quota status
kubectl get resourcequota team-quota -n quota-demo -o jsonpath='{.status}' | jq
```

Expected: You should see the quota usage increase as pods are deployed.

## ⚠️ Step 13: Test Quota Enforcement

Try to scale the deployment beyond the quota limits:

```powershell
# Try to scale to 20 replicas (should exceed pod quota of 10)
kubectl scale deployment quota-test -n quota-demo --replicas=20

# Wait a moment
Start-Sleep -Seconds 5

# Check the actual number of pods
Write-Host "📊 Pod Count:"
(kubectl get pods -n quota-demo --no-headers | Measure-Object -Line).Lines

# Check deployment status
kubectl get deployment quota-test -n quota-demo

# Check ReplicaSet events for quota errors
Write-Host "`n📋 ReplicaSet Events:"
kubectl get events -n quota-demo --sort-by='.lastTimestamp' | Select-String -Pattern "quota" -CaseSensitive:$false
```

Expected: Only 10 pods should be created (the quota limit), and events should show quota exceeded errors.

## 🔄 Step 14: Create Multi-Tenant Scenario

Let's create a realistic multi-tenant setup with separate quotas for different teams:

```powershell
# Team A Quota (larger team)
@"
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
    limits.cpu: "16"
    limits.memory: "32Gi"
    pods: "20"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: team-a-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "2Gi"
"@ | kubectl apply -f -

Write-Host "✅ Team A quota and limits created"

# Team B Quota (smaller team)
@"
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-b-quota
  namespace: team-b
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: team-b-limits
  namespace: team-b
spec:
  limits:
  - type: Container
    default:
      cpu: "250m"
      memory: "256Mi"
    defaultRequest:
      cpu: "50m"
      memory: "64Mi"
    max:
      cpu: "1"
      memory: "1Gi"
"@ | kubectl apply -f -

Write-Host "✅ Team B quota and limits created"
```

## 📊 Step 15: Deploy Workloads for Both Teams

```powershell
# Deploy Team A workload
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: team-a-app
  namespace: team-a
spec:
  replicas: 5
  selector:
    matchLabels:
      app: team-a-app
  template:
    metadata:
      labels:
        app: team-a-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
"@ | kubectl apply -f -

Write-Host "✅ Team A workload deployed"

# Deploy Team B workload
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: team-b-app
  namespace: team-b
spec:
  replicas: 5
  selector:
    matchLabels:
      app: team-b-app
  template:
    metadata:
      labels:
        app: team-b-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
"@ | kubectl apply -f -

Write-Host "✅ Team B workload deployed"
```

## 🔍 Step 16: Compare Team Resource Usage

```powershell
# Wait for deployments
Start-Sleep -Seconds 15

# Compare quotas
Write-Host "📊 Team A Quota Usage:"
kubectl describe resourcequota team-a-quota -n team-a | Select-String -Pattern "Used" -Context 0,10

Write-Host "`n📊 Team B Quota Usage:"
kubectl describe resourcequota team-b-quota -n team-b | Select-String -Pattern "Used" -Context 0,10

# Compare actual pod resources
Write-Host "`n📊 Team A Pod Resources:"
kubectl get pods -n team-a -o custom-columns=NAME:.metadata.name,CPU-REQ:.spec.containers[0].resources.requests.cpu,MEM-REQ:.spec.containers[0].resources.requests.memory,CPU-LIM:.spec.containers[0].resources.limits.cpu,MEM-LIM:.spec.containers[0].resources.limits.memory

Write-Host "`n📊 Team B Pod Resources:"
kubectl get pods -n team-b -o custom-columns=NAME:.metadata.name,CPU-REQ:.spec.containers[0].resources.requests.cpu,MEM-REQ:.spec.containers[0].resources.requests.memory,CPU-LIM:.spec.containers[0].resources.limits.cpu,MEM-LIM:.spec.containers[0].resources.limits.memory
```

## 📈 Step 17: View Cluster-Wide Resource Summary

```powershell
# Get all quotas across namespaces
Write-Host "📊 All ResourceQuotas:"
kubectl get resourcequota --all-namespaces

# Get all LimitRanges
Write-Host "`n📏 All LimitRanges:"
kubectl get limitrange --all-namespaces

# Summary of resource usage
Write-Host "`n📊 Node Resources:"
kubectl top nodes

# Check which pods are using the most resources
Write-Host "`n📊 Top Resource Consuming Pods:"
kubectl top pods --all-namespaces --sort-by=memory | Select-Object -First 20
```

## 🎓 Summary

You have successfully:
- ✅ Created LimitRange to set default and maximum/minimum resource values
- ✅ Tested LimitRange enforcement with various pod configurations
- ✅ Created ResourceQuota to limit total namespace resources
- ✅ Observed quota enforcement when exceeding limits
- ✅ Implemented multi-tenant resource management for different teams
- ✅ Compared resource usage across teams

## 🔑 Key Concepts Learned

1. **LimitRange**:
   - Sets per-container defaults and constraints
   - Enforced at pod creation time
   - Prevents creation of non-compliant pods
   - Useful for ensuring QoS consistency

2. **ResourceQuota**:
   - Limits total resource consumption in a namespace
   - Tracks actual usage vs limits
   - Prevents new pods if quota would be exceeded
   - Essential for multi-tenant clusters

3. **Multi-Tenancy**:
   - Each team gets dedicated namespace with own quota
   - Prevents one team from consuming all cluster resources
   - Enables fair resource sharing and cost allocation

## 📝 Best Practices

- ✅ Always use LimitRange to set defaults in production namespaces
- ✅ Implement ResourceQuota for all tenant namespaces
- ✅ Set quotas based on team size and workload requirements
- ✅ Monitor quota usage regularly
- ✅ Leave buffer in quotas for peak loads
- ✅ Use both object count and resource quotas
- ✅ Document quota limits and request increases through proper channels

## 🧹 Cleanup

```powershell
# Delete all demo namespaces
kubectl delete namespace limitrange-demo quota-demo team-a team-b

Write-Host "✅ Cleanup complete"
```

## 🔍 Troubleshooting

**Issue**: Pods stuck in Pending with "insufficient quota" error
- Check quota status: `kubectl describe resourcequota -n <namespace>`
- Either scale down existing pods or request quota increase
- Check if quota includes both requests and limits

**Issue**: Pod rejected due to LimitRange violation
- Review LimitRange: `kubectl describe limitrange -n <namespace>`
- Adjust pod resources to fit within constraints
- Or update LimitRange if constraints are too restrictive

**Issue**: Default resources not applied to pods
- Verify LimitRange exists in the namespace
- Check LimitRange has `default` and `defaultRequest` specified
- LimitRange only applies at pod creation time (not to existing pods)

## 🚀 Next Steps

Proceed to:
- **[Chapter 3: Horizontal Pod Autoscaler (HPA)](../chapter-3-hpa/README.md)**

In the next chapter, you'll learn how to automatically scale your applications based on resource utilization using the Horizontal Pod Autoscaler.
