# 📋 Day-05 – Centralized Logging with EFK Stack

---

## 📚 Concepts Covered
- Learned the **second pillar of observability**: **Logging** (after Metrics from Days 1-4)
- Understood the importance of **centralized logging** in distributed systems
- Compared **EFK** vs **ELK** stack architectures
- Learned **FluentBit**, **Elasticsearch**, and **Kibana** components and their roles
- Understood log collection, storage, and visualization pipeline
- Explored **Kibana Query Language (KQL)** for log filtering and analysis

> ⚡️ Note: This is **theoretical understanding** with architecture diagrams. Practical demo requires Kubernetes cluster with persistent storage (EBS volumes in AWS).

---

## 🎯 Why Centralized Logging?

### Problem with Individual Pod Logs
```bash
# Traditional approach - checking logs pod by pod
kubectl logs pod-1 -n namespace-1
kubectl logs pod-2 -n namespace-2
# ... repeat for 100+ microservices
```

### Centralized Solution Benefits
- **Single Query**: Search across all 100+ services at once
- **Faster Debugging**: Quickly identify which services have issues
- **Security Events**: Rapidly find vulnerable services (e.g., Log4J vulnerability)
- **Correlation**: See how issues propagate across services

---

## 🏗️ EFK Stack Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   FluentBit     │───▶│  Elasticsearch  │───▶│     Kibana      │
│  (Log Forwarder)│    │   (Database)    │    │ (Visualization) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Component Breakdown

| Component | Role | Analogy to Metrics Stack |
|-----------|------|---------------------------|
| **FluentBit** | Log Forwarder | Node Exporter (collects data) |
| **Elasticsearch** | Database/Storage | Prometheus TSDB (stores data) |
| **Kibana** | Visualization/UI | Grafana (dashboards & queries) |

---

## 🛠️ Component Deep Dive

### 1. FluentBit (F) - Log Forwarder
```yaml
# Deployed as DaemonSet (one per node)
kind: DaemonSet
metadata:
  name: fluent-bit
spec:
  # Runs on every Kubernetes node
  # Reads logs from /var/log/**/*.log
  # Forwards to Elasticsearch
```

**Key Features:**
- **Lightweight**: Minimal resource usage
- **Vendor Neutral**: Can forward to Splunk, AWS CloudWatch, etc.
- **Real-time**: Streams logs as they're generated

### 2. Elasticsearch (E) - Database
```yaml
# StatefulSet with persistent storage
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        storageClassName: gp2  # AWS EBS
        resources:
          requests:
            storage: 100Gi
```

**Key Features:**
- **Persistent Storage**: Connected to EBS volumes for backup
- **Search Engine**: Fast full-text search across logs
- **Scalable**: Can handle multiple GB of logs daily

### 3. Kibana (K) - Visualization
```yaml
# Service exposed via LoadBalancer
kind: Service
metadata:
  name: kibana
spec:
  type: LoadBalancer
  ports:
    - port: 5601
```

**Key Features:**
- **Web UI**: Graphical dashboard for log analysis
- **KQL**: Kibana Query Language for filtering
- **Dashboards**: Similar to Grafana for metrics

---

## 🔄 EFK vs ELK Comparison

| Aspect | EFK (FluentBit) | ELK (Logstash) |
|--------|-----------------|----------------|
| **Resource Usage** | Lightweight | Resource-heavy |
| **Complexity** | Simple configuration | Advanced features |
| **Filtering** | Basic filtering | Rich data transformation |
| **Vendor Lock-in** | Vendor neutral | More Elasticsearch-focused |
| **Recommended For** | Starting point | Advanced use cases |

---

## 📊 Log Flow Architecture

```
┌─────────────┐    ┌──────────────┐    ┌─────────────┐    ┌─────────────┐
│    Pods     │───▶│  FluentBit   │───▶│Elasticsearch│───▶│   Kibana    │
│ (Generate   │    │ (DaemonSet)  │    │ (StatefulSet│    │ (Dashboard) │
│  Logs)      │    │              │    │  + EBS)     │    │             │
└─────────────┘    └──────────────┘    └─────────────┘    └─────────────┘
```

### Step-by-Step Flow:
1. **Applications** write logs to stdout/stderr
2. **Kubernetes** saves logs to `/var/log/containers/`
3. **FluentBit** reads logs from each node
4. **FluentBit** forwards logs to Elasticsearch
5. **Elasticsearch** indexes and stores logs
6. **Users** query logs through Kibana UI

---

## ⚙️ FluentBit Configuration Structure

```yaml
# fluent-bit-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        
    [OUTPUT]
        Name            es
        Match           *
        Host            elasticsearch
        Port            9200
        HTTP_User       elastic
        HTTP_Passwd     ${ELASTICSEARCH_PASSWORD}
        TLS             On
```

### Configuration Sections:
1. **SERVICE**: Overall FluentBit service settings
2. **INPUT**: Where logs come from (`/var/log/containers/`)
3. **FILTER**: Log processing (add Kubernetes metadata)
4. **OUTPUT**: Where to send logs (Elasticsearch)

---

## 🔐 Production Deployment Considerations

### 1. IAM Roles & CSI Driver (AWS)
```bash
# Required for EBS volume mounting
eksctl create iamserviceaccount \
  --name elasticsearch-service-account \
  --namespace logging \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEBSCSIDriverPolicy
```

### 2. Storage Classes
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp2-elasticsearch
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
  fsType: ext4
```

### 3. Security Configuration
```bash
# Retrieve Elasticsearch credentials
kubectl get secret elasticsearch-es-elastic-user \
  -o=jsonpath='{.data.elastic}' | base64 --decode
```

---

## 🔍 Sample Kibana Queries (KQL)

```bash
# Find errors in specific namespace
kubernetes.namespace_name:"production" AND level:"ERROR"

# Find logs from specific pod
kubernetes.pod_name:"my-app-*" AND message:*database*

# Find logs in time range
@timestamp:[now-1h TO now] AND kubernetes.container_name:"api"

# Find HTTP 5xx errors
response_code:[500 TO 599] AND kubernetes.namespace_name:"web"
```

---

## 📈 Logging vs Metrics vs Traces

| Observability Pillar | Purpose | Example |
|---------------------|---------|---------|
| **Metrics** (Prometheus) | What is happening? | CPU usage is 80% |
| **Logs** (EFK) | Why is it happening? | "Database connection timeout on line 5000" |
| **Traces** (Jaeger) | How to fix it? | Request flow across microservices |

---

## ✅ Day-05 Learnings Recap

By the end of Day-05, we understood:

- **Centralized logging** importance in distributed systems
- **EFK stack** components and their roles
- **FluentBit** configuration structure (Input, Filter, Output)
- **Elasticsearch** as a persistent log database
- **Kibana** for log visualization and querying
- **KQL syntax** for log filtering and analysis
- **Production considerations** (IAM roles, storage, security)

---

## 🚀 Next Steps

**Day-06**: Distributed Tracing with Jaeger & OpenTelemetry
- Learn the third pillar of observability
- Understand request flow across microservices
- Implement trace collection and visualization

**Future Demo**: 
- Deploy EFK stack on AWS EKS with proper IAM roles
- Configure FluentBit to collect application logs
- Create Kibana dashboards for log analysis
- Set up log-based alerts and monitoring

---

