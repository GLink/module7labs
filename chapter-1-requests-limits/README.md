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

```bash
# Load your lab configuration
source ./lab-config.sh

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

```bash
# Create namespace for this chapter
kubectl create namespace resources-demo

echo "✅ Namespace created"
```

## 📊 Step 2: Deploy a Pod with Requests Only (Burstable)

Let's deploy a pod with only requests defined:

```bash
# Create deployment with requests only
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ Burstable pod deployed"
```

## 🔍 Step 3: Check QoS Class

```bash
# Wait for pod to be running
kubectl wait --for=condition=Ready pod/burstable-pod -n resources-demo --timeout=60s

# Check the QoS class
kubectl get pod burstable-pod -n resources-demo -o jsonpath='{.status.qosClass}'
echo ""

# Get detailed pod information
kubectl describe pod burstable-pod -n resources-demo | grep -A 10 "QoS Class"

# Check resource usage
echo -e "\n📊 Resource Usage:"
kubectl top pod burstable-pod -n resources-demo
```

Expected: QoS Class should be **Burstable** because we defined requests but no limits.

## 🎯 Step 4: Deploy a Pod with Requests and Limits (Guaranteed)

Now deploy a pod where requests equal limits:

```bash
# Create deployment with requests == limits
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ Guaranteed pod deployed"
```

## 🔍 Step 5: Verify Guaranteed QoS

```bash
# Wait for pod to be running
kubectl wait --for=condition=Ready pod/guaranteed-pod -n resources-demo --timeout=60s

# Check the QoS class
echo "QoS Class:"
kubectl get pod guaranteed-pod -n resources-demo -o jsonpath='{.status.qosClass}'
echo ""

# Compare both pods
echo -e "\n📊 Comparison of QoS Classes:"
kubectl get pods -n resources-demo -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass,CPU-REQUEST:.spec.containers[0].resources.requests.cpu,MEM-REQUEST:.spec.containers[0].resources.requests.memory,CPU-LIMIT:.spec.containers[0].resources.limits.cpu,MEM-LIMIT:.spec.containers[0].resources.limits.memory
```

Expected: QoS Class should be **Guaranteed**.

## ⚠️ Step 6: Deploy a Pod with No Resources (BestEffort)

Deploy a pod without any resource specifications:

```bash
# Create deployment with no resources
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ BestEffort pod deployed"
```

## 🔍 Step 7: Verify All QoS Classes

```bash
# Wait for pod to be running
kubectl wait --for=condition=Ready pod/besteffort-pod -n resources-demo --timeout=60s

# Show all pods with their QoS classes
echo "📊 All Pods and Their QoS Classes:"
kubectl get pods -n resources-demo -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass,STATUS:.status.phase

# Detailed view
kubectl get pods -n resources-demo -o json | jq -r '.items[] | "\(.metadata.name): \(.status.qosClass)"'
```

## 💥 Step 8: Test Memory Limit Enforcement (OOM Kill)

Deploy a pod that will try to use more memory than its limit:

```bash
# Create a pod that will exceed memory limits
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ OOM test pod deployed"
echo "⏳ Waiting for OOM kill (this should happen within 30 seconds)..."
```

## 🔍 Step 9: Observe OOM Kill

```bash
# Wait a moment for the pod to start and get killed
sleep 20

# Check pod status
kubectl get pod oom-pod -n resources-demo

# Check for OOMKilled status
kubectl get pod oom-pod -n resources-demo -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
echo ""

# Get detailed termination info
echo -e "\n📊 Termination Details:"
kubectl describe pod oom-pod -n resources-demo | grep -A 10 "Last State"

# Check events
echo -e "\n📋 Events:"
kubectl get events -n resources-demo --sort-by='.lastTimestamp' | grep oom-pod
```

Expected: The pod should show status **OOMKilled** (Out Of Memory Killed).

## 🔄 Step 10: Test CPU Throttling

Deploy a pod that will be CPU-throttled:

```bash
# Create a pod with CPU limits
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ CPU throttle test pod deployed"
```

## 📊 Step 11: Monitor CPU Throttling

```bash
# Wait for pod to be running
kubectl wait --for=condition=Ready pod/cpu-throttle-pod -n resources-demo --timeout=60s

# Monitor CPU usage in real-time
echo "📊 Monitoring CPU usage (should be capped at ~200m):"
for i in {1..10}; do
  echo "=== Sample $i/10 ==="
  kubectl top pod cpu-throttle-pod -n resources-demo 2>/dev/null || echo "Waiting for metrics..."
  sleep 3
done

# Show final status
kubectl get pod cpu-throttle-pod -n resources-demo
```

Expected: CPU usage should be capped at approximately 200m (the limit), even though the stress command tries to use more.

## 📦 Step 12: Deploy a Realistic Application

Let's deploy a more realistic web application with proper resource configuration:

```bash
# Create a deployment with proper resource settings
cat <<EOF | kubectl apply -f -
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
EOF

echo "✅ Web application deployed"
```

## 🔍 Step 13: Monitor Application Resources

```bash
# Wait for deployment to be ready
kubectl wait --for=condition=available deployment/web-app -n resources-demo --timeout=120s

# Check pod status
kubectl get pods -n resources-demo -l app=web-app

# Check resource usage for all web app pods
echo -e "\n📊 Web App Resource Usage:"
kubectl top pods -n resources-demo -l app=web-app

# Show QoS for web app pods
echo -e "\n🏷️ QoS Classes:"
kubectl get pods -n resources-demo -l app=web-app -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass

# Summary view of all resources
echo -e "\n📊 All Pods Summary:"
kubectl top pods -n resources-demo
```

## 📊 Step 14: Node Resource Analysis

```bash
# Check node resource allocation
echo "📊 Node Resource Allocation:"
kubectl top nodes

# Show resource requests per node
echo -e "\n📈 Detailed Node Resource Usage:"
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check which pods are on which nodes
echo -e "\n🗺️ Pod Distribution Across Nodes:"
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

```bash
# Delete all resources in the namespace
kubectl delete namespace resources-demo

echo "✅ Cleanup complete"
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
