# Chapter 1: Resource Requests and Limits

In this chapter, you'll learn how to configure resource requests and limits for your pods to ensure optimal scheduling and resource utilization in Kubernetes.

## 🎯 Learning Objectives

- Understand the difference between resource requests and limits
- Configure CPU and memory requests and limits
- Observe Quality of Service (QoS) classes: Guaranteed, Burstable, and BestEffort
- Test resource enforcement and out-of-memory (OOM) scenarios
- Monitor resource usage with `kubectl top`

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- AKS cluster running with metrics-server installed
- kubectl configured to access the cluster

## 🔄 Load Your Configuration

```powershell
# Load your lab configuration
. .\lab-config.ps1

# Verify cluster access
kubectl get nodes
```

## 📖 Understanding Requests and Limits

### Resource Requests
- **Definition**: The amount of resources guaranteed to the container
- **Purpose**: Used by the scheduler to decide which node to place the pod on
- **Guarantee**: The container will always have at least this amount available

### Resource Limits
- **Definition**: The maximum amount of resources a container can use
- **Purpose**: Prevents a container from consuming all node resources
- **Enforcement**: Container will be throttled (CPU) or killed (memory) if it exceeds limits

### Quality of Service (QoS) Classes

Kubernetes assigns QoS classes based on requests and limits:

| QoS Class      | Condition                                     | Scheduling Priority | Eviction Priority        |
| -------------- | --------------------------------------------- | ------------------- | ------------------------ |
| **Guaranteed** | requests == limits for all resources          | Highest             | Lowest (last to evict)   |
| **Burstable**  | At least one container has requests or limits | Medium              | Medium                   |
| **BestEffort** | No requests or limits set                     | Lowest              | Highest (first to evict) |

## 🏗️ Step 1: Create a Namespace

```powershell
# Create namespace for this chapter
kubectl create namespace resources-demo

Write-Host "✅ Namespace created"
```

## 📊 Step 2: Deploy a Pod with Requests Only (Burstable)

Let's deploy a pod with only requests defined:

```powershell
# Create deployment with requests only
@"
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
  namespace: resources-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
"@ | kubectl apply -f -

Write-Host "✅ Burstable pod deployed"
```

## 🔍 Step 3: Check QoS Class

```powershell
# Wait for pod to be running
kubectl wait --for=condition=Ready pod/burstable-pod -n resources-demo --timeout=60s

# Check the QoS class
kubectl get pod burstable-pod -n resources-demo -o jsonpath='{.status.qosClass}'
Write-Host ""

# Get detailed pod information
kubectl describe pod burstable-pod -n resources-demo | Select-String -Pattern "QoS Class" -Context 0,10

# Check resource usage
Write-Host "`n📊 Resource Usage:"
kubectl top pod burstable-pod -n resources-demo
```

Expected: QoS Class should be **Burstable** because we defined requests but no limits.

## 🎯 Step 4: Deploy a Pod with Requests and Limits (Guaranteed)

Now deploy a pod where requests equal limits:

```powershell
# Create deployment with requests == limits
@"
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
  namespace: resources-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "250m"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "100M", "--vm-hang", "1"]
"@ | kubectl apply -f -

Write-Host "✅ Guaranteed pod deployed"
```

## 🔍 Step 5: Verify Guaranteed QoS

```powershell
# Wait for pod to be running
kubectl wait --for=condition=Ready pod/guaranteed-pod -n resources-demo --timeout=60s

# Check the QoS class
Write-Host "QoS Class:"
kubectl get pod guaranteed-pod -n resources-demo -o jsonpath='{.status.qosClass}'
Write-Host ""

# Compare both pods
Write-Host "`n📊 Comparison of QoS Classes:"
kubectl get pods -n resources-demo -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass,CPU-REQUEST:.spec.containers[0].resources.requests.cpu,MEM-REQUEST:.spec.containers[0].resources.requests.memory,CPU-LIMIT:.spec.containers[0].resources.limits.cpu,MEM-LIMIT:.spec.containers[0].resources.limits.memory
```

Expected: QoS Class should be **Guaranteed**.

## ⚠️ Step 6: Deploy a Pod with No Resources (BestEffort)

Deploy a pod without any resource specifications:

```powershell
# Create deployment with no resources
@"
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
  namespace: resources-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
"@ | kubectl apply -f -

Write-Host "✅ BestEffort pod deployed"
```

## 🔍 Step 7: Verify All QoS Classes

```powershell
# Wait for pod to be running
kubectl wait --for=condition=Ready pod/besteffort-pod -n resources-demo --timeout=60s

# Show all pods with their QoS classes
Write-Host "📊 All Pods and Their QoS Classes:"
kubectl get pods -n resources-demo -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass,STATUS:.status.phase

# Detailed view
$pods = kubectl get pods -n resources-demo -o json | ConvertFrom-Json
$pods.items | ForEach-Object {
  Write-Host "$($_.metadata.name): $($_.status.qosClass)"
}
```

## 💥 Step 8: Test Memory Limit Enforcement (OOM Kill)

Deploy a pod that will try to use more memory than its limit:

```powershell
# Create a pod that will exceed memory limits
@"
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
  namespace: resources-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
        cpu: "100m"
      limits:
        memory: "100Mi"
        cpu: "200m"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
"@ | kubectl apply -f -

Write-Host "✅ OOM test pod deployed"
Write-Host "⏳ Waiting for OOM kill (this should happen within 30 seconds)..."
```

## 🔍 Step 9: Observe OOM Kill

```powershell
# Wait a moment for the pod to start and get killed
Start-Sleep -Seconds 20

# Check pod status
kubectl get pod oom-pod -n resources-demo

# Check for OOMKilled status
kubectl get pod oom-pod -n resources-demo -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
Write-Host ""

# Get detailed termination info
Write-Host "`n📊 Termination Details:"
kubectl describe pod oom-pod -n resources-demo | Select-String -Pattern "Last State" -Context 0,10

# Check events
Write-Host "`n📋 Events:"
kubectl get events -n resources-demo --sort-by='.lastTimestamp' | Select-String -Pattern "oom-pod"
```

Expected: The pod should show status **OOMKilled** (Out Of Memory Killed).

## 🔄 Step 10: Test CPU Throttling

Deploy a pod that will be CPU-throttled:

```powershell
# Create a pod with CPU limits
@"
apiVersion: v1
kind: Pod
metadata:
  name: cpu-throttle-pod
  namespace: resources-demo
spec:
  restartPolicy: Never
  containers:
  - name: stress
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
        cpu: "100m"
      limits:
        memory: "100Mi"
        cpu: "200m"
    command: ["stress"]
    args: ["--cpu", "4", "--timeout", "120s"]
"@ | kubectl apply -f -

Write-Host "✅ CPU throttle test pod deployed"
```

## 📊 Step 11: Monitor CPU Throttling

```powershell
# Wait for pod to be running
kubectl wait --for=condition=Ready pod/cpu-throttle-pod -n resources-demo --timeout=60s

# Monitor CPU usage in real-time
Write-Host "📊 Monitoring CPU usage (should be capped at ~200m):"
1..10 | ForEach-Object {
  $i = $_
  Write-Host "=== Sample $i/10 ==="
  $topOutput = kubectl top pod cpu-throttle-pod -n resources-demo 2>$null
  if ($LASTEXITCODE -eq 0) {
    $topOutput
  } else {
    Write-Host "Waiting for metrics..."
  }
  Start-Sleep -Seconds 3
}

# Show final status
kubectl get pod cpu-throttle-pod -n resources-demo
```

Expected: CPU usage should be capped at approximately 200m (the limit), even though the stress command tries to use more.

## 📦 Step 12: Deploy a Realistic Application

Let's deploy a more realistic web application with proper resource configuration:

```powershell
# Create a deployment with proper resource settings
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: resources-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: resources-demo
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
"@ | kubectl apply -f -

Write-Host "✅ Web application deployed"
```

## 🔍 Step 13: Monitor Application Resources

```powershell
# Wait for deployment to be ready
kubectl wait --for=condition=available deployment/web-app -n resources-demo --timeout=120s

# Check pod status
kubectl get pods -n resources-demo -l app=web-app

# Check resource usage for all web app pods
Write-Host "`n📊 Web App Resource Usage:"
kubectl top pods -n resources-demo -l app=web-app

# Show QoS for web app pods
Write-Host "`n🏷️ QoS Classes:"
kubectl get pods -n resources-demo -l app=web-app -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass

# Summary view of all resources
Write-Host "`n📊 All Pods Summary:"
kubectl top pods -n resources-demo
```

## 📊 Step 14: Node Resource Analysis

```powershell
# Check node resource allocation
Write-Host "📊 Node Resource Allocation:"
kubectl top nodes

# Show resource requests per node
Write-Host "`n📈 Detailed Node Resource Usage:"
kubectl describe nodes | Select-String -Pattern "Allocated resources" -Context 0,5

# Check which pods are on which nodes
Write-Host "`n🗺️ Pod Distribution Across Nodes:"
kubectl get pods -n resources-demo -o wide
```

## 🎓 Summary

You have successfully:
- ✅ Created pods with different resource configurations
- ✅ Observed three QoS classes: Guaranteed, Burstable, and BestEffort
- ✅ Tested memory limit enforcement (OOMKilled)
- ✅ Observed CPU throttling behavior
- ✅ Deployed a realistic application with proper resource settings
- ✅ Monitored resource usage across pods and nodes

## 🔑 Key Concepts Learned

1. **Requests vs Limits**:
   - Requests: Guaranteed resources for scheduling
   - Limits: Maximum resources before throttling/killing

2. **QoS Classes**:
   - Guaranteed: Most stable, last to be evicted
   - Burstable: Can burst above requests up to limits
   - BestEffort: First to be evicted under pressure

3. **Resource Enforcement**:
   - Memory: Hard limit - pod killed if exceeded (OOMKilled)
   - CPU: Soft limit - pod throttled if exceeded (not killed)

4. **Scheduling Impact**:
   - Scheduler uses requests to determine node placement
   - Limits don't affect scheduling decisions

## 📝 Best Practices

- ✅ Always set requests for production workloads
- ✅ Set limits to prevent resource exhaustion
- ✅ Use Guaranteed QoS for critical workloads
- ✅ Monitor actual usage to right-size requests and limits
- ✅ Leave some buffer between requests and limits for burstable workloads
- ✅ Test with realistic load to validate resource settings

## 🧹 Cleanup

```powershell
# Delete all resources in the namespace
kubectl delete namespace resources-demo

Write-Host "✅ Cleanup complete"
```

## 🔍 Troubleshooting

**Issue**: Pods stuck in Pending state
- Check if nodes have enough resources: `kubectl describe node`
- Reduce resource requests or add more nodes

**Issue**: Pods frequently OOMKilled
- Increase memory limits
- Investigate memory leaks in the application
- Check actual usage: `kubectl top pod`

**Issue**: Application performance degraded
- Check if CPU is being throttled
- Increase CPU limits or optimize application

**Issue**: Metrics not available
- Verify metrics-server is running: `kubectl get pods -n kube-system -l k8s-app=metrics-server`
- Wait 1-2 minutes for metrics collection

## 🚀 Next Steps

Proceed to:
- **[Chapter 2: LimitRange and ResourceQuota](../chapter-2-limitrange-quota/README.md)**

In the next chapter, you'll learn how to enforce resource policies at the namespace level using LimitRange and ResourceQuota.
