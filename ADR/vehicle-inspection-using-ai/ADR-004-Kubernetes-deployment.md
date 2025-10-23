# ADR 004: Kubernetes Orchestration for AI Inference and Data Pipeline

**Status:** Proposed  
**Date:** 2025-10-22  
**Deciders:** DevOps Lead, Architecture Team  
**Related Issue/Story:** AI-powered vehicle return inspection system

## Context and Problem Statement

The vehicle inspection system requires orchestration of multiple containerized services:

1. **Image upload service:** Receives photos from edge devices (cameras + Jetson/Coral board)
2. **Image processing pipeline:** Resize, crop, quality checks
3. **AI inference pods:** YOLOv8 damage detection, license plate OCR
4. **LLM service pods:** Self-hosted LLaMA for report generation
5. **Database:** PostgreSQL with persistent state
6. **Caching:** Redis for performance
7. **External API calls:** Fallback to GPT-4o for edge cases

The system must handle:
- Real-time image uploads from 5,000 rental locations across European zones
- Scalable AI inference (variable load: 50-500 images per hour depending on peak times)
- EU data residency and compliance requirements
- Reliable state management for inspection history
- Monitoring and alerting for infrastructure issues

**Question:** Should we use Kubernetes for orchestration, and if so, what resource allocation and scaling strategy?

**Constraints:**
- Runs on-premises or EU cloud region (Azure EU, Hetzner, OVH)
- Limited budget; cannot provision excessive resources
- Team expertise with Linux and containerization (DevOps background)
- Need < 500ms inference latency for 90% of requests

## Decision Factors

- **Scalability:** Kubernetes auto-scales based on CPU/memory; handles traffic spikes
- **Resource Efficiency:** Kubernetes bin-packing optimizes hardware utilization
- **Resilience:** Pod restart policies, health checks, multi-replica services
- **State Management:** StatefulSets for databases; careful volume management needed
- **Monitoring:** Prometheus, Loki, Grafana integration for observability
- **Operational Overhead:** Kubernetes cluster requires 2-3 DevOps engineers for production
- **Cost:** Kubernetes infrastructure overhead vs. simpler Docker Swarm or manual VMs
- **Data Residency:** Kubernetes deployments on EU infrastructure ensure compliance

## Alternatives

### Option A: Kubernetes (Full Orchestration)
- **Pros:**
  - Industry standard for production microservices
  - Native auto-scaling for inference pods based on queue depth or CPU
  - Service mesh capabilities (Istio) for advanced traffic management
  - Excellent monitoring ecosystem (Prometheus, Grafana)
  - Multi-region deployment patterns for EU resilience
  - Self-healing (pod restart, node recovery)
  - GitOps deployment workflows (ArgoCD)
- **Cons:**
  - Operational complexity (cluster setup, security, upgrades)
  - Steep learning curve for new team members
  - Overkill for single-region, moderate-scale workload
  - Requires dedicated DevOps/SRE resources

### Option B: Docker Swarm (Simpler Orchestration)
- **Pros:**
  - Simpler than Kubernetes; built into Docker ecosystem
  - Faster deployment and cluster setup
  - Lower operational overhead
  - Works well for teams with Docker expertise
- **Cons:**
  - Limited auto-scaling capabilities
  - Smaller ecosystem and fewer managed services
  - Harder to manage stateful services (databases)
  - Active development ceased; Docker focusing on Kubernetes
  - Less suitable for high-availability production systems

### Option C: Serverless (AWS Lambda, Azure Functions)
- **Pros:**
  - Automatic scaling; pay only for compute used
  - Minimal operational overhead
  - Good for bursty workloads (inspection submission)
- **Cons:**
  - **Critical:** Cold-start latency (500ms-2s) violates < 500ms inference requirement
  - Difficult for GPU workloads (Lambda has limited GPU support)
  - Long-running LLM inference not ideal for serverless
  - Vendor lock-in to cloud provider
  - Data residency concerns if using AWS US regions

### Option D: Manual VMs with Docker and Load Balancing
- **Pros:**
  - Full control over infrastructure
  - Simpler mental model; easier to debug
  - Lower overhead than Kubernetes
- **Cons:**
  - No automatic scaling or self-healing
  - Requires custom monitoring and alerting
  - Manual deployment and updates
  - Error-prone; high operational toil
  - Not suitable for resilient 24/7 service

## Decision Outcome

**We will adopt Kubernetes (Option A) with the following configuration:**

### Cluster Architecture

```yaml
# High-level structure
Namespace: inspection-system
├── Data Tier (Stateful)
│   ├── PostgreSQL (1 primary + 1 replica)
│   └── Redis (single node, no replication needed for cache)
├── Processing Tier (Stateless, Auto-scaled)
│   ├── Image Upload Service (3 replicas, HPA: 2-10)
│   ├── Image Processing Pipeline (2-20 replicas, HPA based on queue depth)
│   ├── YOLOv8 Inference Pods (2-8 GPU pods, HPA: CPU > 70%)
│   ├── LLaMA LLM Service (1-2 GPU pods, fixed allocation)
│   └── GPT-4o Fallback Service (stateless, 1-3 replicas)
└── Observability Tier
    ├── Prometheus (metrics collection)
    ├── Grafana (dashboards)
    └── Loki (log aggregation)
```

### Resource Allocation

**GPU Nodes (for inference):**
- YOLOv8 pods: 1x A100 GPU or 2x RTX 4090 (time-shared)
- LLaMA service: 1x A100 GPU or 2x RTX 4090
- Inference latency target: 100-150ms per image

**CPU Nodes:**
- Image upload service: 4 vCPU, 8GB RAM
- Image processing: 8 vCPU, 16GB RAM
- API services: 4 vCPU, 8GB RAM
- Database (PostgreSQL): 8 vCPU, 32GB RAM, 500GB SSD persistent volume

**High Availability:**
- PostgreSQL: 1 primary + 1 read replica (streaming replication)
- Redis: Single instance (cache loss acceptable; recreated on restart)
- Inference services: 2-3 replicas minimum

### Auto-Scaling Strategy

```yaml
# Horizontal Pod Autoscaler for inference
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: yolov8-inference-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: yolov8-inference
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
```

### Data Residency & Compliance

- All pods deployed in EU region (Azure EU West or on-premises in Europe)
- Persistent volumes stored on EU infrastructure (GDPR compliance)
- Network policies restrict cross-zone data flow
- Secrets stored in Kubernetes Secrets or external vault (sealed-secrets or HashiCorp Vault)

### Monitoring & Alerting

```yaml
# Example Prometheus alert for SLA breaches
- alert: HighInferenceLatency
  expr: histogram_quantile(0.95, rate(inference_duration_seconds_bucket[5m])) > 0.5
  for: 5m
  annotations:
    summary: "95th percentile inference latency exceeded 500ms"
```

### Deployment Workflow

- **GitOps:** ArgoCD manages deployments from git repository
- **CI/CD:** GitHub Actions / GitLab CI builds container images, pushes to registry
- **Rollout:** Blue-green or canary deployments for zero-downtime updates

### Rationale for Kubernetes

1. **Scalability:** Auto-scales inference pods during peak rental return times (morning, evening)
2. **Resilience:** Self-healing pods, health checks, multi-replica services
3. **Operational Excellence:** DevOps team expertise; proven patterns for production ML workloads
4. **Cost Efficiency:** Bin-packing and auto-scale reduce idle resource costs
5. **Monitoring:** Rich observability ecosystem (Prometheus, Grafana, Loki)
6. **Compliance:** Full control over EU data residency and infrastructure
7. **Future Growth:** Scales to multi-region deployments if rental fleet expands

## Implementation Roadmap

- **Phase 1:** Deploy single-node K8s cluster (testing), Minikube or K3s
- **Phase 2:** Production cluster with 3 control planes, 5-7 worker nodes (3 CPU, 2 GPU, 2 storage)
- **Phase 3:** Add monitoring (Prometheus, Grafana), logging (Loki), and tracing
- **Phase 4:** Implement auto-scaling, health checks, and SLO-based alerting

## Links / References

- ADR-001: LLM Model Selection (LLaMA pod resource requirements)
- ADR-003: Damage Detection Model (YOLOv8 pod GPU allocation)
- ADR-005: Image Processing Pipeline (worker pod configuration)
- External: Kubernetes documentation, CNCF best practices, PersistentVolume patterns
- Tools: Helm for deployment, kustomize for configuration management, ArgoCD for GitOps
