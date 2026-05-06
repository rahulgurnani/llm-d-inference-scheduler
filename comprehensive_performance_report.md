# Comprehensive Performance Report: EPP vs. Direct Service (20-40 RPS)

This report provides a consolidated analysis of the performance and reliability of the **Endpoint Picker (EPP)** with multimodal cache affinity versus **Direct Kubernetes LoadBalancer Service** access across varying load levels (20, 30, and 40 RPS).

---

## Infrastructure Details

The benchmarks were conducted on a GKE cluster with the following node configuration:

| Pool | Machine Type | CPU | Memory | Accelerator (per node) |
| :--- | :--- | :--- | :--- | :--- |
| **Model Server Pool** | **a3-highgpu-4g** | 104 vCPU | 1024 GB | **4x NVIDIA H100 GPUs** |
| **Default Pool** | e2-standard-16 | 16 vCPU | 64 GB | - |

**Deployment Topology:**
- **Model Servers:** 6 replicas of `Qwen2.5-VL-7B-Instruct` distributed across the A3 nodes.
- **Inference Gateway:** Deployed in the same cluster, utilizing the GKE Regional L7 Load Balancer.

---

## Executive Summary
Across all tested load levels, the EPP-managed traffic demonstrates **perfect reliability (100% success rate)** and **significantly lower latencies**. As the load increases, the performance gap between EPP and Direct access widens dramatically, particularly in tail latencies (p95), where EPP is up to **4.4x faster**.

The EPP acts as a critical stability layer, preventing backend saturation through intelligent multimodal cache-aware routing and queue-based load balancing.

---

## Unified Performance Comparison

| RPS | Metric | EPP (Affinity Enabled) | Direct Service | EPP Improvement |
| :--- | :--- | :--- | :--- | :--- |
| **20** | **Success Rate** | **100% (1200/1200)** | 99.9% (1199/1200) | **Zero Failures** |
| | **p50 Latency** | **0.080 s** | 0.179 s | 2.2x Faster |
| | **p95 Latency** | **0.605 s** | 0.816 s | 1.3x Faster |
| **30** | **Success Rate** | **100% (1800/1800)** | 99.9% (1799/1800) | **Zero Failures** |
| | **p50 Latency** | **0.074 s** | 0.236 s | 3.2x Faster |
| | **p95 Latency** | **0.363 s** | 1.018 s | 2.8x Faster |
| **40** | **Success Rate** | **100% (2400/2400)** | 94.4% (2265/2400) | **Zero Failures** |
| | **p50 Latency** | **0.089 s** | 0.304 s | 3.4x Faster |
| | **p95 Latency** | **0.764 s** | 3.338 s | 4.4x Faster |

---

## Detailed Latency Comparison (seconds)

| Percentile | RPS | EPP Latency (s) | Direct Latency (s) | Gap (s) |
| :--- | :--- | :--- | :--- | :--- |
| **p50 (Median)** | 20 | 0.080 | 0.179 | 0.099 |
| | 30 | 0.074 | 0.236 | 0.162 |
| | 40 | 0.089 | 0.304 | 0.215 |
| **p75** | 20 | 0.170 | 0.365 | 0.195 |
| | 30 | 0.138 | 0.473 | 0.335 |
| | 40 | 0.206 | 2.223 | 2.017 |
| **p95** | 20 | 0.605 | 0.816 | 0.211 |
| | 30 | 0.363 | 1.018 | 0.655 |
| | 40 | 0.764 | 3.338 | 2.574 |

---

## Root Cause Analysis: Why Direct Access Failed

Investigation into cluster logs and events revealed three primary reasons for the performance degradation and failure rates observed in direct service runs:

### 1. Model Server Resource Saturation
Direct access lack of intelligent routing caused frequent **Readiness Probe Failures**:
- **Observation:** Multiple pods reported `context deadline exceeded (Client.Timeout exceeded while awaiting headers)` for `/health` checks.
- **Impact:** Model server pods became so overwhelmed by unmanaged multimodal requests that they could not respond to K8s health checks, causing them to be temporarily removed from the service and resulting in connection drops for the client.

### 2. Lack of Multimodal Cache Affinity
- **EPP Advantage:** The EPP used the `mm-cache-affinity-scorer` to route requests with the same images to the same pods, reusing the expensive encoder cache.
- **Direct Access Penalty:** Standard K8s LoadBalancing distributed requests randomly. Every pod was forced to repeatedly re-encode the same images, wasting massive amounts of GPU/CPU cycles and causing the massive tail latency spikes (up to 3.3s at p95).

### 3. Cascading Connection Failures
- **Impact:** At higher loads (40 RPS), the failure rate reached 5.6%. As pods became unresponsive due to load (saturated CPU/Memory), the LoadBalancer attempted to shift traffic to remaining pods, quickly overwhelming them as well and leading to `Broken pipe` and `Server disconnected` errors.

---

## EPP Configuration Profile

The EPP was configured with a custom plugin profile designed to balance load distribution with multimodal cache efficiency. The following weights were used in the `default` scheduling profile:

| Plugin | Weight | Description |
| :--- | :--- | :--- |
| **`mm-cache-affinity-scorer`** | **4** | Prioritizes pods that already have the multimodal content (images) in their encoder cache. |
| **`queue-scorer`** | **4** | Distributes load based on the current request queue depth of each pod. |
| `kv-cache-utilization-scorer` | 0 | Disabled for this benchmark. |
| `prefix-cache-scorer` | 0 | Disabled for this benchmark. |

### Active Plugins (Snippet)
```yaml
plugins:
- type: multimodal-encoder-cache-data-producer
  parameters:
    cacheSize: 10000
- type: mm-cache-affinity-scorer
- type: queue-scorer
schedulingProfiles:
- name: default
  plugins:
  - pluginRef: mm-cache-affinity-scorer
    weight: 4
  - pluginRef: queue-scorer
    weight: 4
```

---

## Direct Kubernetes Service Configuration

For the direct access benchmarks, a standard Kubernetes `LoadBalancer` service was used to expose the model server pods. This configuration bypasses the Inference Gateway and EPP, relying on default GKE L4 load balancing.

### Service Manifest
```yaml
apiVersion: v1
kind: Service
metadata:
  name: vllm-qwen-vl-service
  namespace: default
spec:
  selector:
    app: vllm-qwen-vl-instruct
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
  type: LoadBalancer
```

---

## Conclusion
The Inference Gateway with EPP is not just an optimization tool; it is a **foundational reliability requirement** for multimodal workloads on GKE. 

By implementing **multimodal cache affinity**, the EPP:
1.  **Ensures Stability:** Maintains a 100% success rate even when the underlying pods are under high pressure.
2.  **Optimizes Compute:** Drastically reduces total compute load by maximizing encoder cache reuse.
3.  **Predictable Latency:** Keeps tail latencies (p95) within acceptable production bounds, whereas direct access leads to uncontrollable latency spikes.
