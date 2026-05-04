# AKS Resource Management & Autoscaling - Hands-on Lab

Welcome to the Azure Kubernetes Service (AKS) Resource Management and Autoscaling Lab! This comprehensive lab will guide you through essential techniques for managing resources, implementing autoscaling, and ensuring application resilience in Kubernetes.

## 🎯 Learning Objectives

By completing this lab, you will gain hands-on experience with:

- **Resource Management**: Configure resource requests and limits for optimal pod scheduling and resource utilization
- **Resource Governance**: Implement LimitRange and ResourceQuota to enforce resource policies at namespace level
- **Horizontal Autoscaling**: Configure Horizontal Pod Autoscaler (HPA) to automatically scale applications based on metrics
- **Event-Driven Autoscaling**: Deploy KEDA to scale applications based on external events and custom metrics
- **Application Resilience**: Implement Pod Disruption Budgets (PDB) to maintain availability during cluster maintenance

## 📚 Lab Structure

This lab is organized into the following chapters:

- **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)**
  - Setting your student initials
  - Creating resource group and AKS cluster
  - Installing required tools and add-ons

- **[Chapter 1: Resource Requests and Limits](./chapter-1-requests-limits/README.md)**
  - Understanding resource requests vs limits
  - Configuring CPU and memory resources
  - Observing QoS classes (Guaranteed, Burstable, BestEffort)
  - Testing resource enforcement and OOM scenarios

- **[Chapter 2: LimitRange and ResourceQuota](./chapter-2-limitrange-quota/README.md)**
  - Implementing default resource limits with LimitRange
  - Setting namespace-level constraints with ResourceQuota
  - Testing quota enforcement and resource tracking
  - Managing multi-tenant cluster resources

- **[Chapter 3: Horizontal Pod Autoscaler (HPA)](./chapter-3-hpa/README.md)**
  - Deploying metrics-server for resource metrics
  - Configuring HPA based on CPU and memory
  - Load testing and observing scale-up/scale-down behavior
  - Understanding HPA cooldown periods and thresholds

- **[Chapter 4: KEDA - Event-Driven Autoscaling](./chapter-4-keda/README.md)**
  - Installing KEDA in AKS
  - Scaling based on Azure Service Bus queue length
  - Scaling based on HTTP requests (KEDA HTTP Add-on)
  - Configuring scale-to-zero scenarios
  - Comparing KEDA with standard HPA

- **[Chapter 5: Pod Disruption Budgets](./chapter-5-pdb/README.md)**
  - Understanding voluntary vs involuntary disruptions
  - Configuring minAvailable and maxUnavailable constraints
  - Testing PDB during node drains
  - Balancing availability with cluster maintenance
  - PDB best practices for different workload types

## ⏱️ Estimated Time

- **Total Lab Duration**: 2-3 hours
- **Chapter 0 (Setup)**: 15-20 minutes
- **Chapter 1 (Requests & Limits)**: 20-25 minutes
- **Chapter 2 (LimitRange & Quota)**: 20-25 minutes
- **Chapter 3 (HPA)**: 25-30 minutes
- **Chapter 4 (KEDA)**: 30-35 minutes
- **Chapter 5 (PDB)**: 15-20 minutes

## 🔧 Prerequisites

- Azure subscription with appropriate permissions
- Basic knowledge of Kubernetes concepts (pods, deployments, services)
- Familiarity with Azure CLI and kubectl
- Understanding of basic resource concepts (CPU, memory)
- Text editor (VS Code recommended)

## 🚀 Getting Started

1. Start with **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)**
2. Complete each chapter in sequence
3. Keep your AKS cluster running throughout the lab
4. Each chapter builds on previous concepts

## 💡 Tips for Success

- **Set your initials**: You'll use student initials for resource naming to avoid conflicts
- **One cluster for all**: Use the same AKS cluster throughout all chapters
- **Monitor resources**: Use `kubectl top nodes` and `kubectl top pods` to observe resource usage
- **Load generation**: Several chapters include load generation - ensure load generators complete
- **Save your work**: Keep track of resource names and configurations
- **Ask for help**: Don't hesitate to ask your instructor if you get stuck

## 📝 Notes

- All students in the same subscription should use unique initials for naming
- Some resources may take several minutes to provision
- HPA and KEDA may take 1-2 minutes to react to load changes
- Keep your Azure CLI session active throughout the lab
- Save important commands and outputs for reference

## 🧹 Cleanup

After completing all chapters, refer to the cleanup section in Chapter 0 to remove all resources and avoid unnecessary charges.

---

**Ready to begin?** Head over to **[Chapter 0: Setup and Prerequisites](./chapter-0-setup/README.md)** to get started!
