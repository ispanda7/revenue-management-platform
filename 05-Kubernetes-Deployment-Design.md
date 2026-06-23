# Kubernetes Deployment Design

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft  
**เอกสารที่เกี่ยวข้อง:** `04-Infrastructure-Design.md`, `03-Software-Design-Specification.md`  
**กลุ่มเป้าหมาย:** DevOps Engineers, Platform Engineers, SRE

---

##  สารบัญ

1. [Namespace Strategy](#1-namespace-strategy)
2. [Deployment Design](#2-deployment-design)
3. [Service Design](#3-service-design)
4. [Ingress Design](#4-ingress-design)
5. [ConfigMap](#5-configmap)
6. [Secrets](#6-secrets)
7. [HPA (Horizontal Pod Autoscaler)](#7-hpa-horizontal-pod-autoscaler)
8. [Resource Limits](#8-resource-limits)
9. [Pod Security](#9-pod-security)
10. [Service Mesh](#10-service-mesh)
11. [Monitoring](#11-monitoring)
12. [Logging](#12-logging)

---

## 1. Namespace Strategy

### 1.1 Namespace Organization

```
┌─────────────────────────────────────────────────────────────────┐
│                    NAMESPACE ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  APPLICATION NAMESPACES                                          │
│  ├─ revenue-mgmt-dev          # Development environment         │
│  ├─ revenue-mgmt-staging      # Staging/Pre-production          │
│  ├─ revenue-mgmt-prod         # Production environment          │
│  └─ revenue-mgmt-dr           # Disaster Recovery               │
│                                                                  │
│  INFRASTRUCTURE NAMESPACES                                       │
│  ├─ ingress-nginx             # NGINX Ingress Controller        │
│  ├─ cert-manager              # TLS Certificate Management      │
│  ├─ argocd                    # GitOps CD Tool                  │
│  ├─ monitoring                # Prometheus, Grafana, Alertmanager│
│  ├─ logging                   # Fluentd, Elasticsearch, Kibana  │
│  ├─ service-mesh              # Istio/Linkerd control plane     │
│  └─ velero                    # Backup & Recovery               │
│                                                                  │
│  SYSTEM NAMESPACES                                               │
│  ├─ kube-system               # Kubernetes system components    │
│  ├─ kube-node-lease           # Node lease objects              │
│  ├─ kube-public               # Public cluster info             │
│  └─ default                   # Default namespace (avoid use)   │
│                                                                  │
└─────────────────────────────────────────────────────────────────
```

### 1.2 Namespace Configuration

```yaml
# Application Namespace Template
apiVersion: v1
kind: Namespace
metadata:
  name: revenue-mgmt-prod
  labels:
    name: revenue-mgmt-prod
    environment: production
    team: revenue-engineering
    cost-center: CC-1001
    managed-by: terraform
  annotations:
    description: "Production environment for Revenue Management"
    contact: "devops@company.com"

---
# Resource Quota for Production
apiVersion: v1
kind: ResourceQuota
metadata:
  name: revenue-mgmt-prod-quota
  namespace: revenue-mgmt-prod
spec:
  hard:
    requests.cpu: "32"
    requests.memory: "64Gi"
    limits.cpu: "64"
    limits.memory: "128Gi"
    pods: "100"
    services: "30"
    configmaps: "50"
    secrets: "50"
    persistentvolumeclaims: "20"
    replicationcontrollers: "20"
    services.loadbalancers: "5"
    services.nodeports: "0"

---
# Limit Range for Default Container Limits
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: revenue-mgmt-prod
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
        cpu: "4"
        memory: "8Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"
    - type: PersistentVolumeClaim
      max:
        storage: "100Gi"
      min:
        storage: "1Gi"

---
# Network Policy: Default Deny All
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: revenue-mgmt-prod
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# Network Policy: Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: revenue-mgmt-prod
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
        - podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### 1.3 Namespace Labels & Annotations Standard

```yaml
# Required Labels
labels:
  name: <namespace-name>
  environment: production|staging|development
  team: <team-name>
  cost-center: <cost-center-code>
  managed-by: terraform|argocd|manual

# Required Annotations
annotations:
  description: "<namespace-description>"
  contact: "<team-email>"
  slack-channel: "#<channel-name>"
  oncall-team: "<team-name>"
  compliance: "soc2|hipaa|pci-dss"
```

---

## 2. Deployment Design

### 2.1 Deployment Strategies

```
┌─────────────────────────────────────────────────────────────────┐
│                  DEPLOYMENT STRATEGIES                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. ROLLING UPDATE (Default)                                     │
│     ├─ Use Case: Most applications                              │
│     ├─ Strategy: Gradually replace old pods with new            │
│     ├─ Downtime: Zero                                           │
│     └─ Configuration:                                           │
│        maxSurge: 25%                                            │
│        maxUnavailable: 25%                                      │
│                                                                  │
│  2. BLUE-GREEN                                                   │
│     ├─ Use Case: Critical production services                   │
│     ├─ Strategy: Deploy new version alongside old, switch traffic│
│     ├─ Downtime: Zero                                           │
│     └─ Tool: Argo Rollouts                                      │
│                                                                  │
│  3. CANARY                                                       │
│     ├─ Use Case: Risky changes, A/B testing                     │
│     ├─ Strategy: Gradually shift traffic to new version         │
│     ├─ Downtime: Zero                                           │
│     └─ Tool: Argo Rollouts, Istio                               │
│                                                                  │
│  4. RECREATE                                                     │
│     ├─ Use Case: Stateful applications, database migrations     │
│     ├─ Strategy: Terminate all old pods, then create new        │
│     ├─ Downtime: Yes                                            │
│     └─ Configuration:                                           │
│        strategy: Recreate                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Deployment Manifests

#### NestJS API Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: revenue-mgmt-api
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt-api
    tier: backend
    version: v1.2.3
    environment: production
  annotations:
    description: "NestJS API Gateway for Revenue Management"
    contact: "backend-team@company.com"
    prometheus.io/scrape: "true"
    prometheus.io/port: "3000"
    prometheus.io/path: "/metrics"
spec:
  replicas: 6
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  selector:
    matchLabels:
      app: revenue-mgmt-api
  template:
    metadata:
      labels:
        app: revenue-mgmt-api
        tier: backend
        version: v1.2.3
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
    spec:
      # Node Selection
      nodeSelector:
        role: general
        kubernetes.io/os: linux
      
      # Affinity Rules
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
                        - revenue-mgmt-api
                topologyKey: kubernetes.io/hostname
            - weight: 50
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - revenue-mgmt-api
                topologyKey: topology.kubernetes.io/zone
      
      # Tolerations
      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "general"
          effect: "NoSchedule"
      
      # Service Account
      serviceAccountName: revenue-mgmt-api-sa
      
      # Security Context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
      
      # Init Containers
      initContainers:
        - name: wait-for-database
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z revenue-mgmt-db 5432; do
                echo "Waiting for database..."
                sleep 2
              done
              echo "Database is ready!"
          resources:
            requests:
              cpu: "10m"
              memory: "32Mi"
            limits:
              cpu: "50m"
              memory: "64Mi"
      
      # Main Containers
      containers:
        - name: api
          image: 444444444444.dkr.ecr.ap-southeast-1.amazonaws.com/revenue-mgmt/api:v1.2.3
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: http
              protocol: TCP
            - containerPort: 3001
              name: metrics
              protocol: TCP
          
          # Environment Variables
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "3000"
            - name: METRICS_PORT
              value: "3001"
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: revenue-mgmt-config
                  key: LOG_LEVEL
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: redis-url
            - name: KAFKA_BROKERS
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: kafka-brokers
            - name: AWS_REGION
              value: "ap-southeast-1"
            - name: S3_DOCUMENTS_BUCKET
              value: "revenue-mgmt-documents-prod"
          
          # Resource Requests & Limits
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
              ephemeral-storage: "1Gi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
              ephemeral-storage: "2Gi"
          
          # Liveness Probe
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
              scheme: HTTP
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          
          # Readiness Probe
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          
          # Startup Probe
          startupProbe:
            httpGet:
              path: /health
              port: 3000
              scheme: HTTP
            initialDelaySeconds: 0
            periodSeconds: 5
            timeoutSeconds: 3
            successThreshold: 1
            failureThreshold: 30
          
          # Volume Mounts
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: logs
              mountPath: /app/logs
            - name: config
              mountPath: /app/config
              readOnly: true
      
      # Volumes
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: "512Mi"
        - name: logs
          emptyDir:
            sizeLimit: "1Gi"
        - name: config
          configMap:
            name: revenue-mgmt-config
      
      # Termination Grace Period
      terminationGracePeriodSeconds: 30
      
      # DNS Policy
      dnsPolicy: ClusterFirst
      
      # Restart Policy
      restartPolicy: Always

---
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: revenue-mgmt-api-sa
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt-api
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::444444444444:role/revenue-mgmt-api-role
```

#### Go Billing Service Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: billing-service
  namespace: revenue-mgmt-prod
  labels:
    app: billing-service
    tier: backend
    version: v1.2.3
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: billing-service
  template:
    metadata:
      labels:
        app: billing-service
        tier: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      nodeSelector:
        role: compute
      
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - billing-service
              topologyKey: kubernetes.io/hostname
      
      containers:
        - name: billing
          image: 444444444444.dkr.ecr.ap-southeast-1.amazonaws.com/revenue-mgmt/go-services:billing-v1.2.3
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 9090
              name: grpc
            - containerPort: 8080
              name: http
            - containerPort: 9091
              name: metrics
          
          env:
            - name: ENV
              value: "production"
            - name: GRPC_PORT
              value: "9090"
            - name: HTTP_PORT
              value: "8080"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: db-host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: revenue-mgmt-secrets
                  key: db-password
          
          resources:
            requests:
              cpu: "1000m"
              memory: "1Gi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          
          livenessProbe:
            grpc:
              port: 9090
            initialDelaySeconds: 15
            periodSeconds: 20
            timeoutSeconds: 5
            failureThreshold: 3
          
          readinessProbe:
            grpc:
              port: 9090
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          
          startupProbe:
            grpc:
              port: 9090
            initialDelaySeconds: 0
            periodSeconds: 5
            failureThreshold: 20
      
      terminationGracePeriodSeconds: 60
```

### 2.3 Deployment Best Practices

```yaml
# Best Practices Checklist

Security:
  - [ ] Run as non-root user
  - [ ] Set readOnlyRootFilesystem: true (if possible)
  - [ ] Drop all capabilities
  - [ ] Use specific image tags (not :latest)
  - [ ] Enable image pull secrets
  - [ ] Set securityContext at pod and container level

Reliability:
  - [ ] Configure liveness, readiness, and startup probes
  - [ ] Set appropriate terminationGracePeriodSeconds
  - [ ] Configure pod anti-affinity for high availability
  - [ ] Use PDB (Pod Disruption Budget)
  - [ ] Set revisionHistoryLimit

Performance:
  - [ ] Set resource requests and limits
  - [ ] Use node selectors for workload isolation
  - [ ] Configure HPA for auto-scaling
  - [ ] Use local ephemeral storage limits

Observability:
  - [ ] Add Prometheus annotations
  - [ ] Configure structured logging
  - [ ] Add health check endpoints
  - [ ] Set appropriate log levels

Configuration:
  - [ ] Use ConfigMaps for non-sensitive config
  - [ ] Use Secrets for sensitive data
  - [ ] Use environment variables from ConfigMaps/Secrets
  - [ ] Add checksum annotations for auto-restart on config change
```

---

## 3. Service Design

### 3.1 Service Types

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE TYPES                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. CLUSTERIP (Default)                                          │
│     ├─ Use Case: Internal service-to-service communication      │
│     ├─ Accessibility: Within cluster only                       │
│     ├─ Example: API, Billing, Auth services                     │
│     └─ DNS: <service-name>.<namespace>.svc.cluster.local        │
│                                                                  │
│  2. NODEPORT                                                     │
│     ├─ Use Case: External access without load balancer          │
│     ├─ Accessibility: Node IP:Port (30000-32767)                │
│     ├─ Example: Development environments                        │
│     └─ Note: Not recommended for production                     │
│                                                                  │
│  3. LOADBALANCER                                                 │
│     ├─ Use Case: External access via cloud load balancer        │
│     ├─ Accessibility: External IP provided by cloud provider    │
│     ├─ Example: Production APIs                                 │
│     └─ AWS: Creates ALB/NLB automatically                       │
│                                                                  │
│  4. EXTERNALNAME                                                 │
│     ├─ Use Case: Map service to external DNS name               │
│     ├─ Accessibility: Returns CNAME record                      │
│     ├─ Example: External databases, SaaS APIs                   │
│     └─ Note: No proxying, just DNS redirection                  │
│                                                                  │
│  5. HEADLESS (ClusterIP: None)                                   │
│     ├─ Use Case: Direct pod access, stateful services           │
│     ├─ Accessibility: Returns pod IPs directly                  │
│     ├─ Example: Kafka, Elasticsearch, MongoDB                   │
│     └─ DNS: Returns A records for each pod                      │
│                                                                  │
─────────────────────────────────────────────────────────────────┘
```

### 3.2 Service Manifests

#### ClusterIP Service (Internal)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: revenue-mgmt-api
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt-api
    tier: backend
  annotations:
    description: "Internal service for NestJS API"
    prometheus.io/scrape: "true"
    prometheus.io/port: "3001"
spec:
  type: ClusterIP
  selector:
    app: revenue-mgmt-api
  ports:
    - name: http
      port: 3000
      targetPort: 3000
      protocol: TCP
    - name: metrics
      port: 3001
      targetPort: 3001
      protocol: TCP
  sessionAffinity: None

---
# Headless Service for StatefulSet (if needed)
apiVersion: v1
kind: Service
metadata:
  name: revenue-mgmt-api-headless
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt-api
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: revenue-mgmt-api
  ports:
    - name: http
      port: 3000
      targetPort: 3000
      protocol: TCP
```

#### LoadBalancer Service (External)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: revenue-mgmt-api-external
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt-api
    tier: frontend
  annotations:
    # AWS ALB Annotations
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:ap-southeast-1:444444444444:certificate/xxx"
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "https"
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-path: "/health"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "3000"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "HTTP"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "30"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"
spec:
  type: LoadBalancer
  selector:
    app: revenue-mgmt-api
  ports:
    - name: https
      port: 443
      targetPort: 3000
      protocol: TCP
    - name: http
      port: 80
      targetPort: 3000
      protocol: TCP
  loadBalancerSourceRanges:
    - 0.0.0.0/0
  externalTrafficPolicy: Local
```

#### gRPC Service (Internal)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: billing-service-grpc
  namespace: revenue-mgmt-prod
  labels:
    app: billing-service
    tier: backend
spec:
  type: ClusterIP
  selector:
    app: billing-service
  ports:
    - name: grpc
      port: 9090
      targetPort: 9090
      protocol: TCP
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
    - name: metrics
      port: 9091
      targetPort: 9091
      protocol: TCP
```

### 3.3 Service Design Best Practices

```yaml
# Service Naming Convention
Format: <app-name>-<purpose>
Examples:
  - revenue-mgmt-api
  - billing-service-grpc
  - auth-service-internal
  - payment-service-external

# Port Naming Convention
Format: <protocol> or <protocol>-<purpose>
Examples:
  - http
  - https
  - grpc
  - metrics
  - admin

# Service Discovery
Internal DNS:
  - Short: <service-name>
  - FQDN: <service-name>.<namespace>.svc.cluster.local
  - With port: <service-name>.<namespace>.svc.cluster.local:<port>

# Session Affinity
Use Cases:
  - None: Default, stateless services
  - ClientIP: When client IP should stick to same pod
  - Note: Avoid unless necessary, use sticky sessions at load balancer level
```

---

## 4. Ingress Design

### 4.1 Ingress Controller Selection

```
┌─────────────────────────────────────────────────────────────────┐
│                  INGRESS CONTROLLERS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  NGINX Ingress Controller (Recommended)                          │
│  ├─ Pros: Mature, feature-rich, good performance                │
│  ├─ Cons: Requires maintenance, configuration complexity        │
│  ├─ Use Case: General purpose, complex routing                  │
│  └─ Installation: Helm chart                                   │
│                                                                  │
│  AWS Load Balancer Controller                                    │
│  ├─ Pros: Native AWS integration, ALB/NLB support               │
│  ├─ Cons: AWS-specific, limited features                        │
│  ├─ Use Case: AWS-only deployments                              │
│  └─ Installation: Helm chart                                   │
│                                                                  │
│  Traefik                                                         │
│  ├─ Pros: Easy to configure, good dashboard                     │
│  ├─ Cons: Less mature than NGINX                                │
│  ├─ Use Case: Simple setups, dynamic configuration              │
│  └─ Installation: Helm chart                                   │
│                                                                  │
│  Istio Ingress Gateway                                           │
│  ├─ Pros: Service mesh integration, advanced traffic management │
│  ├─ Cons: Complex, requires Istio                               │
│  ├─ Use Case: When using Istio service mesh                     │
│  └─ Installation: Part of Istio                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 NGINX Ingress Controller Installation

```bash
# Add Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=3 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="nlb" \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-cross-zone-load-balancing-enabled"="true" \
  --set controller.metrics.enabled=true \
  --set controller.metrics.serviceMonitor.enabled=true \
  --set controller.metrics.serviceMonitor.additionalLabels.release="prometheus" \
  --set controller.admissionWebhooks.enabled=true \
  --set controller.config.use-forwarded-headers="true" \
  --set controller.config.compute-full-forwarded-for="true" \
  --set controller.config.use-proxy-protocol="false" \
  --set controller.config.ssl-redirect="true" \
  --set controller.config.force-ssl-redirect="true" \
  --set controller.config.hsts="true" \
  --set controller.config.hsts-max-age="31536000" \
  --set controller.config.hsts-include-subdomains="true" \
  --set controller.config.hsts-preload="true" \
  --set controller.config.server-tokens="false" \
  --set controller.config.proxy-body-size="50m" \
  --set controller.config.proxy-read-timeout="300" \
  --set controller.config.proxy-send-timeout="300" \
  --set controller.config.proxy-connect-timeout="10" \
  --set controller.config.proxy-next-upstream="error timeout" \
  --set controller.config.proxy-next-upstream-timeout="0" \
  --set controller.config.proxy-next-upstream-tries="3" \
  --set controller.resources.requests.cpu="500m" \
  --set controller.resources.requests.memory="512Mi" \
  --set controller.resources.limits.cpu="1000m" \
  --set controller.resources.limits.memory="1Gi"
```

### 4.3 Ingress Manifests

#### Production Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: revenue-mgmt-ingress
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt
    environment: production
  annotations:
    # NGINX Ingress Annotations
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-next-upstream: "error timeout"
    nginx.ingress.kubernetes.io/proxy-next-upstream-timeout: "0"
    nginx.ingress.kubernetes.io/proxy-next-upstream-tries: "3"
    
    # Rate Limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-rpm: "6000"
    nginx.ingress.kubernetes.io/limit-connections: "50"
    
    # CORS (if needed)
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://app.revenuemanagement.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, PATCH, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type, X-Correlation-Id"
    nginx.ingress.kubernetes.io/cors-expose-headers: "X-Request-Id"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/cors-max-age: "3600"
    
    # Security Headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
      more_set_headers "Content-Security-Policy: default-src 'self'";
      more_set_headers "Permissions-Policy: camera=(), microphone=(), geolocation=()";
    
    # Certificate Manager
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    
    # External DNS
    external-dns.alpha.kubernetes.io/hostname: "api.revenuemanagement.com"
    external-dns.alpha.kubernetes.io/ttl: "300"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.revenuemanagement.com
      secretName: revenue-mgmt-tls-secret
  rules:
    - host: api.revenuemanagement.com
      http:
        paths:
          # API Routes
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000
          
          - path: /api/v2
            pathType: Prefix
            backend:
              service:
                name: revenue-mgmt-api-v2
                port:
                  number: 3000
          
          # GraphQL
          - path: /graphql
            pathType: Exact
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000
          
          # Health Check
          - path: /health
            pathType: Exact
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000
          
          # Metrics (restricted)
          - path: /metrics
            pathType: Exact
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3001
          
          # WebSocket
          - path: /ws
            pathType: Prefix
            backend:
              service:
                name: revenue-mgmt-api
                port:
                  number: 3000

---
# Ingress for Internal Services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: revenue-mgmt-internal-ingress
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt
    environment: production
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - grpc.revenuemanagement.com
      secretName: revenue-mgmt-grpc-tls-secret
  rules:
    - host: grpc.revenuemanagement.com
      http:
        paths:
          - path: /billing.BillingService
            pathType: Prefix
            backend:
              service:
                name: billing-service-grpc
                port:
                  number: 9090
          
          - path: /rating.RatingService
            pathType: Prefix
            backend:
              service:
                name: rating-service-grpc
                port:
                  number: 9090
```

### 4.4 Ingress Best Practices

```yaml
# Ingress Design Principles

1. One Ingress per Application
   - Group related routes together
   - Avoid monolithic ingress configurations

2. Use Path-Based Routing
   - /api/v1/* → API service
   - /graphql → GraphQL service
   - /health → Health check service

3. TLS Termination at Ingress
   - Use cert-manager for automatic certificate management
   - Enable HSTS
   - Force SSL redirect

4. Rate Limiting
   - Protect against DDoS
   - Prevent abuse
   - Configure per-service limits

5. Security Headers
   - Add via configuration-snippet
   - X-Frame-Options, X-Content-Type-Options, etc.

6. Monitoring
   - Enable NGINX metrics
   - Monitor request rates, latencies, errors
   - Set up alerts

7. High Availability
   - Run multiple ingress controller replicas
   - Use pod anti-affinity
   - Configure HPA
```

---

## 5. ConfigMap

### 5.1 ConfigMap Design

```
┌─────────────────────────────────────────────────────────────────
│                    CONFIGMAP STRATEGY                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CONFIGMAP TYPES                                                 │
│  ├─ Application Config: Non-sensitive configuration             │
│  ├─ Feature Flags: Toggle features on/off                       │
│  ├─ Environment Config: Environment-specific settings           │
│  └─ Shared Config: Common configuration across services         │
│                                                                  │
│  BEST PRACTICES                                                  │
│  ├─ One ConfigMap per application                               │
│  ├─ Use meaningful keys                                         │
│  ├─ Version control ConfigMaps (GitOps)                         │
│  ├─ Use checksum annotations for auto-restart                   │
│  └─ Avoid storing secrets in ConfigMaps                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 ConfigMap Manifests

#### Application ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: revenue-mgmt-config
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt
    environment: production
  annotations:
    description: "Application configuration for Revenue Management"
data:
  # Application Settings
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  ENABLE_METRICS: "true"
  METRICS_PORT: "3001"
  
  # API Settings
  API_VERSION: "v1"
  API_PREFIX: "/api"
  API_TIMEOUT: "30000"
  API_RATE_LIMIT: "100"
  
  # Database Settings (non-sensitive)
  DB_POOL_MIN: "5"
  DB_POOL_MAX: "20"
  DB_CONNECTION_TIMEOUT: "10000"
  DB_IDLE_TIMEOUT: "30000"
  
  # Redis Settings (non-sensitive)
  REDIS_POOL_SIZE: "10"
  REDIS_KEY_PREFIX: "revenue-mgmt:"
  REDIS_DEFAULT_TTL: "3600"
  
  # Kafka Settings (non-sensitive)
  KAFKA_CLIENT_ID: "revenue-mgmt-api"
  KAFKA_GROUP_ID: "revenue-mgmt-consumer"
  KAFKA_AUTO_OFFSET_RESET: "latest"
  KAFKA_ENABLE_AUTO_COMMIT: "false"
  
  # Feature Flags
  FEATURE_NEW_BILLING: "true"
  FEATURE_AI_PREDICTIONS: "true"
  FEATURE_ADVANCED_REPORTING: "false"
  
  # Email Settings
  EMAIL_FROM: "noreply@revenuemanagement.com"
  EMAIL_SUPPORT: "support@revenuemanagement.com"
  
  # File Upload Settings
  MAX_FILE_SIZE: "52428800"  # 50MB in bytes
  ALLOWED_FILE_TYPES: "pdf,doc,docx,xls,xlsx,jpg,png"
  
  # Cache Settings
  CACHE_ENABLED: "true"
  CACHE_DEFAULT_TTL: "3600"
  CACHE_PRODUCT_TTL: "7200"
  CACHE_PRICING_TTL: "300"
  
  # Pagination Settings
  DEFAULT_PAGE_SIZE: "20"
  MAX_PAGE_SIZE: "100"

---
# NGINX ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: ingress-nginx
  labels:
    app: ingress-nginx
data:
  use-forwarded-headers: "true"
  compute-full-forwarded-for: "true"
  use-proxy-protocol: "false"
  ssl-redirect: "true"
  force-ssl-redirect: "true"
  hsts: "true"
  hsts-max-age: "31536000"
  hsts-include-subdomains: "true"
  hsts-preload: "true"
  server-tokens: "false"
  proxy-body-size: "50m"
  proxy-read-timeout: "300"
  proxy-send-timeout: "300"
  proxy-connect-timeout: "10"
  proxy-next-upstream: "error timeout"
  proxy-next-upstream-timeout: "0"
  proxy-next-upstream-tries: "3"
  worker-processes: "auto"
  worker-connections: "1024"
  keep-alive: "75"
  keep-alive-requests: "100"
  upstream-keepalive-connections: "320"
  upstream-keepalive-timeout: "60"
  upstream-keepalive-requests: "100"

---
# Prometheus ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
  labels:
    app: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
      scrape_timeout: 10s
    
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
                - alertmanager:9093
    
    rule_files:
      - "/etc/prometheus/rules/*.yml"
    
    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
          - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
            action: keep
            regex: default;kubernetes;https
      
      - job_name: 'kubernetes-nodes'
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        kubernetes_sd_configs:
          - role: node
        relabel_configs:
          - action: labelmap
            regex: __meta_kubernetes_node_label_(.+)
          - target_label: __address__
            replacement: kubernetes.default.svc:443
          - source_labels: [__meta_kubernetes_node_name]
            regex: (.+)
            target_label: __metrics_path__
            replacement: /api/v1/nodes/${1}/proxy/metrics
      
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
            action: replace
            target_label: __metrics_path__
            regex: (.+)
          - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            regex: ([^:]+)(?::\d+)?;(\d+)
            replacement: $1:$2
            target_label: __address__
          - action: labelmap
            regex: __meta_kubernetes_pod_label_(.+)
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: kubernetes_namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: kubernetes_pod_name
      
      - job_name: 'revenue-mgmt-api'
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - revenue-mgmt-prod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: revenue-mgmt-api
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
            action: replace
            target_label: __address__
            regex: (.+)
            replacement: $1:3001
```

### 5.3 ConfigMap Usage in Deployments

```yaml
# Using ConfigMap as Environment Variables
env:
  - name: LOG_LEVEL
    valueFrom:
      configMapKeyRef:
        name: revenue-mgmt-config
        key: LOG_LEVEL
  - name: FEATURE_NEW_BILLING
    valueFrom:
      configMapKeyRef:
        name: revenue-mgmt-config
        key: FEATURE_NEW_BILLING

# Using ConfigMap as Volume
volumeMounts:
  - name: config
    mountPath: /app/config
    readOnly: true

volumes:
  - name: config
    configMap:
      name: revenue-mgmt-config
      items:
        - key: application.yml
          path: application.yml

# Auto-restart on ConfigMap Change
template:
  metadata:
    annotations:
      checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

---

## 6. Secrets

### 6.1 Secrets Management Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                    SECRETS MANAGEMENT                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  APPROACH 1: Kubernetes Secrets (Base64 encoded)                 │
│  ├─ Use Case: Simple secrets, development                       │
│  ├─ Pros: Native, easy to use                                   │
│  ├─ Cons: Base64 is not encryption, stored in etcd              │
│  └─ Security: Enable encryption at rest in etcd                 │
│                                                                  │
│  APPROACH 2: External Secrets Operator                           │
│  ├─ Use Case: Production, compliance requirements               │
│  ├─ Pros: Integrates with AWS Secrets Manager, Vault            │
│  ├─ Cons: Additional component to manage                        │
│  └─ Security: Secrets never stored in Kubernetes                │
│                                                                  │
│  APPROACH 3: Sealed Secrets                                      │
│  ├─ Use Case: GitOps workflows                                  │
│  ├─ Pros: Can commit encrypted secrets to Git                   │
│  ├─ Cons: Requires kubeseal tool                                │
│  └─ Security: Encrypted with cluster-specific key               │
│                                                                  │
│  APPROACH 4: HashiCorp Vault                                     │
│  ├─ Use Case: Enterprise, dynamic secrets                       │
│  ├─ Pros: Dynamic credentials, leasing, rotation                │
│  ├─ Cons: Complex setup, additional infrastructure              │
│  └─ Security: Most secure option                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 Kubernetes Secrets (Basic)

```yaml
# Create Secret using kubectl (imperative)
# kubectl create secret generic revenue-mgmt-secrets \
#   --from-literal=database-url='postgresql://user:pass@host:5432/db' \
#   --from-literal=redis-url='redis://:password@host:6379' \
#   --from-literal=kafka-brokers='broker1:9092,broker2:9092' \
#   --namespace=revenue-mgmt-prod

# Secret Manifest (declarative)
apiVersion: v1
kind: Secret
metadata:
  name: revenue-mgmt-secrets
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt
    environment: production
  annotations:
    description: "Application secrets for Revenue Management"
type: Opaque
data:
  # Base64 encoded values
  # echo -n 'postgresql://revenue_admin:password@host:5432/revenue_mgmt' | base64
  database-url: cG9zdGdyZXNxbDovL3JldmVudWVfYWRtaW46cGFzc3dvcmRAaG9zdDo1NDMyL3JldmVudWVfbWdtdA==
  
  # echo -n 'redis://:password@host:6379' | base64
  redis-url: cmVkaXM6Ly86cGFzc3dvcmRAaG9zdDo2Mzc5
  
  # echo -n 'broker1:9092,broker2:9092,broker3:9092' | base64
  kafka-brokers: YnJva2VyMTo5MDkyLGJyb2tlcjI6OTA5Mixicm9rZXIzOjkwOTI=
  
  # echo -n 'your-jwt-secret-key' | base64
  jwt-secret: eW91ci1qd3Qtc2VjcmV0LWtleQ==
  
  # echo -n 'your-encryption-key' | base64
  encryption-key: eW91ci1lbmNyeXB0aW9uLWtleQ==

---
# Docker Registry Secret
apiVersion: v1
kind: Secret
metadata:
  name: ecr-registry-secret
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>

# Usage in Deployment
spec:
  template:
    spec:
      imagePullSecrets:
        - name: ecr-registry-secret
```

### 6.3 External Secrets Operator

```yaml
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace

---
# ClusterSecretStore: AWS Secrets Manager
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-southeast-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets

---
# ExternalSecret: Database Credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: revenue-mgmt-database
  namespace: revenue-mgmt-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: revenue-mgmt-secrets
    creationPolicy: Owner
    template:
      engineVersion: v2
      data:
        database-url: "{{ .host }}:{{ .port }}"
        database-username: "{{ .username }}"
        database-password: "{{ .password }}"
  data:
    - secretKey: host
      remoteRef:
        key: revenue-mgmt/database
        property: host
    - secretKey: port
      remoteRef:
        key: revenue-mgmt/database
        property: port
    - secretKey: username
      remoteRef:
        key: revenue-mgmt/database
        property: username
    - secretKey: password
      remoteRef:
        key: revenue-mgmt/database
        property: password

---
# ExternalSecret: Redis Credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: revenue-mgmt-redis
  namespace: revenue-mgmt-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: revenue-mgmt-redis-secret
    creationPolicy: Owner
  data:
    - secretKey: redis-url
      remoteRef:
        key: revenue-mgmt/redis
        property: url
    - secretKey: redis-password
      remoteRef:
        key: revenue-mgmt/redis
        property: password

---
# ExternalSecret: Kafka Credentials
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: revenue-mgmt-kafka
  namespace: revenue-mgmt-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: revenue-mgmt-kafka-secret
    creationPolicy: Owner
  data:
    - secretKey: kafka-brokers
      remoteRef:
        key: revenue-mgmt/kafka
        property: brokers
    - secretKey: kafka-username
      remoteRef:
        key: revenue-mgmt/kafka
        property: username
    - secretKey: kafka-password
      remoteRef:
        key: revenue-mgmt/kafka
        property: password
```

### 6.4 Secrets Best Practices

```yaml
# Secrets Management Best Practices

1. Never commit secrets to Git
   - Use External Secrets Operator
   - Use Sealed Secrets for GitOps
   - Use environment-specific secret stores

2. Enable encryption at rest
   - Configure etcd encryption
   - Use AWS KMS for EKS

3. Implement secret rotation
   - Rotate database passwords regularly
   - Rotate API keys periodically
   - Use AWS Secrets Manager rotation

4. Use RBAC for secrets
   - Restrict access to secrets
   - Use service accounts with minimal permissions
   - Audit secret access

5. Monitor secret usage
   - Track secret access patterns
   - Alert on unusual access
   - Regular audits

6. Backup secrets
   - Backup AWS Secrets Manager
   - Document recovery procedures
   - Test secret restoration
```

---

## 7. HPA (Horizontal Pod Autoscaler)

### 7.1 HPA Configuration

```yaml
# HPA for NestJS API
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: revenue-mgmt-api-hpa
  namespace: revenue-mgmt-prod
  labels:
    app: revenue-mgmt-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: revenue-mgmt-api
  minReplicas: 6
  maxReplicas: 20
  metrics:
    # CPU Utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # Memory Utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    
    # Custom Metric: HTTP Requests per Second
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
    
    # Custom Metric: P95 Latency
    - type: Pods
      pods:
        metric:
          name: http_request_duration_seconds_p95
        target:
          type: AverageValue
          averageValue: "2.0"
  
  # Scaling Behavior
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
        - type: Pods
          value: 2
          periodSeconds: 60
      selectPolicy: Min
    
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
        - type: Pods
          value: 4
          periodSeconds: 60
      selectPolicy: Max

---
# HPA for Go Billing Service
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: billing-service-hpa
  namespace: revenue-mgmt-prod
  labels:
    app: billing-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: billing-service
  minReplicas: 4
  maxReplicas: 12
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 75
    
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 85
    
    # Custom Metric: gRPC Requests
    - type: Pods
      pods:
        metric:
          name: grpc_requests_per_second
        target:
          type: AverageValue
          averageValue: 50
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 120
    
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60

---
# HPA for Python AI Service
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-service-hpa
  namespace: revenue-mgmt-prod
  labels:
    app: ai-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-service
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
    
    # Custom Metric: Prediction Requests
    - type: Pods
      pods:
        metric:
          name: prediction_requests_per_second
        target:
          type: AverageValue
          averageValue: 20
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 600  # Longer stabilization for AI services
      policies:
        - type: Percent
          value: 25
          periodSeconds: 120
    
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
```

### 7.2 Custom Metrics Setup

```yaml
# Prometheus Adapter for Custom Metrics
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-adapter
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prometheus-adapter
  template:
    metadata:
      labels:
        app: prometheus-adapter
    spec:
      containers:
        - name: prometheus-adapter
          image: directxman12/k8s-prometheus-adapter:v0.11.1
          args:
            - --cert-dir=/tmp/cert
            - --logtostderr=true
            - --metrics-relist-interval=1m
            - --prometheus-url=http://prometheus:9090
            - --secure-port=6443
          ports:
            - containerPort: 6443
              name: https
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"

---
# ConfigMap for Prometheus Adapter Rules
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
      - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace:
              resource: namespace
            pod:
              resource: pod
        name:
          matches: "^http_requests_total"
          as: "http_requests_per_second"
        metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
      
      - seriesQuery: 'http_request_duration_seconds_bucket{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace:
              resource: namespace
            pod:
              resource: pod
        name:
          matches: "^http_request_duration_seconds_bucket"
          as: "http_request_duration_seconds_p95"
        metricsQuery: |
          histogram_quantile(0.95,
            sum(rate(<<.Series>>{<<.LabelMatchers>>,le!=""}[5m])) by (le,<<.GroupBy>>)
          )
      
      - seriesQuery: 'grpc_server_handled_total{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace:
              resource: namespace
            pod:
              resource: pod
        name:
          matches: "^grpc_server_handled_total"
          as: "grpc_requests_per_second"
        metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

### 7.3 HPA Best Practices

```yaml
# HPA Best Practices

1. Set Appropriate Min/Max Replicas
   - Min: Based on baseline traffic
   - Max: Based on budget and capacity
   - Example: Min 6, Max 20 for production API

2. Use Multiple Metrics
   - CPU and memory for resource-based scaling
   - Custom metrics for business logic scaling
   - Avoid relying on single metric

3. Configure Scaling Behavior
   - Stabilization window to prevent flapping
   - Different policies for scale up/down
   - Conservative scale down, aggressive scale up

4. Monitor HPA Performance
   - Track scaling events
   - Monitor metric values
   - Alert on scaling failures

5. Test Scaling Behavior
   - Load test to verify scaling
   - Validate custom metrics
   - Test scale down behavior

6. Consider VPA (Vertical Pod Autoscaler)
   - For workloads that can't scale horizontally
   - For stateful applications
   - Use with caution in production
```

---

## 8. Resource Limits

### 8.1 Resource Management Strategy

```
┌─────────────────────────────────────────────────────────────────┐
│                  RESOURCE MANAGEMENT                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  QoS CLASSES                                                     │
│  ├─ Guaranteed: requests = limits (highest priority)            │
│  ├─ Burstable: requests < limits (medium priority)              │
│  └─ BestEffort: no requests/limits (lowest priority)            │
│                                                                  │
│  RESOURCE TYPES                                                  │
│  ├─ CPU: Measured in millicores (m)                             │
│  ├─ Memory: Measured in Mi/Gi                                   │
│  ├─ Ephemeral Storage: Temporary storage                        │
│  └─ Extended Resources: GPU, FPGA, etc.                         │
│                                                                  │
│  BEST PRACTICES                                                  │
│  ├─ Always set requests and limits                              │
│  ├─ Use Guaranteed QoS for critical services                    │
│  ├─ Monitor actual usage vs allocated                           │
│  └─ Right-size based on metrics                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Resource Configuration Examples

```yaml
# NestJS API - Burstable QoS
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
    ephemeral-storage: "1Gi"
  limits:
    cpu: "1000m"
    memory: "1Gi"
    ephemeral-storage: "2Gi"

# Go Billing Service - Guaranteed QoS
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
    ephemeral-storage: "2Gi"
  limits:
    cpu: "1000m"
    memory: "2Gi"
    ephemeral-storage: "2Gi"

# Python AI Service - Burstable QoS
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
    ephemeral-storage: "5Gi"
  limits:
    cpu: "2000m"
    memory: "4Gi"
    ephemeral-storage: "10Gi"

# Redis Proxy - Guaranteed QoS
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "500m"
    memory: "1Gi"

# Monitoring Sidecar - BestEffort (not recommended)
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "100m"
    memory: "128Mi"
```

### 8.3 Resource Quotas and Limits

```yaml
# Namespace Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
  namespace: revenue-mgmt-prod
spec:
  hard:
    requests.cpu: "32"
    requests.memory: "64Gi"
    limits.cpu: "64"
    limits.memory: "128Gi"
    requests.ephemeral-storage: "100Gi"
    limits.ephemeral-storage: "200Gi"
    pods: "100"

---
# Limit Range for Default Values
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: revenue-mgmt-prod
spec:
  limits:
    - type: Container
      default:
        cpu: "500m"
        memory: "512Mi"
        ephemeral-storage: "1Gi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
        ephemeral-storage: "512Mi"
      max:
        cpu: "4"
        memory: "8Gi"
        ephemeral-storage: "10Gi"
      min:
        cpu: "50m"
        memory: "64Mi"
        ephemeral-storage: "256Mi"
    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"
        ephemeral-storage: "20Gi"
    - type: PersistentVolumeClaim
      max:
        storage: "100Gi"
      min:
        storage: "1Gi"
```

### 8.4 Resource Monitoring and Optimization

```yaml
# Resource Monitoring Queries (Prometheus)

# CPU Usage by Pod
sum(rate(container_cpu_usage_seconds_total{namespace="revenue-mgmt-prod"}[5m])) by (pod)

# Memory Usage by Pod
sum(container_memory_working_set_bytes{namespace="revenue-mgmt-prod"}) by (pod)

# Resource Requests vs Limits
sum(kube_pod_container_resource_requests{namespace="revenue-mgmt-prod", resource="cpu"}) by (pod)
sum(kube_pod_container_resource_limits{namespace="revenue-mgmt-prod", resource="cpu"}) by (pod)

# OOM Kill Events
sum(increase(kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}[1h])) by (pod)

# CPU Throttling
sum(rate(container_cpu_cfs_throttled_seconds_total{namespace="revenue-mgmt-prod"}[5m])) by (pod)

# Resource Optimization Recommendations

1. Right-Size Based on Metrics
   - Monitor actual usage for 2 weeks
   - Adjust requests to P95 usage
   - Set limits to P99 usage + 20%

2. Use Vertical Pod Autoscaler (VPA)
   - Get recommendations
   - Apply in auto mode (with caution)
   - Review recommendations regularly

3. Implement Resource Budgets
   - Set namespace quotas
   - Track resource consumption
   - Alert on quota breaches

4. Optimize for Cost
   - Use spot instances for non-critical workloads
   - Schedule batch jobs during off-peak
   - Delete unused resources
```

---

## 9. Pod Security

### 9.1 Pod Security Standards

```
┌─────────────────────────────────────────────────────────────────┐
│                  POD SECURITY STANDARDS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PRIVILEGED (Not Recommended)                                    │
│  ├─ Allows all privileges                                       │
│  ├─ No restrictions                                             │
│  └─ Use Case: System daemons, kernel modules                    │
│                                                                  │
│  BASELINE (Recommended for most workloads)                       │
│  ├─ Prevents known privilege escalations                        │
│  ├─ Restricts host access                                       │
│  └─ Use Case: General applications                              │
│                                                                  │
│  RESTRICTED (Recommended for sensitive workloads)                │
│  ├─ Heavily restricted                                          │
│  ├─ Follows security best practices                             │
│  └─ Use Case: Financial services, healthcare                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2 Pod Security Configuration

```yaml
# Pod Security Context (Restricted)
apiVersion: v1
kind: Pod
metadata:
  name: revenue-mgmt-api-pod
  namespace: revenue-mgmt-prod
spec:
  # Pod-level security context
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
    seccompProfile:
      type: RuntimeDefault
  
  containers:
    - name: api
      image: revenue-mgmt/api:v1.2.3
      
      # Container-level security context
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 1001
        capabilities:
          drop:
            - ALL
        seccompProfile:
          type: RuntimeDefault
      
      # Volume mounts for writable directories
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: logs
          mountPath: /app/logs
        - name: cache
          mountPath: /app/.cache
  
  volumes:
    - name: tmp
      emptyDir:
        sizeLimit: "512Mi"
    - name: logs
      emptyDir:
        sizeLimit: "1Gi"
    - name: cache
      emptyDir:
        sizeLimit: "256Mi"

---
# Pod Security Admission (Namespace Level)
apiVersion: v1
kind: Namespace
metadata:
  name: revenue-mgmt-prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest

---
# Network Policy for Pod Security
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: revenue-mgmt-prod
spec:
  podSelector:
    matchLabels:
      app: revenue-mgmt-api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 3000
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: revenue-mgmt-db
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### 9.3 Security Best Practices

```yaml
# Pod Security Checklist

Container Security:
  - [ ] Run as non-root user
  - [ ] Set readOnlyRootFilesystem: true
  - [ ] Drop all capabilities
  - [ ] Set allowPrivilegeEscalation: false
  - [ ] Use seccomp profile
  - [ ] Use specific image tags (not :latest)
  - [ ] Scan images for vulnerabilities

Pod Security:
  - [ ] Set pod-level securityContext
  - [ ] Use service accounts with minimal permissions
  - [ ] Configure network policies
  - [ ] Set resource limits
  - [ ] Use pod disruption budgets

Image Security:
  - [ ] Use private container registry
  - [ ] Enable image scanning
  - [ ] Sign images with cosign
  - [ ] Verify image signatures
  - [ ] Use distroless base images

Secrets Security:
  - [ ] Use External Secrets Operator
  - [ ] Enable encryption at rest
  - [ ] Implement secret rotation
  - [ ] Restrict secret access with RBAC
  - [ ] Audit secret access

Network Security:
  - [ ] Implement default deny network policies
  - [ ] Use TLS for all communication
  - [ ] Restrict egress traffic
  - [ ] Use service mesh for mTLS
  - [ ] Monitor network traffic

Runtime Security:
  - [ ] Use Falco for runtime threat detection
  - [ ] Monitor for privilege escalations
  - [ ] Detect unusual process execution
  - [ ] Alert on suspicious network activity
  - [ ] Regular security audits
```

---

## 10. Service Mesh

### 10.1 Service Mesh Selection

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE MESH OPTIONS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Istio (Recommended for Enterprise)                              │
│  ├─ Pros: Feature-rich, strong community, CNCF project          │
│  ├─ Cons: Complex, resource-intensive                           │
│  ├─ Features: mTLS, traffic management, observability, security │
│  └─ Use Case: Large-scale, complex microservices                │
│                                                                  │
│  Linkerd (Recommended for Simplicity)                            │
│  ├─ Pros: Lightweight, easy to install, fast                    │
│  ├─ Cons: Fewer features than Istio                             │
│  ├─ Features: mTLS, observability, traffic splitting            │
│  └─ Use Case: Simple setups, performance-critical               │
│                                                                  │
│  AWS App Mesh (AWS Native)                                       │
│  ├─ Pros: Native AWS integration, managed                       │
│  ├─ Cons: AWS-specific, limited features                        │
│  ├─ Features: mTLS, traffic management, observability           │
│  └─ Use Case: AWS-only deployments                              │
│                                                                  │
│  Consul Connect (HashiCorp)                                      │
│  ├─ Pros: Multi-platform, service discovery                     │
│  ├─ Cons: Requires Consul cluster                               │
│  ├─ Features: mTLS, service discovery, traffic management       │
│  └─ Use Case: Hybrid cloud, multi-platform                      │
│                                                                  │
─────────────────────────────────────────────────────────────────┘
```

### 10.2 Istio Installation

```bash
# Install Istio using istioctl
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.20.0 sh -
cd istio-1.20.0
export PATH=$PWD/bin:$PATH

# Install Istio with demo profile (for development)
istioctl install --set profile=demo -y

# Install Istio with production profile
istioctl install --set profile=default \
  --set meshConfig.accessLogFile=/dev/stdout \
  --set meshConfig.enableAutoMtls=true \
  --set meshConfig.defaultConfig.tracing.sampling=1.0 \
  --set components.pilot.k8s.resources.requests.cpu=500m \
  --set components.pilot.k8s.resources.requests.memory=2Gi \
  --set components.pilot.k8s.resources.limits.cpu=1000m \
  --set components.pilot.k8s.resources.limits.memory=4Gi \
  -y

# Enable sidecar injection for namespace
kubectl label namespace revenue-mgmt-prod istio-injection=enabled

# Verify installation
istioctl verify-install
kubectl get pods -n istio-system
```

### 10.3 Istio Configuration

```yaml
# VirtualService: Traffic Management
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: revenue-mgmt-api-vs
  namespace: revenue-mgmt-prod
spec:
  hosts:
    - revenue-mgmt-api
  http:
    # Canary deployment: 90% to v1, 10% to v2
    - route:
        - destination:
            host: revenue-mgmt-api
            subset: v1
          weight: 90
        - destination:
            host: revenue-mgmt-api
            subset: v2
          weight: 10
      timeout: 30s
      retries:
        attempts: 3
        perTryTimeout: 10s
        retryOn: gateway-error,connect-failure,refused-stream
    
    # Fault injection for testing
    - fault:
        abort:
          httpStatus: 500
          percentage:
            value: 0.1
        delay:
          fixedDelay: 5s
          percentage:
            value: 0.1
      route:
        - destination:
            host: revenue-mgmt-api
            subset: v1

---
# DestinationRule: Traffic Policies
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: revenue-mgmt-api-dr
  namespace: revenue-mgmt-prod
spec:
  host: revenue-mgmt-api
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: DEFAULT
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2

---
# Gateway: External Traffic
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: revenue-mgmt-gateway
  namespace: revenue-mgmt-prod
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "api.revenuemanagement.com"
      tls:
        httpsRedirect: true
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "api.revenuemanagement.com"
      tls:
        mode: SIMPLE
        credentialName: revenue-mgmt-tls-secret

---
# AuthorizationPolicy: Security
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: revenue-mgmt-api-authz
  namespace: revenue-mgmt-prod
spec:
  selector:
    matchLabels:
      app: revenue-mgmt-api
  rules:
    - from:
        - source:
            principals:
              - "cluster.local/ns/ingress-nginx/sa/ingress-nginx"
              - "cluster.local/ns/revenue-mgmt-prod/sa/frontend"
      to:
        - operation:
            methods: ["GET", "POST", "PUT", "DELETE", "PATCH"]
            paths: ["/api/*", "/graphql", "/health"]

---
# RequestAuthentication: JWT Validation
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: revenue-mgmt-api-jwt
  namespace: revenue-mgmt-prod
spec:
  selector:
    matchLabels:
      app: revenue-mgmt-api
  jwtRules:
    - issuer: "https://auth.revenuemanagement.com"
      jwksUri: "https://auth.revenuemanagement.com/.well-known/jwks.json"
      audiences:
        - "revenue-mgmt-api"
      forwardOriginalToken: true
```

### 10.4 Service Mesh Observability

```yaml
# Enable Istio Telemetry
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: revenue-mgmt-telemetry
  namespace: revenue-mgmt-prod
spec:
  selector:
    matchLabels:
      app: revenue-mgmt-api
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: REQUEST_COUNT
          tagOverrides:
            request_host:
              value: request.host
            response_code:
              value: response.code
  accessLogging:
    - providers:
        - name: envoy
      filter:
        expression: response.code >= 400

---
# Kiali Dashboard for Service Mesh Visualization
# Install Kiali
helm install kiali-server kiali/kiali-server \
  --namespace istio-system \
  --set external_services.prometheus.url=http://prometheus:9090 \
  --set external_services.tracing.in_cluster_url=http://jaeger:16686 \
  --set external_services.tracing.url=https://jaeger.revenuemanagement.com \
  --set external_services.grafana.in_cluster_url=http://grafana:3000 \
  --set external_services.grafana.url=https://grafana.revenuemanagement.com
```

---

## 11. Monitoring

### 11.1 Monitoring Stack Architecture

```
─────────────────────────────────────────────────────────────────┐
│                    MONITORING STACK                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  METRICS COLLECTION                                              │
│  ├─ Prometheus: Metrics collection and storage                  │
│  ├─ Node Exporter: Host-level metrics                           │
│  ├─ kube-state-metrics: Kubernetes object metrics               │
│  └─ Custom exporters: Application-specific metrics              │
│                                                                  │
│  VISUALIZATION                                                   │
│  ├─ Grafana: Dashboards and visualization                       │
│  ├─ Kiali: Service mesh visualization                           │
│  └─ Kubernetes Dashboard: Basic cluster overview                │
│                                                                  │
│  ALERTING                                                        │
│  ├─ Alertmanager: Alert routing and notification                │
│  ├─ PagerDuty: Critical incident management                     │
│  ├─ Slack: Team notifications                                   │
│  └─ Email: Fallback notification                                │
│                                                                  │
│  TRACING                                                         │
│  ├─ Jaeger: Distributed tracing                                 │
│  ├─ OpenTelemetry: Instrumentation library                      │
│  └─ AWS X-Ray: AWS service tracing                              │
│                                                                  │
│  PROFILING                                                       │
│  ├─ Pyroscope: Continuous profiling                             │
│  ├─ Parca: eBPF-based profiling                                 │
│  └─ pprof: Go profiling                                         │
│                                                                  │
─────────────────────────────────────────────────────────────────┘
```

### 11.2 Prometheus Installation

```bash
# Install Prometheus using kube-prometheus-stack
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values prometheus-values.yaml
```

```yaml
# prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    resources:
      requests:
        cpu: 1000m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 4Gi
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false

alertmanager:
  alertmanagerSpec:
    replicas: 2
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi

grafana:
  enabled: true
  replicas: 2
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  adminPassword: <change-me>
  sidecar:
    dashboards:
      enabled: true
      searchNamespace: ALL
    datasources:
      enabled: true

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
```

### 11.3 Grafana Dashboards

```yaml
# Grafana Dashboard: Revenue Management Overview
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-revenue-mgmt
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  revenue-mgmt-overview.json: |
    {
      "dashboard": {
        "title": "Revenue Management Overview",
        "uid": "revenue-mgmt-overview",
        "panels": [
          {
            "title": "API Request Rate",
            "type": "graph",
            "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0},
            "targets": [
              {
                "expr": "sum(rate(http_requests_total{namespace=\"revenue-mgmt-prod\", job=\"revenue-mgmt-api\"}[5m]))",
                "legendFormat": "Requests/sec"
              }
            ]
          },
          {
            "title": "Error Rate",
            "type": "graph",
            "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
            "targets": [
              {
                "expr": "sum(rate(http_requests_total{namespace=\"revenue-mgmt-prod\", status=~\"5..\"}[5m])) / sum(rate(http_requests_total{namespace=\"revenue-mgmt-prod\"}[5m]))",
                "legendFormat": "Error Rate"
              }
            ]
          },
          {
            "title": "P95 Latency",
            "type": "graph",
            "gridPos": {"h": 8, "w": 12, "x": 0, "y": 8},
            "targets": [
              {
                "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{namespace=\"revenue-mgmt-prod\"}[5m])) by (le))",
                "legendFormat": "P95"
              }
            ]
          },
          {
            "title": "Active Pods",
            "type": "stat",
            "gridPos": {"h": 8, "w": 12, "x": 12, "y": 8},
            "targets": [
              {
                "expr": "sum(kube_pod_status_phase{namespace=\"revenue-mgmt-prod\", phase=\"Running\"})",
                "legendFormat": "Running Pods"
              }
            ]
          },
          {
            "title": "Database Connections",
            "type": "graph",
            "gridPos": {"h": 8, "w": 12, "x": 0, "y": 16},
            "targets": [
              {
                "expr": "pg_stat_activity_count{datname=\"revenue_mgmt\"}",
                "legendFormat": "Active Connections"
              }
            ]
          },
          {
            "title": "Redis Hit Rate",
            "type": "gauge",
            "gridPos": {"h": 8, "w": 12, "x": 12, "y": 16},
            "targets": [
              {
                "expr": "redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total)",
                "legendFormat": "Hit Rate"
              }
            ]
          }
        ],
        "time": {"from": "now-1h", "to": "now"},
        "refresh": "30s"
      }
    }
```

### 11.4 Alerting Rules

```yaml
# Prometheus Alerting Rules
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-alerts
  namespace: monitoring
data:
  revenue-mgmt-alerts.yaml: |
    groups:
      - name: revenue-mgmt-critical
        rules:
          - alert: HighErrorRate
            expr: |
              sum(rate(http_requests_total{namespace="revenue-mgmt-prod", status=~"5.."}[5m])) 
              / 
              sum(rate(http_requests_total{namespace="revenue-mgmt-prod"}[5m])) > 0.05
            for: 5m
            labels:
              severity: critical
              team: revenue-engineering
            annotations:
              summary: "High error rate detected"
              description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"
              runbook_url: "https://wiki.internal/runbooks/high-error-rate"
          
          - alert: DatabaseDown
            expr: pg_up{namespace="revenue-mgmt-prod"} == 0
            for: 1m
            labels:
              severity: critical
              team: revenue-engineering
            annotations:
              summary: "Database is down"
              description: "PostgreSQL instance {{ $labels.instance }} is down"
          
          - alert: HighMemoryUsage
            expr: |
              sum(container_memory_working_set_bytes{namespace="revenue-mgmt-prod"}) 
              / 
              sum(container_spec_memory_limit_bytes{namespace="revenue-mgmt-prod"}) > 0.9
            for: 10m
            labels:
              severity: critical
              team: revenue-engineering
            annotations:
              summary: "High memory usage"
              description: "Memory usage is {{ $value | humanizePercentage }}"
      
      - name: revenue-mgmt-warning
        rules:
          - alert: HighLatency
            expr: |
              histogram_quantile(0.95, 
                rate(http_request_duration_seconds_bucket{namespace="revenue-mgmt-prod"}[5m])
              ) > 2
            for: 10m
            labels:
              severity: warning
              team: revenue-engineering
            annotations:
              summary: "High API latency"
              description: "P95 latency is {{ $value }}s"
          
          - alert: PodRestarting
            expr: |
              increase(kube_pod_container_status_restarts_total{namespace="revenue-mgmt-prod"}[1h]) > 5
            for: 5m
            labels:
              severity: warning
              team: revenue-engineering
            annotations:
              summary: "Pod restarting frequently"
              description: "Pod {{ $labels.pod }} has restarted {{ $value }} times in the last hour"
          
          - alert: DiskSpaceLow
            expr: |
              (node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"} 
              / 
              node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"}) < 0.1
            for: 5m
            labels:
              severity: warning
              team: revenue-engineering
            annotations:
              summary: "Low disk space"
              description: "Disk space is {{ $value | humanizePercentage }}"

---
# Alertmanager Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 5m
      smtp_smarthost: 'smtp.company.com:587'
      smtp_from: 'alertmanager@revenuemanagement.com'
      smtp_auth_username: 'alertmanager@company.com'
      smtp_auth_password: '<password>'
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack-notifications'
      routes:
        - match:
            severity: 'critical'
          receiver: 'pagerduty-critical'
          continue: true
        - match:
            severity: 'warning'
          receiver: 'slack-notifications'
    
    receivers:
      - name: 'slack-notifications'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/xxx/xxx/xxx'
            channel: '#revenue-mgmt-alerts'
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
            send_resolved: true
      
      - name: 'pagerduty-critical'
        pagerduty_configs:
          - service_key: '<pagerduty-service-key>'
            description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'
            severity: 'critical'
```

---

## 12. Logging

### 12.1 Logging Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOGGING ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LOG COLLECTION                                                  │
│  ├─ Fluentd/Fluent Bit: Log collection and forwarding           │
│  ├─ Filebeat: Alternative log collector                         │
│  └─ Vector: High-performance log collector                      │
│                                                                  │
│  LOG STORAGE                                                     │
│  ├─ Elasticsearch: Log storage and indexing                     │
│  ├─ Loki: Lightweight log aggregation (Grafana Labs)            │
│  └─ CloudWatch Logs: AWS managed log service                    │
│                                                                  │
│  LOG VISUALIZATION                                               │
│  ├─ Kibana: Elasticsearch visualization                         │
│  ├─ Grafana: Loki visualization                                 │
│  └─ CloudWatch Insights: AWS log analysis                       │
│                                                                  │
│  LOG PROCESSING                                                  │
│  ├─ Parsing: Extract structured data from logs                  │
│  ├─ Enrichment: Add metadata (pod, namespace, etc.)             │
│  ├─ Filtering: Remove unnecessary logs                          │
│  └─ Routing: Send logs to different destinations                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 12.2 Fluentd Installation

```bash
# Install Fluentd using Helm
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

helm install fluentd fluent/fluentd \
  --namespace logging \
  --create-namespace \
  --values fluentd-values.yaml
```

```yaml
# fluentd-values.yaml
fluentd:
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  
  config:
    system:
      log_level: info
    
    inputs:
      - type: tail
        path: /var/log/containers/*.log
        pos_file: /var/log/fluentd-containers.log.pos
        tag: kubernetes.*
        read_from_head: true
        <parse>
          @type json
          time_key time
          time_format %Y-%m-%dT%H:%M:%S.%NZ
        </parse>
    
    filters:
      - type: kubernetes_metadata
      - type: record_transformer
        <record>
          hostname "#{Socket.gethostname}"
        </record>
    
    outputs:
      - type: elasticsearch
        host: elasticsearch.logging.svc.cluster.local
        port: 9200
        index_name: fluentd-kubernetes
        <buffer>
          @type file
          path /var/log/fluentd/buffer
          flush_interval 5s
          chunk_limit_size 2M
          queue_limit_length 32
          retry_max_interval 30
          retry_forever true
        </buffer>
```

### 12.3 Elasticsearch Installation

```bash
# Install Elasticsearch using Helm
helm repo add elastic https://helm.elastic.co
helm repo update

helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --create-namespace \
  --set replicas=3 \
  --set resources.requests.memory=2Gi \
  --set resources.limits.memory=4Gi \
  --set persistence.enabled=true \
  --set persistence.size=100Gi
```

### 12.4 Log Format Standard

```json
{
  "timestamp": "2026-06-22T10:30:00.000Z",
  "level": "INFO",
  "service": "revenue-mgmt-api",
  "version": "1.2.3",
  "environment": "production",
  "correlationId": "abc-123-def-456",
  "traceId": "trace-789",
  "spanId": "span-012",
  "userId": "user-456",
  "tenantId": "tenant-789",
  "action": "invoice.generated",
  "message": "Invoice INV-001 generated successfully",
  "metadata": {
    "invoiceId": "INV-001",
    "accountId": "ACC-123",
    "amount": 1500.00,
    "currency": "USD",
    "duration_ms": 245
  },
  "context": {
    "ip": "192.168.1.1",
    "userAgent": "Mozilla/5.0...",
    "region": "ap-southeast-1"
  }
}
```

### 12.5 Logging Best Practices

```yaml
# Logging Best Practices

1. Use Structured Logging
   - JSON format for easy parsing
   - Consistent field names
   - Include metadata (correlation ID, trace ID)

2. Implement Log Levels
   - ERROR: System failures, data corruption
   - WARN: Potential issues, degraded performance
   - INFO: Business transactions, important events
   - DEBUG: Detailed processing info (dev only)
   - TRACE: Very detailed tracing (dev only)

3. Include Context
   - Correlation ID for request tracing
   - User ID for audit trail
   - Tenant ID for multi-tenancy
   - Service name and version

4. Avoid Sensitive Data
   - Never log passwords, tokens, PII
   - Mask sensitive fields
   - Use log sanitization

5. Implement Log Rotation
   - Set retention policies
   - Archive old logs
   - Delete logs after retention period

6. Monitor Log Volume
   - Track log ingestion rates
   - Alert on unusual spikes
   - Optimize verbose logging

7. Centralize Logs
   - Use centralized log storage
   - Enable log search and analysis
   - Set up log-based alerts

8. Secure Logs
   - Encrypt logs at rest
   - Restrict log access
   - Audit log access
```

---

## ภาคผนวก

### ภาคผนวก A: Deployment Checklist

```yaml
Pre-Deployment Checklist:
  Namespace:
    - [ ] Namespace created with proper labels
    - [ ] Resource quotas configured
    - [ ] Limit ranges configured
    - [ ] Network policies applied
    - [ ] Pod security standards enforced
  
  Deployment:
    - [ ] Deployment manifest reviewed
    - [ ] Resource requests/limits set
    - [ ] Health checks configured
    - [ ] Security context applied
    - [ ] Image tags specific (not :latest)
    - [ ] Pod anti-affinity configured
    - [ ] Service account created
  
  Service:
    - [ ] Service type appropriate
    - [ ] Ports correctly configured
    - [ ] Selectors match deployment
    - [ ] Session affinity set if needed
  
  Ingress:
    - [ ] Ingress controller installed
    - [ ] TLS certificates configured
    - [ ] Rate limiting enabled
    - [ ] Security headers added
    - [ ] Routing rules tested
  
  ConfigMap & Secrets:
    - [ ] ConfigMaps created
    - [ ] Secrets created (via External Secrets)
    - [ ] Checksum annotations added
    - [ ] Secrets not in Git
  
  HPA:
    - [ ] HPA configured
    - [ ] Min/max replicas set
    - [ ] Metrics defined
    - [ ] Scaling behavior configured
  
  Monitoring:
    - [ ] Prometheus annotations added
    - [ ] Grafana dashboards created
    - [ ] Alerts configured
    - [ ] Log collection enabled
  
  Security:
    - [ ] Network policies applied
    - [ ] Pod security standards enforced
    - [ ] RBAC configured
    - [ ] Secrets encrypted
    - [ ] Images scanned
```

### ภาคผนวก B: Troubleshooting Guide

```markdown
# Common Issues and Solutions

## Issue: Pod stuck in Pending state
**Cause:** Insufficient resources, node selector mismatch, taints/tolerations
**Solution:**
1. Check pod events: `kubectl describe pod <pod-name>`
2. Check node resources: `kubectl describe nodes`
3. Verify node selectors and tolerations
4. Check resource quotas

## Issue: Pod stuck in ContainerCreating
**Cause:** Image pull failure, volume mount issues
**Solution:**
1. Check image name and tag
2. Verify image pull secrets
3. Check volume claims and mounts
4. Review pod events

## Issue: CrashLoopBackOff
**Cause:** Application crash, configuration error, missing dependencies
**Solution:**
1. Check pod logs: `kubectl logs <pod-name>`
2. Check previous logs: `kubectl logs <pod-name> --previous`
3. Verify environment variables and secrets
4. Check health check configuration

## Issue: OOMKilled
**Cause:** Memory limit too low, memory leak
**Solution:**
1. Increase memory limits
2. Profile application for memory leaks
3. Check actual memory usage
4. Consider vertical scaling

## Issue: High Latency
**Cause:** Resource contention, network issues, downstream dependencies
**Solution:**
1. Check resource utilization
2. Review network policies
3. Check downstream service health
4. Analyze distributed traces

## Issue: HPA not scaling
**Cause:** Metrics not available, HPA misconfiguration
**Solution:**
1. Check HPA status: `kubectl describe hpa <hpa-name>`
2. Verify metrics server is running
3. Check custom metrics configuration
4. Review HPA events
```

### ภาคผนวก C: Useful kubectl Commands

```bash
# Pod Management
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl exec -it <pod-name> -n <namespace> -- /bin/bash

# Deployment Management
kubectl get deployments -n <namespace>
kubectl describe deployment <deployment-name> -n <namespace>
kubectl rollout status deployment/<deployment-name> -n <namespace>
kubectl rollout history deployment/<deployment-name> -n <namespace>
kubectl rollout undo deployment/<deployment-name> -n <namespace>
kubectl scale deployment/<deployment-name> --replicas=5 -n <namespace>

# Service Management
kubectl get services -n <namespace>
kubectl describe service <service-name> -n <namespace>
kubectl get endpoints <service-name> -n <namespace>

# Ingress Management
kubectl get ingress -n <namespace>
kubectl describe ingress <ingress-name> -n <namespace>

# HPA Management
kubectl get hpa -n <namespace>
kubectl describe hpa <hpa-name> -n <namespace>

# Resource Monitoring
kubectl top pods -n <namespace>
kubectl top nodes
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota -n <namespace>

# Debugging
kubectl get events -n <namespace> --sort-by='.metadata.creationTimestamp'
kubectl get pods -n <namespace> -o wide
kubectl get pods -n <namespace> -o yaml
kubectl debug -it <pod-name> -n <namespace> --image=busybox
```

---

**เอกสารนี้จัดทำขึ้นเพื่อใช้เป็นแนวทางในการ Deploy และ Manage Kubernetes สำหรับระบบ Revenue Management**

**เวอร์ชัน:** 1.0  
**วันที่:** 22 มิถุนายน 2026  
**สถานะ:** Draft for Review  
**ผู้จัดทำ:** DevOps Team
