# Part 71: Kubernetes Advanced

## Kubernetes Advanced — Production-ready K8s สำหรับ Spring Boot

---

## 🎯 เป้าหมายของ Part นี้

- ConfigMaps และ Secrets management
- Horizontal Pod Autoscaling (HPA)
- StatefulSets สำหรับ databases
- Helm charts
- GitOps กับ Argo CD
- สร้าง Production K8s setup

---

## 📖 1. ConfigMaps และ Secrets

### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: spring-app-config
  namespace: production
data:
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://postgres-service:5432/myapp
      redis:
        host: redis-service
        port: 6379
    server:
      port: 8080
    logging:
      level:
        root: INFO
        com.example: DEBUG
```

### Secret (ข้อมูลละเอียดอ่อน)

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: spring-app-secrets
  namespace: production
type: Opaque
data:
  # base64 encoded values
  db-password: cGFzc3dvcmQxMjM=
  jwt-secret: bXlzdXBlcnNlY3JldGtleQ==
  api-key: c2VjcmV0YXBpa2V5MTIz
```

### Deployment ที่ใช้ ConfigMap และ Secret

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
  namespace: production
  labels:
    app: spring-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
        version: "1.0.0"
    spec:
      containers:
        - name: spring-app
          image: registry.example.com/spring-app:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: spring-app-secrets
                  key: db-password
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: spring-app-secrets
                  key: jwt-secret
            - name: SPRING_CONFIG_IMPORT
              value: "optional:configtree:/etc/config/"
          volumeMounts:
            - name: config-volume
              mountPath: /etc/config
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
      volumes:
        - name: config-volume
          configMap:
            name: spring-app-config
```

---

## ⚖️ 2. Horizontal Pod Autoscaling (HPA)

### Metrics Server ต้องติดตั้งก่อน

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### HPA Config

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: spring-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: spring-app
  minReplicas: 2
  maxReplicas: 10
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

### KEDA (Kubernetes Event-driven Autoscaling)

```yaml
# keda-scaledobject.yaml — scale based on Kafka lag
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: kafka-consumer-deployment
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        consumerGroup: my-consumer-group
        topic: orders
        lagThreshold: "100"
```

---

## 🗄️ 3. StatefulSets สำหรับ Databases

### PostgreSQL StatefulSet

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: "postgres"
  replicas: 3  # Primary + 2 replicas
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "standard-ssd"
        resources:
          requests:
            storage: 50Gi

---
# headless service สำหรับ StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  clusterIP: None  # headless service
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

---

## ⛵ 4. Helm Charts

### โครงสร้าง Helm Chart

```
spring-app/
├── Chart.yaml
├── values.yaml
├── values-production.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── configmap.yaml
│   └── _helpers.tpl
└── charts/          # dependencies
```

### Chart.yaml

```yaml
apiVersion: v2
name: spring-app
description: A Spring Boot application Helm chart
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

### values.yaml

```yaml
replicaCount: 2

image:
  repository: registry.example.com/spring-app
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  host: app.example.com
  tls: true

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true
  auth:
    database: myapp
    existingSecret: postgres-secrets

config:
  logLevel: INFO
  jwtExpiration: 86400
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "spring-app.fullname" . }}
  labels:
    {{- include "spring-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "spring-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "spring-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: kubernetes
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel }}
```

### Deploy ด้วย Helm

```bash
# ติดตั้ง
helm install my-app ./spring-app -f values-production.yaml

# อัพเดต
helm upgrade my-app ./spring-app -f values-production.yaml

# Rollback
helm rollback my-app 1

# ลบ
helm uninstall my-app
```

---

## 🔄 5. GitOps กับ Argo CD

### ติดตั้ง Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# เข้าถึง Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Application Definition

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: spring-app-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/k8s-manifests
    targetRevision: main
    path: apps/spring-app/production
    helm:
      valueFiles:
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## 🌐 6. Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: spring-app-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: spring-app
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
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
    - to:  # DNS
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

---

## 📊 7. สรุปตาราง K8s Resources

| Resource | ใช้สำหรับ | Stateful? |
|---------|----------|-----------|
| Deployment | Stateless apps | ไม่ |
| StatefulSet | Databases, stateful apps | ใช่ |
| DaemonSet | Node-level agents | ไม่ |
| Job | Batch processing | ไม่ |
| CronJob | Scheduled tasks | ไม่ |
| ConfigMap | Non-sensitive config | ไม่ |
| Secret | Sensitive config | ไม่ |
| PersistentVolume | Storage | ใช่ |
| HPA | CPU/Memory autoscaling | ไม่ |
| KEDA | Event-driven autoscaling | ไม่ |

---

## 💡 Best Practices

1. **ใช้ resource limits/requests** เสมอเพื่อป้องกัน noisy neighbor
2. **Readiness/Liveness probes** สำคัญมากสำหรับ zero-downtime deploys
3. **Helm + GitOps** = repeatable, auditable deployments
4. **Network policies** เพื่อ micro-segmentation security
5. **Secret management** ใช้ Vault หรือ External Secrets Operator แทน plain Secrets

---

*Part 71/100+ | Kotlin & Spring Boot Complete Course*
