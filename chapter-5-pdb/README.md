# Chapter 5: Pod Disruption Budgets (PDB)

In this chapter, you'll learn how to maintain application availability during voluntary disruptions using Pod Disruption Budgets (PDB).

## 🎯 Learning Objectives

- Understand voluntary vs involuntary disruptions
- Configure Pod Disruption Budgets with minAvailable and maxUnavailable
- Test PDB during node drains and upgrades
- Balance availability requirements with cluster maintenance
- Apply PDB best practices for different workload types
- Understand PDB limitations and edge cases

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- AKS cluster with at least 3 nodes
- kubectl configured to access the cluster

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./lab-config.sh

# Verify cluster access
kubectl get nodes
```

## 📖 Understanding Pod Disruption Budgets

### What Are Disruptions?

**Voluntary Disruptions** (PDB applies):
- Node drain for maintenance
- Cluster upgrades
- Manual pod deletions
- Deployment updates with kubectl

**Involuntary Disruptions** (PDB does NOT apply):
- Hardware failures
- Kernel panics
- Node crashes
- Out-of-memory kills

### PDB Configuration Options

**minAvailable**:
- Minimum number of pods that must be available
- Can be absolute number (e.g., 3) or percentage (e.g., 80%)
- Example: `minAvailable: 2` = At least 2 pods must always be running

**maxUnavailable**:
- Maximum number of pods that can be unavailable
- Can be absolute number (e.g., 1) or percentage (e.g., 20%)
- Example: `maxUnavailable: 1` = At most 1 pod can be down at a time

**Note**: You must specify either `minAvailable` OR `maxUnavailable`, not both.

## 🏗️ Step 1: Create a Namespace

```bash
# Create namespace for this chapter
kubectl create namespace pdb-demo

echo "✅ Namespace created"
```

## 📦 Step 2: Deploy an Application Without PDB

First, let's see what happens without a PDB:

```bash
# Deploy application with 5 replicas spread across nodes
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-without-pdb
  namespace: pdb-demo
spec:
  replicas: 5
  selector:
    matchLabels:
      app: app-without-pdb
  template:
    metadata:
      labels:
        app: app-without-pdb
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - app-without-pdb
              topologyKey: kubernetes.io/hostname
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
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: app-without-pdb
  namespace: pdb-demo
spec:
  selector:
    app: app-without-pdb
  ports:
  - port: 80
    targetPort: 80
EOF

echo "✅ Application without PDB deployed"
```

## ✅ Step 3: Verify Pod Distribution

```bash
# Wait for deployment to be ready
kubectl wait --for=condition=available deployment/app-without-pdb -n pdb-demo --timeout=120s

# Check pod distribution across nodes
echo "📊 Pod Distribution:"
kubectl get pods -n pdb-demo -l app=app-without-pdb -o wide

# Show pod count per node
echo -e "\n📊 Pods Per Node:"
kubectl get pods -n pdb-demo -l app=app-without-pdb -o wide --no-headers | awk '{print $7}' | sort | uniq -c
```

## 🧪 Step 4: Test Node Drain Without PDB

```bash
# Pick the first node
TARGET_NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
echo "Target node for drain: $TARGET_NODE"

# Check how many pods are on this node
echo -e "\n📊 Pods on target node before drain:"
kubectl get pods -n pdb-demo -l app=app-without-pdb -o wide | grep $TARGET_NODE || echo "No pods on this node"

# Drain the node (this will evict all pods)
echo -e "\n⏳ Draining node $TARGET_NODE..."
kubectl drain $TARGET_NODE --ignore-daemonsets --delete-emptydir-data --force --timeout=60s

echo -e "\n✅ Node drained"

# Check pod distribution after drain
echo -e "\n📊 Pods after drain:"
kubectl get pods -n pdb-demo -l app=app-without-pdb -o wide

# Uncordon the node
kubectl uncordon $TARGET_NODE
echo "✅ Node uncordoned"
```

Expected: All pods are evicted immediately from the node without any constraints.

## 🛡️ Step 5: Deploy Application With PDB (minAvailable)

Now let's deploy an application with a PDB:

```bash
# Deploy application with PDB
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-pdb
  namespace: pdb-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-with-pdb
  template:
    metadata:
      labels:
        app: app-with-pdb
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - app-with-pdb
            topologyKey: kubernetes.io/hostname
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
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-with-pdb
  namespace: pdb-demo
spec:
  minAvailable: 2  # At least 2 pods must be available
  selector:
    matchLabels:
      app: app-with-pdb
---
apiVersion: v1
kind: Service
metadata:
  name: app-with-pdb
  namespace: pdb-demo
spec:
  selector:
    app: app-with-pdb
  ports:
  - port: 80
    targetPort: 80
EOF

echo "✅ Application with PDB deployed"
```

## 🔍 Step 6: Verify PDB Status

```bash
# Wait for deployment
kubectl wait --for=condition=available deployment/app-with-pdb -n pdb-demo --timeout=120s

# Check PDB status
echo "📊 PDB Status:"
kubectl get pdb -n pdb-demo

# Detailed PDB information
echo -e "\n📋 PDB Details:"
kubectl describe pdb app-with-pdb -n pdb-demo

# Check pod distribution
echo -e "\n📊 Pod Distribution:"
kubectl get pods -n pdb-demo -l app=app-with-pdb -o wide
```

Expected: PDB should show:
- **ALLOWED DISRUPTIONS**: Number of pods that can be disrupted (replicas - minAvailable)
- **MIN AVAILABLE**: 2

With 3 replicas and minAvailable: 2, the PDB allows 1 pod to be disrupted at a time (3 - 2 = 1).

## 🧪 Step 7: Test Node Drain With PDB

```bash
# Pick a node with pods from our app
TARGET_NODE=$(kubectl get pods -n pdb-demo -l app=app-with-pdb -o wide --no-headers | awk '{print $7}' | head -1)
echo "Target node for drain: $TARGET_NODE"

# Check how many pods are on this node
PODS_ON_NODE=$(kubectl get pods -n pdb-demo -l app=app-with-pdb -o wide --no-headers | grep $TARGET_NODE | wc -l)
echo "Pods on target node: $PODS_ON_NODE"

# Check PDB status before drain
echo -e "\n📊 PDB Status Before Drain:"
kubectl get pdb app-with-pdb -n pdb-demo

# Drain the node (PDB will protect availability)
echo -e "\n⏳ Draining node $TARGET_NODE (this will respect PDB)..."
kubectl drain $TARGET_NODE --ignore-daemonsets --delete-emptydir-data --timeout=120s &

DRAIN_PID=$!

# Monitor PDB and pods during drain
echo -e "\n📊 Monitoring drain (checking every 5 seconds):"
for i in {1..20}; do
  echo "=== Check $i/20 ($(date +%T)) ==="
  
  # Check if drain is still running
  if ! ps -p $DRAIN_PID > /dev/null 2>&1; then
    echo "Drain completed"
    break
  fi
  
  # Check PDB status
  kubectl get pdb app-with-pdb -n pdb-demo --no-headers 2>/dev/null || echo "PDB status unavailable"
  
  # Check pod count
  READY_PODS=$(kubectl get pods -n pdb-demo -l app=app-with-pdb --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)
  echo "Ready pods: $READY_PODS"
  
  echo -e "\n---"
  sleep 5
done

# Wait for drain to complete
wait $DRAIN_PID 2>/dev/null

echo -e "\n✅ Drain completed"

# Check final state
echo -e "\n📊 Final State:"
kubectl get pdb app-with-pdb -n pdb-demo
kubectl get pods -n pdb-demo -l app=app-with-pdb -o wide

# Uncordon the node
kubectl uncordon $TARGET_NODE
echo "✅ Node uncordoned"
```

Expected: 
- Drain respects PDB and maintains at least 2 pods running
- Pods are evicted gradually, not all at once
- At least 2 pods remain available throughout the drain

**Important Notes:**
- With required anti-affinity, each pod runs on a different node (1 pod per node)
- The PDB allows disrupting **1 pod** at a time (3 replicas - 2 minAvailable = 1)
- When draining a node with 1 pod, the drain will:
  1. Evict the pod from the target node
  2. Wait for it to be rescheduled and become ready on another node
  3. Once ready, eviction succeeds and drain completes
- For successful draining with PDB:
  - Ensure pods are spread across multiple nodes (use `podAntiAffinity`)
  - Have enough cluster capacity for pod rescheduling
  - Allow sufficient timeout for gradual eviction (default: 120s)
- If the cluster is at capacity, the drain may block waiting for resources

## 📊 Step 8: Deploy Application With maxUnavailable

Let's create a PDB using `maxUnavailable` instead:

```bash
# Deploy application with maxUnavailable PDB
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-maxunavailable
  namespace: pdb-demo
spec:
  replicas: 6
  selector:
    matchLabels:
      app: app-maxunavailable
  template:
    metadata:
      labels:
        app: app-maxunavailable
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-maxunavailable-pdb
  namespace: pdb-demo
spec:
  maxUnavailable: 2  # At most 2 pods can be unavailable
  selector:
    matchLabels:
      app: app-maxunavailable
EOF

echo "✅ Application with maxUnavailable PDB deployed"
```

## 🔍 Step 9: Compare minAvailable vs maxUnavailable

```bash
# Wait for deployment
kubectl wait --for=condition=available deployment/app-maxunavailable -n pdb-demo --timeout=120s

# Compare both PDBs
echo "📊 All PDBs:"
kubectl get pdb -n pdb-demo -o custom-columns=NAME:.metadata.name,MIN-AVAILABLE:.spec.minAvailable,MAX-UNAVAILABLE:.spec.maxUnavailable,ALLOWED-DISRUPTIONS:.status.disruptionsAllowed,CURRENT:.status.currentHealthy,DESIRED:.status.desiredHealthy

# Explanation
echo -e "\n📋 Explanation:"
echo "app-with-pdb (minAvailable: 3):"
echo "  - 5 replicas, minimum 3 must be available"
echo "  - Allowed disruptions: 5 - 3 = 2"
echo ""
echo "app-maxunavailable-pdb (maxUnavailable: 2):"
echo "  - 6 replicas, maximum 2 can be unavailable"
echo "  - Allowed disruptions: 2"
```

## 🎯 Step 10: Deploy Critical Application with Percentage-Based PDB

```bash
# Deploy critical application with percentage-based PDB
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
  namespace: pdb-demo
spec:
  replicas: 10
  selector:
    matchLabels:
      app: critical-app
      tier: critical
  template:
    metadata:
      labels:
        app: critical-app
        tier: critical
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
  namespace: pdb-demo
spec:
  minAvailable: 80%  # At least 80% of pods must be available
  selector:
    matchLabels:
      app: critical-app
      tier: critical
EOF

echo "✅ Critical application with percentage-based PDB deployed"
```

## 🔍 Step 11: Verify Percentage-Based PDB

```bash
# Wait for deployment
kubectl wait --for=condition=available deployment/critical-app -n pdb-demo --timeout=120s

# Check PDB status
echo "📊 Critical App PDB:"
kubectl get pdb critical-app-pdb -n pdb-demo

# Detailed view
kubectl describe pdb critical-app-pdb -n pdb-demo

# Calculate actual numbers
echo -e "\n📊 Calculation:"
echo "Total replicas: 10"
echo "minAvailable: 80% = 8 pods"
echo "Allowed disruptions: 10 - 8 = 2 pods"
```

## ⚠️ Step 12: Test Edge Case - PDB Blocking Drain

Let's see what happens when PDB prevents drain:

```bash
# Deploy application where all pods are on one node (if possible)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: single-node-app
  namespace: pdb-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: single-node-app
  template:
    metadata:
      labels:
        app: single-node-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - single-node-app
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: single-node-app-pdb
  namespace: pdb-demo
spec:
  minAvailable: 3  # All pods must be available
  selector:
    matchLabels:
      app: single-node-app
EOF

echo "✅ Single-node application deployed"
```

## 🧪 Step 13: Observe PDB Behavior with Strict Requirements

```bash
# Wait for deployment
sleep 20

# Check pod distribution
echo "📊 Pod Distribution (should be on different nodes):"
kubectl get pods -n pdb-demo -l app=single-node-app -o wide

# Try to drain a node with one of these pods
TARGET_NODE=$(kubectl get pods -n pdb-demo -l app=single-node-app -o wide --no-headers | awk '{print $7}' | head -1)
echo -e "\nTarget node: $TARGET_NODE"

# This drain should succeed slowly because pods can move to other nodes
echo -e "\n⏳ Attempting drain (this should succeed gradually)..."
timeout 60 kubectl drain $TARGET_NODE --ignore-daemonsets --delete-emptydir-data --pod-selector=app=single-node-app 2>&1 || echo "Drain timed out or completed"

# Check status
echo -e "\n📊 Status After Drain Attempt:"
kubectl get pods -n pdb-demo -l app=single-node-app -o wide
kubectl get pdb single-node-app-pdb -n pdb-demo

# Uncordon
kubectl uncordon $TARGET_NODE
```

## 📊 Step 14: View All PDBs and Their Status

```bash
# List all PDBs
echo "📊 All Pod Disruption Budgets:"
kubectl get pdb -n pdb-demo

# Detailed comparison
echo -e "\n📋 Detailed PDB Comparison:"
kubectl get pdb -n pdb-demo -o json | jq -r '.items[] | {
  name: .metadata.name,
  minAvailable: .spec.minAvailable,
  maxUnavailable: .spec.maxUnavailable,
  currentHealthy: .status.currentHealthy,
  desiredHealthy: .status.desiredHealthy,
  disruptionsAllowed: .status.disruptionsAllowed
}'

# Check all deployments
echo -e "\n📊 All Deployments:"
kubectl get deployments -n pdb-demo
```

## 📊 Step 15: Test Multiple PDBs

```bash
# Deploy StatefulSet with PDB
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-app
  namespace: pdb-demo
spec:
  serviceName: stateful-app
  replicas: 5
  selector:
    matchLabels:
      app: stateful-app
  template:
    metadata:
      labels:
        app: stateful-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        resources:
          requests:
            cpu: "50m"
            memory: "32Mi"
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: stateful-app-pdb
  namespace: pdb-demo
spec:
  maxUnavailable: 1  # Only 1 pod can be down at a time
  selector:
    matchLabels:
      app: stateful-app
---
apiVersion: v1
kind: Service
metadata:
  name: stateful-app
  namespace: pdb-demo
spec:
  selector:
    app: stateful-app
  ports:
  - port: 80
  clusterIP: None  # Headless service
EOF

echo "✅ StatefulSet with PDB deployed"

# Wait and check
sleep 20
kubectl get statefulset stateful-app -n pdb-demo
kubectl get pdb stateful-app-pdb -n pdb-demo
```

## 🎓 Summary

You have successfully:
- ✅ Understood voluntary vs involuntary disruptions
- ✅ Created PDBs with minAvailable and maxUnavailable
- ✅ Tested PDB enforcement during node drains
- ✅ Observed the difference between with and without PDB
- ✅ Used percentage-based PDB configurations
- ✅ Tested PDB behavior with rolling updates
- ✅ Applied PDBs to different workload types (Deployment, StatefulSet)

## 🔑 Key Concepts Learned

1. **PDB Purpose**:
   - Maintains application availability during voluntary disruptions
   - Does NOT protect against involuntary disruptions (hardware failures)
   - Enforced by eviction API

2. **minAvailable vs maxUnavailable**:
   - `minAvailable`: Minimum pods that must be running
   - `maxUnavailable`: Maximum pods that can be down
   - Use only ONE, not both
   - Can be absolute number or percentage

3. **PDB Behavior**:
   - Blocks drains if it would violate availability
   - Respects PDB during rolling updates
   - Works with eviction API (not direct pod deletion)

4. **Common Patterns**:
   - High availability apps: `minAvailable: "80%"`
   - Stateful apps: `maxUnavailable: 1` (one-at-a-time updates)
   - Critical services: `minAvailable: <high number>`

## 📝 Best Practices

- ✅ Always create PDBs for production applications
- ✅ Use percentage-based PDBs for applications that scale
- ✅ Set `minAvailable` based on your SLA requirements
- ✅ For StatefulSets, use `maxUnavailable: 1` for ordered updates
- ✅ Test PDB behavior during drains and upgrades
- ✅ Don't set PDB too restrictive (can block maintenance)
- ✅ Combine with anti-affinity to spread pods across nodes
- ✅ Monitor PDB status: `kubectl get pdb`
- ✅ Use `maxUnavailable` for better node drain flexibility
- ⚠️ Avoid PDB for single-replica deployments (no effect)

## 🧹 Cleanup

```bash
# Delete namespace
kubectl delete namespace pdb-demo

echo "✅ Cleanup complete"
```

## 🔍 Troubleshooting

**Issue**: Node drain stuck or timing out
- Check PDB status: `kubectl get pdb -A`
- Verify enough healthy pods exist to satisfy PDB
- Check if pods can be rescheduled to other nodes
- Consider temporarily adjusting PDB (not recommended in production)

**Issue**: PDB shows 0 allowed disruptions
- Current healthy pods = minAvailable
- Cannot drain any nodes without violating PDB
- Scale up application or adjust PDB

**Issue**: Pod deleted despite PDB
- PDB only protects against evictions (drain, kubectl drain)
- Direct pod deletion (`kubectl delete pod`) bypasses PDB
- Hardware failures are involuntary disruptions (not protected)

**Issue**: PDB not preventing disruptions
- Verify selector matches pod labels exactly
- Check PDB is in the same namespace as pods
- Ensure eviction API is being used (not direct deletion)

**Issue**: Rolling update stuck
- PDB might be too restrictive for the update strategy
- Check deployment strategy (RollingUpdate settings)
- Verify enough replicas to satisfy PDB during update

## 🎉 Lab Complete!

Congratulations! You have completed all chapters of the AKS Resource Management & Autoscaling Lab.

You've learned:
- ✅ Resource requests and limits (Chapter 1)
- ✅ LimitRange and ResourceQuota (Chapter 2)
- ✅ Horizontal Pod Autoscaler (Chapter 3)
- ✅ KEDA event-driven autoscaling (Chapter 4)
- ✅ Pod Disruption Budgets (Chapter 5)

## 🧹 Final Cleanup

To remove all resources created during this lab:

```bash
# Load configuration
source ./lab-config.sh

# Delete the resource group (removes everything)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

echo "🗑️ Resource group deletion initiated"
echo "✅ Lab cleanup complete!"
```

---

**Thank you for completing this lab!** 

For questions or feedback, please contact your instructor.
