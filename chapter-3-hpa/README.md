# Chapter 3: Horizontal Pod Autoscaler (HPA)

In this chapter, you'll learn how to automatically scale your applications based on resource utilization using the Kubernetes Horizontal Pod Autoscaler (HPA).

## 🎯 Learning Objectives

- Understand how HPA works and its components
- Configure HPA based on CPU and memory metrics
- Perform load testing to trigger autoscaling
- Observe scale-up and scale-down behavior
- Configure HPA parameters (min, max, thresholds, cooldown)
- Understand HPA best practices and limitations

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- Completed [Chapter 1: Resource Requests and Limits](../chapter-1-requests-limits/README.md)
- AKS cluster running with metrics-server installed and working
- kubectl configured to access the cluster

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./lab-config.sh

# Verify cluster access and metrics-server
kubectl get nodes
kubectl top nodes
```

## 📖 Understanding Horizontal Pod Autoscaler

### How HPA Works

1. **Metrics Collection**: Metrics-server collects resource metrics from pods
2. **HPA Controller**: Periodically checks metrics (default: 15 seconds)
3. **Decision Making**: Compares current metrics against target thresholds
4. **Scaling Action**: Adjusts replica count by updating deployment/replicaset

**Critical Understanding**: HPA has NO built-in scaling thresholds. The target you specify IS the scaling logic.

Without HPA (or with no target specified):
- Your deployment stays at fixed replicas (e.g., 3 pods)
- Even if CPU hits 100%, nothing happens
- Even if CPU drops to 5%, nothing happens
- No automatic scaling occurs

With HPA and target specified:
- You tell HPA: "Keep average CPU around 50%"
- HPA continuously calculates: `(current avg CPU / 50%) = scaling factor`
- If avg CPU = 80%, HPA scales up: `(80% / 50%) = 1.6x` → add pods
- If avg CPU = 25%, HPA scales down: `(25% / 50%) = 0.5x` → remove pods

**The target IS the trigger** - it's not affecting some other scaling logic; it's creating the scaling behavior from scratch.

### Key Concepts

- **Target Utilization**: Desired average utilization across all pods (e.g., 50% CPU)
- **Scale-up**: Adds pods when current utilization > target
- **Scale-down**: Removes pods when current utilization < target
- **Cooldown**: Delays between scaling actions to prevent flapping

#### Understanding CPU Utilization Target

**Important**: HPA's CPU utilization target is based on **CPU requests**, not limits.

**How it works**:
- If a pod has `requests.cpu: 100m` and HPA target is `50%`
- HPA will scale when average CPU usage across pods exceeds **50m** (50% of 100m request)
- The pod's CPU limit (e.g., 400m) doesn't affect HPA scaling decisions
- HPA only cares about: `(actual CPU usage / CPU request) * 100%`

**Example**:
```yaml
resources:
  requests:
    cpu: "200m"    # HPA uses this as the baseline
  limits:
    cpu: "800m"    # HPA ignores this
```

With `targetCPUUtilizationPercentage: 50`:
- Scale up when: average CPU usage > 100m (50% of 200m request)
- Scale down when: average CPU usage < 100m
- The 800m limit only prevents individual pods from exceeding that cap

**Why use 50-70% targets?**
- Leaves headroom for traffic spikes before scaling completes
- Prevents constant scaling (flapping)
- Balances cost vs. performance
- Accounts for scaling lag (new pods take time to start)

### HPA Formula

```
desiredReplicas = ceil[currentReplicas * (currentMetric / targetMetric)]
```

## 🏗️ Step 1: Create a Namespace

```bash
# Create namespace for this chapter
kubectl create namespace hpa-demo

echo "✅ Namespace created"
```

## 📦 Step 2: Deploy an Application

Let's deploy a PHP application that we'll use for testing:

```bash
# Create deployment with resource requests (required for HPA)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: hpa-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  namespace: hpa-demo
spec:
  selector:
    app: php-apache
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

echo "✅ Application deployed"
```

## ✅ Step 3: Verify Application is Running

```bash
# Wait for deployment to be ready
kubectl wait --for=condition=available deployment/php-apache -n hpa-demo --timeout=120s

# Check pods
kubectl get pods -n hpa-demo

# Check resource usage
kubectl top pods -n hpa-demo
```

## 📊 Step 4: Create HPA Based on CPU

Create an HPA that targets 50% CPU utilization:

```bash
# Create HPA
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
  namespace: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 15
      selectPolicy: Max
EOF

echo "✅ HPA created"
```

### HPA Configuration Explained

- **minReplicas**: Minimum number of pods (1)
- **maxReplicas**: Maximum number of pods (10)
- **target.averageUtilization**: Target CPU utilization (50%)
- **scaleDown.stabilizationWindowSeconds**: Wait time before scale-down (60s)
- **scaleUp.stabilizationWindowSeconds**: Wait time before scale-up (0s = immediate)
- **policies**: Control how aggressively to scale (100% = double pods, or add 2 pods)

## 🔍 Step 5: Verify HPA Status

```bash
# Check HPA status
kubectl get hpa -n hpa-demo

# Detailed view
kubectl describe hpa php-apache-hpa -n hpa-demo

# Watch HPA in real-time
echo "📊 Watching HPA (press Ctrl+C to stop):"
kubectl get hpa php-apache-hpa -n hpa-demo --watch
```

Press Ctrl+C after a few seconds. You should see current/target metrics and current replica count.

## 🔥 Step 6: Generate Load to Trigger Scale-Up

Now let's generate load to trigger autoscaling:

```bash
# Create a load generator pod
kubectl run load-generator \
  --image=busybox \
  --namespace=hpa-demo \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

echo "✅ Load generator started"
echo "⏳ This will generate continuous load to trigger HPA scaling..."
```

## 📊 Step 7: Monitor Scale-Up Behavior

```bash
# Watch HPA scaling in real-time (in a separate terminal or periodically)
echo "📊 Monitoring HPA (checking every 15 seconds for 5 minutes):"

for i in {1..20}; do
  echo "=== Check $i/20 ($(date +%T)) ==="
  
  echo "HPA Status:"
  kubectl get hpa php-apache-hpa -n hpa-demo
  
  echo -e "\nPod Count:"
  kubectl get pods -n hpa-demo -l app=php-apache --no-headers | wc -l
  
  echo -e "\nPod Resource Usage:"
  kubectl top pods -n hpa-demo -l app=php-apache --no-headers 2>/dev/null || echo "Collecting metrics..."
  
  echo -e "\n---"
  sleep 15
done
```

Expected: You should see:
- CPU utilization increase above 50%
- Replica count gradually increase (up to 10)
- CPU utilization stabilize around 50% target

## 🔍 Step 8: View Detailed Scaling Events

```bash
# Check HPA events
echo "📋 HPA Scaling Events:"
kubectl describe hpa php-apache-hpa -n hpa-demo | grep -A 20 "Events:"

# Check deployment events
echo -e "\n📋 Deployment Events:"
kubectl get events -n hpa-demo --sort-by='.lastTimestamp' | grep -i "scaled\|hpa"

# Check current state
echo -e "\n📊 Current State:"
kubectl get deployment php-apache -n hpa-demo
kubectl get hpa php-apache-hpa -n hpa-demo
```

## 🛑 Step 9: Stop Load and Observe Scale-Down

```bash
# Stop the load generator
kubectl delete pod load-generator -n hpa-demo

echo "✅ Load generator stopped"
echo "⏳ Monitoring scale-down behavior (checking every 20 seconds for 5 minutes):"

for i in {1..15}; do
  echo "=== Check $i/15 ($(date +%T)) ==="
  
  kubectl get hpa php-apache-hpa -n hpa-demo
  echo "Pod count: $(kubectl get pods -n hpa-demo -l app=php-apache --no-headers | wc -l)"
  
  echo -e "\n---"
  sleep 20
done
```

Expected: After the cooldown period (60 seconds), pods will gradually scale down to minReplicas (1).

## 📊 Step 10: Create HPA with Memory Metrics

Let's create an HPA that scales based on both CPU and memory:

```bash
# Deploy a new application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-app
  namespace: hpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: memory-app
  template:
    metadata:
      labels:
        app: memory-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
EOF

echo "✅ Memory app deployed"

# Create HPA with both CPU and memory
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: memory-app-hpa
  namespace: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: memory-app
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
EOF

echo "✅ Multi-metric HPA created"
```

## 🔍 Step 11: Verify Multi-Metric HPA

```bash
# Wait for deployment
kubectl wait --for=condition=available deployment/memory-app -n hpa-demo --timeout=120s

# Check HPA status
kubectl get hpa memory-app-hpa -n hpa-demo

# Detailed view
kubectl describe hpa memory-app-hpa -n hpa-demo

# Check current metrics
echo -e "\n📊 Current Resource Usage:"
kubectl top pods -n hpa-demo -l app=memory-app
```

## 📈 Step 12: Create HPA with Custom Scaling Behavior

Let's create an HPA with more controlled scaling behavior:

```bash
# Deploy application
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: controlled-app
  namespace: hpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: controlled-app
  template:
    metadata:
      labels:
        app: controlled-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
EOF

# Create HPA with controlled behavior
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: controlled-app-hpa
  namespace: hpa-demo
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: controlled-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Pods
        value: 1
        periodSeconds: 30
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 120
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
      selectPolicy: Min
EOF

echo "✅ Controlled scaling HPA created"
```

### Controlled Behavior Explained

**Scale-Up**:
- Wait 30 seconds before scaling up
- Add maximum 1 pod every 30 seconds (gradual scale-up)

**Scale-Down**:
- Wait 120 seconds before scaling down
- Remove maximum 1 pod every 60 seconds (very gradual scale-down)

## 🔍 Step 13: Compare All HPAs

```bash
# List all HPAs
echo "📊 All HPAs:"
kubectl get hpa -n hpa-demo

# Compare their configurations
echo -e "\n📋 HPA Details:"
for hpa in $(kubectl get hpa -n hpa-demo -o jsonpath='{.items[*].metadata.name}'); do
  echo "=== $hpa ==="
  kubectl get hpa $hpa -n hpa-demo -o jsonpath='{.spec}' | jq
  echo ""
done
```

## 📊 Step 14: Test Advanced HPA Scenario

Let's test a realistic scenario with controlled load:

```bash
# Generate moderate load
kubectl run load-generator-2 \
  --image=busybox \
  --namespace=hpa-demo \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; sleep 0.5; done"

echo "✅ Moderate load generator started"
echo "⏳ Monitoring for 3 minutes:"

for i in {1..12}; do
  echo "=== Check $i/12 ($(date +%T)) ==="
  kubectl get hpa -n hpa-demo
  echo ""
  sleep 15
done

# Stop load
kubectl delete pod load-generator-2 -n hpa-demo --ignore-not-found=true
echo "✅ Load stopped"
```

## 📈 Step 15: View HPA Metrics History

```bash
# Get all HPA metrics
echo "📊 Current HPA Status:"
kubectl get hpa -n hpa-demo -o custom-columns=NAME:.metadata.name,TARGETS:.status.currentMetrics[0].resource.current.averageUtilization,MINPODS:.spec.minReplicas,MAXPODS:.spec.maxReplicas,REPLICAS:.status.currentReplicas

# Check deployment replica counts
echo -e "\n📊 Deployment Replicas:"
kubectl get deployments -n hpa-demo -o custom-columns=NAME:.metadata.name,DESIRED:.spec.replicas,CURRENT:.status.replicas,READY:.status.readyReplicas

# View all scaling events
echo -e "\n📋 Recent Scaling Events:"
kubectl get events -n hpa-demo --sort-by='.lastTimestamp' | grep -i "scaled"
```

## 🎓 Summary

You have successfully:
- ✅ Created HPA based on CPU utilization
- ✅ Generated load and observed automatic scale-up
- ✅ Observed scale-down behavior after load reduction
- ✅ Created multi-metric HPA (CPU and memory)
- ✅ Configured custom scaling behavior policies
- ✅ Monitored HPA metrics and events
- ✅ Compared different HPA configurations

## 🔑 Key Concepts Learned

1. **HPA Requirements**:
   - Resource requests must be defined
   - Metrics-server must be installed and working
   - Target metric must be measurable

2. **Scaling Behavior**:
   - Scale-up is typically faster than scale-down
   - Stabilization windows prevent flapping
   - Multiple policies can be combined

3. **Target Utilization**:
   - 50-70% is typical for CPU
   - Lower targets = more headroom, higher cost
   - Higher targets = better utilization, less headroom

4. **Cooldown Periods**:
   - Prevent rapid scaling oscillations
   - Scale-down cooldown typically longer than scale-up
   - Default: 3 minutes for scale-down, 0 for scale-up

## 📝 Best Practices

- ✅ Always define resource requests for HPA targets
- ✅ Set appropriate min/max replicas
- ✅ Use 50-70% CPU utilization as target
- ✅ Configure longer scale-down cooldown to prevent flapping
- ✅ Test HPA behavior under realistic load
- ✅ Monitor HPA metrics and adjust thresholds as needed
- ✅ Consider using multiple metrics (CPU + memory)
- ✅ Set maxReplicas to prevent runaway scaling
- ✅ Use controlled scaling behavior for critical workloads

## 🧹 Cleanup

```bash
# Delete load generators if still running
kubectl delete pod load-generator load-generator-2 -n hpa-demo --ignore-not-found=true

# Delete namespace
kubectl delete namespace hpa-demo

echo "✅ Cleanup complete"
```

## 🔍 Troubleshooting

**Issue**: HPA shows "unknown" for current metrics
- Verify metrics-server is running: `kubectl get pods -n kube-system -l k8s-app=metrics-server`
- Wait 1-2 minutes for metrics collection
- Check resource requests are defined on pods

**Issue**: HPA not scaling despite high CPU
- Verify target utilization is less than current utilization
- Check maxReplicas is not already reached
- Review HPA events: `kubectl describe hpa <name>`

**Issue**: Pods scaling up but still showing high CPU
- May need more time to stabilize
- Consider increasing maxReplicas
- Check if application is CPU-bound (optimize code)

**Issue**: HPA scaling too aggressively
- Increase stabilization windows
- Adjust scaling policies to be more conservative
- Consider using Pods policy instead of Percent

**Issue**: HPA not scaling down
- Check scaleDown stabilization window hasn't passed
- Verify current utilization is below target
- Check if minReplicas is reached

## 🚀 Next Steps

Proceed to:
- **[Chapter 4: KEDA - Event-Driven Autoscaling](../chapter-4-keda/README.md)**

In the next chapter, you'll learn about KEDA, which enables scaling based on event sources and custom metrics beyond just CPU and memory.
