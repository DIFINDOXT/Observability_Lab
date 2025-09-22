# 🔍 Day-06 – Distributed Tracing with Jaeger & OpenTelemetry

---

## 📚 Concepts Covered
- Learned the **third pillar of observability**: **Distributed Tracing** (completing the trilogy after Metrics and Logs)
- Understood **trace instrumentation** vs **tool deployment** responsibilities
- Learned **Jaeger architecture** and components (Agent, Collector, Storage, UI)
- Explored **OpenTelemetry** as the vendor-neutral instrumentation standard
- Understood **spans** and **traces** for request flow analysis
- Compared tracing with real-world travel itinerary analogy

> ⚡️ Note: This is **theoretical understanding** with architecture focus. Practical demo requires Kubernetes cluster with Elasticsearch storage.

---

## 🎯 What is Distributed Tracing?

### Real-World Analogy: Travel Itinerary
```
Hyderabad → Dubai (2h) → London (7h) → Boston (8h) → Hotel (2.5h) = 19.5h
Expected: 17h | Actual: 19.5h | Delay: 2.5h in Boston cab ride
```

### Microservices Analogy: Request Flow
```
User → Login Service → Service A → Service B → Payment Service
Expected: 1s | Actual: 4s | Need to find which service caused 3s delay
```

**Tracing Purpose**: Identify exactly where latency occurs in distributed systems

---

## 🏗️ Complete Observability Stack

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   METRICS   │  │    LOGS     │  │   TRACES    │
│ (Prometheus)│  │    (EFK)    │  │  (Jaeger)   │
└─────────────┘  └─────────────┘  └─────────────┘
       │                │                │
       ▼                ▼                ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Grafana   │  │   Kibana    │  │ Jaeger UI   │
│(Dashboards) │  │(Log Search) │  │(Trace View) │
└─────────────┘  └─────────────┘  └─────────────┘
```

### Three Pillars Explained:
| Pillar | Question | Example |
|--------|----------|---------|
| **Metrics** | What is happening? | "CPU usage is 80%" |
| **Logs** | Why is it happening? | "Database connection timeout on line 5000" |
| **Traces** | How to fix it? | "Delay is in Service B → Payment Service call" |

---

## 🛠️ Jaeger Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│Application  │───▶│Jaeger Agent │───▶│   Jaeger    │───▶│   Jaeger    │
│(Instrumented│    │  (Collector)│    │  Collector  │    │     UI      │
│with OTel)   │    │             │    │             │    │             │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                             │
                                             ▼
                                    ┌─────────────┐
                                    │ Elasticsearch│
                                    │ (Storage DB) │
                                    └─────────────┘
```

### Jaeger Components:

| Component | Role | Description |
|-----------|------|-------------|
| **Agent** | Collector | Receives traces from instrumented applications |
| **Collector** | Processor | Processes and stores traces in database |
| **Storage** | Database | Stores traces (Elasticsearch or Cassandra) |
| **UI** | Query Interface | Web interface for viewing and analyzing traces |

---

## 🔧 Implementation Responsibilities

### 1. Developer Responsibility: Instrumentation
```javascript
// Example: OpenTelemetry instrumentation in Node.js
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');

const jaegerExporter = new JaegerExporter({
  endpoint: 'http://jaeger-collector:14268/api/traces',
});

const sdk = new NodeSDK({
  traceExporter: jaegerExporter,
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();
```

### 2. DevOps/SRE Responsibility: Tool Deployment
```yaml
# Jaeger deployment on Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger-collector
spec:
  template:
    spec:
      containers:
      - name: jaeger-collector
        image: jaegertracing/jaeger-collector:latest
        env:
        - name: SPAN_STORAGE_TYPE
          value: elasticsearch
        - name: ES_SERVER_URLS
          value: http://elasticsearch:9200
```

---

## 📊 Spans and Traces Concept

### Single Request Trace Example:
```
Trace ID: abc123 (User request to /payment)
├─ Span 1: HTTP Request (2ms)
├─ Span 2: Authentication (5ms)
├─ Span 3: Database Query (50ms) ← Bottleneck found!
├─ Span 4: Payment Processing (10ms)
└─ Span 5: HTTP Response (1ms)
Total: 68ms
```

### Multi-Service Trace Example:
```
Service A → Service B → Payment Service
├─ Service A spans:
│  ├─ Express middleware (1ms)
│  ├─ Business logic (3ms)
│  └─ HTTP call to Service B (45ms) ← Network delay
├─ Service B spans:
│  ├─ Receive request (1ms)
│  ├─ Process data (2ms)
│  └─ Call Payment Service (15ms)
└─ Payment Service spans:
   ├─ Validate payment (5ms)
   └─ Database write (8ms)
```

---

## 🔬 OpenTelemetry Integration

### Instrumentation Levels:
```python
# Automatic instrumentation (framework level)
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

FlaskInstrumentor().instrument()
RequestsInstrumentor().instrument()

# Manual instrumentation (function level)
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

def process_payment():
    with tracer.start_as_current_span("payment_processing"):
        # Payment logic here
        with tracer.start_as_current_span("database_write"):
            # Database operation
            pass
```

### Trace Context Propagation:
```bash
# HTTP headers automatically propagated between services
traceparent: 00-abc123def456-789ghi012jkl-01
tracestate: vendor1=value1,vendor2=value2
```

---

## 🖥️ Jaeger UI Features

### 1. Service Map View
```
[User] → [Login Service] → [Service A] → [Service B] → [Payment]
  1ms      5ms               45ms         2ms          15ms
                              ↑
                      Bottleneck identified
```

### 2. Trace Timeline View
```
Service A |████████████████████████████████████████████████| 68ms
Service B |                              ██████████████████| 17ms
Payment   |                                        ████████| 15ms
          0ms    10ms    20ms    30ms    40ms    50ms    60ms
```

### 3. Span Details
```
Span: HTTP Request to Service B
Duration: 45ms
Tags:
  - http.method: POST
  - http.url: /api/process
  - http.status_code: 200
Logs:
  - 12:34:56.789 Request started
  - 12:34:56.834 Response received
```

---

## 🏭 Production Deployment Architecture

### Kubernetes Deployment:
```yaml
# Complete observability stack
apiVersion: v1
kind: Namespace
metadata:
  name: observability
---
# Elasticsearch (shared storage)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
  namespace: observability
---
# Jaeger components
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger-collector
  namespace: observability
---
# Jaeger Agent (DaemonSet)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: jaeger-agent
  namespace: observability
```

### Storage Configuration:
```yaml
# Persistent storage for traces
volumeClaimTemplates:
- metadata:
    name: elasticsearch-data
  spec:
    storageClassName: gp2
    resources:
      requests:
        storage: 100Gi
```

---

## 🔍 Debugging with Jaeger

### Common Use Cases:

1. **Latency Investigation**
   - Query: Service = "payment-service", Duration > 1s
   - Find: Which spans take longest time

2. **Error Tracking**
   - Query: Service = "api-gateway", Tags = error:true
   - Find: Where errors originate in request chain

3. **Dependency Analysis**
   - Query: Service = "user-service"
   - Find: Which services user-service depends on

4. **Performance Optimization**
   - Query: Operation = "database-query"
   - Find: Slowest database operations across services

---

## ✅ Day-06 Learnings Recap

By the end of Day-06, we understood:

- **Distributed tracing** as the third observability pillar
- **Jaeger architecture** (Agent → Collector → Storage → UI)
- **OpenTelemetry** for vendor-neutral instrumentation
- **Spans and traces** for request flow visualization
- **Developer vs DevOps** responsibilities in tracing
- **Integration** with complete observability stack
- **Production deployment** considerations

---
