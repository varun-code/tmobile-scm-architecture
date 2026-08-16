# Kubernetes Deployment Configuration - Order Service

## 6.1 Deployment Manifest

### Order Microservice Deployment Configuration

```yaml
# Order Microservice Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
    tier: microservices
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
  selector:
    matchLabels:
      app: order-service
  
  template:
    metadata:
      labels:
        app: order-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/order-service/actuator/prometheus"
    
    spec:
      serviceAccountName: order-service
      imagePullSecrets:
      - name: docker-registry-secret
      terminationGracePeriodSeconds: 60
      
      containers:
      - name: order-service
        image: tmobile/order-service:1.0.0
        imagePullPolicy: IfNotPresent
        
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: SPRING_CLOUD_CONFIG_URI
          value: "http://config-server:8888"
        - name: ORDER_DB_HOST
          valueFrom:
            configMapKeyRef:
              name: order-service-config
              key: db-host
        - name: ORDER_DB_PORT
          valueFrom:
            configMapKeyRef:
              name: order-service-config
              key: db-port
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: order-db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: order-db-credentials
              key: password
        - name: ORACLE_SCM_BASE_URL
          valueFrom:
            configMapKeyRef:
              name: oracle-scm-config
              key: base-url
        - name: OIDC_ISSUER_URI
          valueFrom:
            secretKeyRef:
              name: oauth2-secret
              key: issuer-uri
        - name: KAFKA_BOOTSTRAP_SERVERS
          valueFrom:
            secretKeyRef:
              name: kafka-credentials
              key: bootstrap-servers
        - name: KAFKA_USERNAME
          valueFrom:
            secretKeyRef:
              name: kafka-credentials
              key: username
        - name: KAFKA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: kafka-credentials
              key: password
        - name: REDIS_HOST
          value: "redis-cluster"
        - name: LOG_LEVEL
          value: "INFO"
        
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 2Gi
        
        startupProbe:
          httpGet:
            path: /order-service/actuator/health
            port: http
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30
        
        livenessProbe:
          httpGet:
            path: /order-service/actuator/health/live
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /order-service/actuator/health/ready
            port: http
          initialDelaySeconds: 20
          periodSeconds: 5
          timeoutSeconds: 5
          failureThreshold: 2
        
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - "sleep 10"
        
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
      
      volumes:
      - name: config-volume
        configMap:
          name: order-service-config
      
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
                  - order-service
              topologyKey: kubernetes.io/hostname

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: production
  labels:
    app: order-service
spec:
  type: ClusterIP
  selector:
    app: order-service
  ports:
  - name: http
    port: 8080
    targetPort: http
    protocol: TCP

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 15
  
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
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Percent
        value: 50
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 25
        periodSeconds: 60

---
# PodDisruptionBudget (for graceful shutdown during node maintenance)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service
```

---

## 6.2 Configuration Specifications

### Deployment Specs
| Component | Configuration |
|-----------|----------------|
| **Replicas** | 3 (minimum), scales to 15 (maximum) |
| **Strategy** | RollingUpdate (maxSurge: 1, maxUnavailable: 0) |
| **Namespace** | production |
| **Version** | v1 |

### Container Resources
| Resource | Request | Limit |
|----------|---------|-------|
| **CPU** | 250m | 1000m (4x) |
| **Memory** | 512Mi | 2Gi (4x) |
| **Port** | 8080 | HTTP |

### Health Probes
| Probe | Path | Interval | Timeout | Failures |
|-------|------|----------|---------|----------|
| **Startup** | `/order-service/actuator/health` | 10s | 5s | 30 retries = 5min |
| **Liveness** | `/order-service/actuator/health/live` | 10s | 5s | 3 failures → restart |
| **Readiness** | `/order-service/actuator/health/ready` | 5s | 5s | 2 failures → exclude |

### Environment Configuration

**From ConfigMaps:**
- `ORDER_DB_HOST`: Database hostname
- `ORDER_DB_PORT`: Database port
- `ORACLE_SCM_BASE_URL`: SCM endpoint URL
- `SPRING_CLOUD_CONFIG_URI`: Configuration server

**From Secrets (OCI Vault):**
- `DB_USERNAME` / `DB_PASSWORD`: Database credentials
- `OIDC_ISSUER_URI`: Authentication provider
- `KAFKA_BOOTSTRAP_SERVERS`, `KAFKA_USERNAME`, `KAFKA_PASSWORD`: Kafka credentials

**Hardcoded:**
- `REDIS_HOST`: "redis-cluster"
- `LOG_LEVEL`: "INFO"
- `SPRING_PROFILES_ACTIVE`: "production"

### Autoscaling Strategy (HPA)

**Scaling Triggers:**
```yaml
metrics:
  - CPU Utilization: 70%
  - Memory Utilization: 80%
  - HTTP Requests/sec: 1000 per pod average
```

**Scale-Up Behavior:**
- Stabilization: 30 seconds
- Policy 1: 100% increase (double replicas) every 15s
- Policy 2: +4 pods every 15s
- Selects maximum policy (scales faster)

**Scale-Down Behavior:**
- Stabilization: 300 seconds (5 minutes)
- Policy: 50% reduction every 60s
- Slower scale-down prevents flapping

### Pod Disruption Budget (PDB)
- **minAvailable**: 2 pods
- Ensures minimum 2 pods remain running during:
  - Node maintenance
  - Cluster updates
  - Voluntary pod evictions

---

## 6.3 Autoscaling Challenges & Solutions

### Challenge 1: Cold Start Latency
**Problem**: Startup probe allows 30 retries × 10s = 5 minutes before pod is ready

**Impact**: 
- During traffic spike, new pods take too long to become ready
- Existing pods receive sudden load increase
- Request timeouts and failures

**Solution**:
- Keep 3 minimum replicas always running (warm)
- Implement predictive scaling based on historical patterns
- Use pod preemption to prepare standby pods

---

### Challenge 2: Scaling Oscillation
**Problem**: Rapid scale-up/scale-down cycles waste resources

```
Scenario:
10:00 - CPU hits 70% → Scale UP (add 4 pods)
10:15 - Requests drop → CPU at 60%
10:20 - HPA scales DOWN (remove 2 pods) → stabilization kicks in
10:25 - Traffic spikes again → Scale UP again
```

**Cost Impact**: $0.30/hour per node × 4 churn cycles = $0.60/hour wasted

**Solution**:
- 5-minute scale-down stabilization window prevents rapid fluctuations
- 30-second scale-up window allows quick response to spikes
- Monitor aggregate metrics over 2-3 minute windows

---

### Challenge 3: Resource Fragmentation
**Problem**: Uneven pod distribution causes node underutilization

```
Node 1: 1200m CPU used (of 2000m) → 200m free
Node 2: 800m CPU used (of 2000m) → 1200m free
Node 3: 1600m CPU used (of 2000m) → 400m free

New 250m CPU pod can't fit on Node 1 (only 200m free)
Kubernetes provisions new expensive node instead of rebalancing
```

**Solution**:
- Pod anti-affinity spreads across nodes (preferred, not required)
- Descheduler rebalances pods during maintenance windows
- Cluster autoscaler consolidates underutilized nodes

---

### Challenge 4: Kafka Lag Not Captured by CPU/Memory
**Problem**: HPA only watches CPU and memory, not message queue depth

```
Scenario:
- CPU is 40% (below 70% threshold)
- Memory is 60% (below 80% threshold)
- HPA doesn't scale up
- But Kafka lag is 50,000 messages behind!
- Orders processing falls further behind
```

**Solution**:
- Implement KEDA scaler for Kafka consumer lag
- Combine HPA + KEDA for comprehensive autoscaling
- Scale based on actual work queue size, not just resources

---

### Challenge 5: Cost Optimization vs. Latency

| Strategy | Min Replicas | Max Replicas | Latency | Monthly Cost |
|----------|--------------|--------------|---------|-------------|
| Conservative | 2 | 5 | 500-2000ms | $40 |
| Balanced | 3 | 15 | 100-500ms | $100-120 |
| Aggressive | 4 | 20 | 50-200ms | $150-180 |
| **Optimized** | **3 (Reserved)** | **15 (On-Demand)** | **100-400ms** | **$85-100** |

**Optimized Approach:**
- 3 reserved instances (60% discount) = Base cost
- Scale on-demand 4-15 replicas for bursts
- Use Spot instances for non-critical peaks (70% cheaper)
- Result: 20-30% cost savings vs. aggressive scaling

---

### Challenge 6: Graceful Termination Window
**Problem**: Pods killed abruptly during scale-down cause in-flight request failures

```
Scenario:
- Client sends order request to Pod A
- HPA triggers scale-down, Pod A gets SIGTERM
- Pod A immediately closes connections
- Client request fails even though it was in-flight
```

**Solution:**
- `terminationGracePeriodSeconds: 60` - gives 60s to drain
- `preStop` hook sleeps 10s - allows load balancer to remove pod
- Pods complete existing requests before shutdown
- New requests routed to other pods

---

## 6.4 Recommended Configuration

### For Production Order Service:

**Pod Replicas:**
```yaml
minReplicas: 3        # Always running
maxReplicas: 15       # Peak burst capacity
```

**Resource Allocation:**
```yaml
requests:
  cpu: 250m           # Guaranteed baseline
  memory: 512Mi
limits:
  cpu: 1000m          # Allow 4x burst
  memory: 2Gi
```

**Scaling Triggers (Combined):**
```yaml
# HPA: Resource-based scaling
- CPU: 70%
- Memory: 80%
- Requests/sec: 1000 per pod

# KEDA: Event-based scaling (Kafka lag)
- Kafka lag: 1 pod per 1000 messages behind
```

**Health Check Configuration:**
```yaml
Startup:   30 retries = 5 min max startup time
Liveness:  Restart pod if unhealthy (30s check)
Readiness: Remove from LB if not ready (5s check)
```

---

## 6.5 Deployment Instructions

1. **Create Secrets:**
```bash
kubectl create secret generic order-db-credentials \
  --from-literal=username=dbuser \
  --from-literal=password=******* \
  -n production

kubectl create secret generic kafka-credentials \
  --from-literal=bootstrap-servers=kafka:9092 \
  --from-literal=username=kafkauser \
  --from-literal=password=******* \
  -n production
```

2. **Create ConfigMap:**
```bash
kubectl create configmap order-service-config \
  --from-literal=db-host=oracle-db \
  --from-literal=db-port=1521 \
  -n production
```

3. **Deploy:**
```bash
kubectl apply -f order-service-deployment.yaml
```

4. **Verify:**
```bash
kubectl get pods -n production -l app=order-service
kubectl describe deployment order-service -n production
kubectl get hpa order-service-hpa -n production -w
```

---

## References
- [Kubernetes Deployment API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#deployment-v1-apps)
- [HPA Metrics & Scaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Graceful Termination](https://cloud.google.com/blog/prod-tech/kubernetes-best-practices-terminating-with-grace)
- [OKE Autoscaling Guide](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengusingclusterautoscaler.htm)
