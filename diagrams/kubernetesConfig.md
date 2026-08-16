# Kubernetes Deployment Configuration for Order Microservice

## Overview

This document defines the Kubernetes deployment configuration for the T-Mobile Order Microservice, a production-grade containerized application running on Oracle Container Engine for Kubernetes (OKE).

## Key Configuration Components

### 1. Deployment Metadata
- **Name**: order-service
- **Namespace**: production
- **Replicas**: 3 (for high availability)
- **Strategy**: RollingUpdate with zero downtime

### 2. Rolling Update Strategy
```yaml
maxSurge: 1          # Max new pods during upgrade
maxUnavailable: 0    # Ensures no downtime during updates
```

### 3. Container Specifications

#### Resource Management
- **Requests**: 250m CPU, 512Mi Memory
- **Limits**: 1000m CPU, 2Gi Memory
- These requests are critical for the Kubernetes scheduler and HPA

#### Health Probes
- **Startup Probe**: 30 retries × 10s = up to 5 minutes for app startup
- **Liveness Probe**: Restarts container if health check fails (10s interval, 3 failures)
- **Readiness Probe**: Removes pod from load balancer if not ready (5s interval, 2 failures)

#### Graceful Shutdown
- **terminationGracePeriodSeconds**: 60 seconds
- **preStop hook**: 10-second sleep to drain connections
- Allows in-flight requests to complete before termination

### 4. Environment Configuration

#### Non-Sensitive Config (ConfigMaps)
- Database connection details
- Oracle SCM endpoints
- Redis cluster address
- Spring Cloud Config URI

#### Sensitive Data (OCI Vault/Secrets)
- Database credentials
- OIDC/Authentication tokens
- Kafka bootstrap servers & credentials
- API keys and sensitive URLs

### 5. Pod Distribution & Anti-Affinity
```yaml
podAntiAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      labelSelector: app=order-service
      topologyKey: kubernetes.io/hostname
```
Ensures Order Service pods spread across different nodes for resilience.

### 6. Service Account & Security
- Workload Identity integration with OCI
- Docker registry authentication via secrets
- Prometheus metrics scraping at /actuator/prometheus

---

## Autoscaling Strategy & Challenges

### Autoscaling Architecture

The Order Service is designed to support multiple scaling mechanisms:

#### 1. **Horizontal Pod Autoscaler (HPA)**
Uses Kubernetes' native HPA based on CPU and memory metrics:
```
Min Replicas: 3
Max Replicas: 10
Target CPU: 70%
Target Memory: 80%
```

**How it works:**
- Prometheus collects metrics from `/actuator/prometheus`
- Metrics Server calculates average CPU/memory utilization
- HPA compares current vs. target utilization
- Scales up when threshold exceeded, scales down after cooldown

#### 2. **KEDA Scaling (Kafka-based)**
Event-driven autoscaling based on Kafka consumer lag:
```
Min Replicas: 3
Max Replicas: 15
Trigger: order.created topic lag
Scale Factor: 1 replica per 1000 messages lag
```

**Why KEDA is needed:**
- HPA based on CPU doesn't account for Kafka queue depth
- High message lag indicates falling behind on processing
- KEDA scales based on actual work queue size, not just resource usage

---

## Key Challenges in Autoscaling Dynamism

### 1. **Scheduling Delays**
**Challenge**: Pods take time to provision and become ready
- Pod startup time: ~30 seconds (health probe retries)
- Image pull time: ~10-15 seconds (cold start)
- Application initialization: ~15-20 seconds (Spring Boot)
- **Total cold-start latency**: 60+ seconds

**Impact**: By the time new pods are ready, traffic spike may have passed or caused timeouts

**Mitigation**:
- Maintain minimum 3 replicas (already running)
- Use preemptive scaling based on load prediction
- Implement request queuing with circuit breaker
- Cache application state to reduce startup time

### 2. **Scaling Decision Oscillation**
**Challenge**: Rapid scale-up/scale-down cycles waste resources
```
Scenario:
09:00 - CPU hits 75% → Scale UP (add 2 pods)
09:15 - Requests drop → CPU at 60%
09:30 - HPA scales DOWN (remove 1 pod)
09:45 - Traffic spikes again → Scale UP again
```

**Impact**: Constant pod churn increases costs and reduces stability

**Mitigation**:
- Set cooldown periods: 5 min scale-up, 10 min scale-down
- Use stable window: 2-3 min average before scaling decision
- Implement pod disruption budgets to prevent aggressive scale-down

### 3. **Resource Fragmentation**
**Challenge**: Uneven resource allocation across nodes
```
Scenario:
Node 1: 1200m CPU requested (of 2000m available)
Node 2: 800m CPU requested (of 2000m available)
Node 3: 1600m CPU requested (of 2000m available)

New pod with 250m CPU request → Cannot fit on Node 1 (200m free)
Must provision new node even though cluster has free capacity
```

**Impact**: Inefficient resource utilization, unnecessary node scaling

**Mitigation**:
- Implement Kubernetes descheduler for pod rebalancing
- Use Pod Disruption Budgets (PDB) to allow safe migrations
- Set up cluster autoscaler with consolidation

### 4. **Metrics Accuracy & Lag**
**Challenge**: Prometheus scrape interval and aggregation delay
```
Real-time CPU spike:     ↑↑↑
Prometheus scrape (15s): ↑
Metrics Server calc:     ↑
HPA evaluation (15s):    ↑
Pod provisioning:        Lagged response
```

**Impact**: Scaling decisions based on stale data (30-45 seconds old)

**Mitigation**:
- Use custom metrics via KEDA for more accurate scaling triggers
- Implement application-level load prediction
- Combine HPA with KEDA for multiple scaling signals

### 5. **Cost Implications of Dynamic Scaling**

#### Node Provisioning Costs
```
OCI VM Shape: Standard.E4.Flex (2 OCPUs, 16GB RAM)
Cost: ~$0.30/hour per node

Scenario 1 - Aggressive Scaling:
10:00 - 5 nodes running
10:30 - Scale to 12 nodes (+7 nodes @ +$2.10/hour)
11:00 - Scale back to 5 nodes (-7 nodes @ -$2.10/hour)
Daily cost swing: $50.40 for 24-hour churn

Scenario 2 - Stable Scaling:
Maintain 6-8 nodes based on average load
Consistent cost: predictable budget
```

#### Reserved Instance Strategy
- **Approach 1**: Buy reserved capacity for base load (3-5 nodes)
  - Cost: ~$0.12/hour (60% discount)
  - Flexibility: Lost

- **Approach 2**: Mix reserved + on-demand
  - Base: 3 nodes reserved @ $0.12/hour = $86.40/day
  - Peaks: Scale on-demand @ $0.30/hour for bursts
  - Optimal for traffic with predictable baseline + occasional spikes

#### Pod-Level Costs
```
Per Pod Monthly Cost Estimate:
250m CPU request: ~$0.015/hour = $10.80/month
512Mi Memory: ~$0.002/hour = $1.44/month
Total per pod: ~$12.24/month

At 10 pods: $122.40/month
At 15 pods (peak): $183.60/month
At 3 pods (minimum): $36.72/month
```

### 6. **Latency vs. Cost Tradeoff**

| Strategy | Response Time | Monthly Cost | Burst Handling |
|----------|---------------|-------------|-----------------|
| Conservative (max 5 replicas) | 500-2000ms | $60 | Poor |
| Balanced (max 10 replicas) | 100-500ms | $122 | Good |
| Aggressive (max 15 replicas) | 50-200ms | $183 | Excellent |
| Predictive (KEDA + HPA) | 50-300ms | $100 | Very Good |

### 7. **Stateful Service Challenges**
**Challenge**: Order Service may cache state in memory
```
Pod A: Processes order #123, caches customer profile
User makes follow-up request → Load balancer routes to Pod B
Pod B doesn't have cached data → Cache miss, higher latency
```

**Mitigation**:
- Use distributed cache (Redis) - already configured
- Implement session affinity (sticky sessions) carefully
- Ensure stateless pod design to enable safe scaling

---

## Recommended Autoscaling Configuration

### Hybrid Approach: HPA + KEDA
```yaml
# HPA for resource-based scaling
HPA:
  minReplicas: 3
  maxReplicas: 10
  targetCPU: 70%
  targetMemory: 75%
  behavior:
    scaleUp:
      stabilizationWindow: 0s
      policies:
      - type: Percent
        value: 100
        period: 15s
    scaleDown:
      stabilizationWindow: 300s
      policies:
      - type: Pods
        value: 1
        period: 60s

# KEDA for Kafka lag-based scaling
KEDA:
  minReplicas: 3
  maxReplicas: 15
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka-cluster:9092
      consumerGroup: order-service-consumer
      topic: order.created
      lagThreshold: "1000"
      offsetResetPolicy: latest
```

### Cost Optimization
1. **Base Load**: 3 nodes (reserved instances)
2. **Expected Load**: 5-8 nodes (on-demand)
3. **Peak Load**: 10-12 nodes (on-demand + Spot instances)
4. **Monitoring**: Dashboard showing cost per scaling event

---

## References
- [Kubernetes Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [KEDA Scaler Reference](https://keda.sh/docs/2.12/scalers/apache-kafka/)
- [OKE Cluster Autoscaling](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengusingclusterautoscaler.htm)
- [Kubernetes Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
