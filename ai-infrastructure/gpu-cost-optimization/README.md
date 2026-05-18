# GPU Cost Optimization

Optimize GPU utilization and costs for AI/ML workloads across hyperscalers and NeoClouds.

## Why GPU Optimization Matters

GPU instances are among the most expensive cloud resources:
- **A100 (40GB)**: $3-4/hour on hyperscalers, $1-2/hour on NeoClouds
- **H100 (80GB)**: $5-7/hour on hyperscalers, $2-3/hour on NeoClouds
- **L40S**: $2-3/hour typical pricing

**Poor utilization = massive waste**. A GPU at 30% utilization is burning 70% of its cost.

## Key Metrics to Track

| Metric | Target | Tools | Notes |
|--------|--------|-------|-------|
| GPU Utilization | > 70% average | nvidia-smi, DCGM | Compute utilization, not memory |
| GPU Memory Usage | > 80% | nvidia-smi | Model + batch data fit |
| SM Active % | > 60% | Nsight Systems | Streaming multiprocessor activity |
| Tensor Core Utilization | > 50% | Nsight Compute | For mixed-precision workloads |
| PCIe Throughput | Near max | nvidia-smi topo -m | Multi-GPU communication |
| Power Draw | Appropriate for workload | nvidia-smi | Wasted power = wasted money |

## Optimization Strategies

### 1. GPU Sharing

**Multi-Instance GPU (MIG)** - NVIDIA A100/H100 feature

Split one physical GPU into up to 7 isolated instances:

```
A100 (40GB) can be partitioned as:
- 7 × 5GB instances (for inference)
- 4 × 10GB instances
- 3 × 20GB instances
- 2 × 30GB instances
- 1 × 40GB instance (full GPU)
```

**When to use MIG:**
- Multiple small inference workloads
- Development/testing environments
- Mixed workload types (training + inference)

**Example Configuration:**
```bash
# Enable MIG mode (requires reboot)
nvidia-smi -mig 1

# Create GPU instances
nvidia-smi mig -cgi 1,1,1,1,1,1,1  # 7x 5GB instances

# Launch containers with specific GPU instance
docker run --gpus '"device=GPU-uuid"' my-inference-app
```

**Savings:** Up to 70% for inference workloads

---

### 2. Time-Slicing (TSG)

Share GPU across multiple processes over time:

```yaml
# Kubernetes example with time-slicing
apiVersion: v1
kind: ConfigMap
metadata:
  name: gpu-time-slicing
  namespace: kube-system
data:
  config: |
    version: v1
    sharing:
      timeSlicing:
        resources:
          - name: nvidia.com/gpu
            replicas: 4  # Share across 4 pods
```

**Best for:**
- Development notebooks
- Low-throughput inference
- Batch jobs that don't need full GPU

**Savings:** 50-75% depending on workload count

---

### 3. Spot GPUs

Use preemptible/spot GPU instances for fault-tolerant workloads:

| Provider | Product | Discount | Interruption Rate |
|----------|---------|----------|-------------------|
| AWS | EC2 Spot | 60-70% | Variable |
| GCP | Preemptible | 60-75% | ~5-10% |
| Azure | Spot VMs | 60-80% | Variable |
| Lambda | Spot | 50-60% | Lower |
| CoreWeave | On-demand | 40-50% vs AWS | No spot needed |

**Checkpointing Strategy:**
```python
import torch
import os

def save_checkpoint(model, optimizer, epoch, path):
    torch.save({
        'epoch': epoch,
        'model_state_dict': model.state_dict(),
        'optimizer_state_dict': optimizer.state_dict(),
    }, path)

def load_checkpoint(path):
    checkpoint = torch.load(path)
    return checkpoint['model_state_dict'], checkpoint['optimizer_state_dict']

# Save every N minutes
EPOCHS_PER_CHECKPOINT = 1
CHECKPOINT_DIR = '/mnt/checkpoints'

for epoch in range(total_epochs):
    train_one_epoch()
    
    if epoch % EPOCHS_PER_CHECKPOINT == 0:
        save_checkpoint(model, optimizer, epoch, 
                       f'{CHECKPOINT_DIR}/ckpt_epoch_{epoch}.pt')
```

**Savings:** 60-70% for training workloads

---

### 4. Batch Inference Optimization

Maximize throughput per GPU dollar:

```python
# Poor: One request at a time
def inference_single(request):
    model.forward(request)  # GPU sits idle most of the time

# Better: Dynamic batching
def inference_batched(requests, max_batch_size=32, max_wait_ms=50):
    batch = collect_requests(max_batch_size, max_wait_ms)
    results = model.forward(batch)
    return distribute_results(results)
```

**Key Parameters:**
- `max_batch_size`: Larger = better throughput, higher latency
- `max_wait_ms`: How long to wait for batch to fill
- `min_batch_size`: Minimum before processing (avoid starvation)

**Tools for Batching:**
- **NVIDIA Triton Inference Server**: Dynamic batching built-in
- **TensorRT**: Optimized inference with batching
- **vLLM**: High-throughput LLM serving with PagedAttention
- **TGI (Text Generation Inference)**: HuggingFace's optimized LLM server

**Triton Configuration Example:**
```protobuf
dynamic_batching {
  max_batch_size: 32
  batch_timeout_microseconds: 50000
  preferred_batch_size: [ 16, 32 ]
}
```

**Savings:** 3-5x throughput improvement = 60-80% cost reduction per inference

---

### 5. Model Optimization

Reduce compute requirements through model optimization:

#### Quantization
```python
# FP32 → INT8 = 4x smaller, 2-3x faster
from transformers import AutoModelForCausalLM
import torch

# Load model
model = AutoModelForCausalLM.from_pretrained("llama-2-7b")

# Quantize to INT8
model_int8 = torch.quantization.quantize_dynamic(
    model, {torch.nn.Linear}, dtype=torch.qint8
)

# Or use bitsandbytes for LLMs
import bitsandbytes as bnb
model_4bit = AutoModelForCausalLM.from_pretrained(
    "llama-2-7b",
    load_in_4bit=True,
    device_map="auto"
)
```

#### Distillation
```python
# Train smaller student model from larger teacher
from transformers import Trainer

student_model = AutoModelForCausalLM.from_pretrained("tiny-llama-1.1b")
teacher_model = AutoModelForCausalLM.from_pretrained("llama-2-7b")

# Distillation training loop
# Student learns to mimic teacher outputs
# Result: 10x smaller model with ~90% accuracy
```

#### Pruning
```python
# Remove unused weights
from transformers import BertForSequenceClassification

model = BertForSequenceClassification.from_pretrained("bert-base-uncased")

# Structured pruning (remove entire neurons)
pruned_model = prune_model_structured(model, sparsity=0.3)
# 30% smaller, minimal accuracy loss
```

**Savings by Technique:**

| Technique | Size Reduction | Speed Improvement | Accuracy Loss |
|-----------|---------------|-------------------|---------------|
| INT8 Quantization | 4x | 2-3x | < 1% |
| INT4 Quantization | 8x | 3-4x | 1-3% |
| Distillation (10x) | 10x | 5-10x | 3-5% |
| Pruning (50%) | 2x | 1.5-2x | 1-2% |

---

### 6. Right-Size GPU Selection

Match GPU to workload requirements:

#### Training Workloads

| Workload Size | Recommended GPU | Why |
|--------------|-----------------|-----|
| Small models (< 1B params) | L4, A10 | Cost-effective, good memory |
| Medium models (1-10B) | A100 (40GB) | Balance of memory and speed |
| Large models (10-70B) | A100 (80GB) / H100 | Need memory capacity |
| Very large (> 70B) | H100 (80GB) × 8+ | Need NVLink for parallelism |

#### Inference Workloads

| Latency Requirement | Recommended GPU | Why |
|--------------------|-----------------|-----|
| Real-time (< 50ms) | L4, A10G | Low latency, good for single requests |
| Batch (< 500ms) | A100, H100 | High throughput with batching |
| Offline (no urgency) | Spot GPUs, MIG | Maximize cost efficiency |

**Cost Comparison (per hour):**

```
AWS Pricing (us-east-1):
- g6.xlarge (L4):           $0.54/hr
- g5.xlarge (A10G):         $1.01/hr
- p4d.24xlarge (A100 × 8): $32.77/hr ($4.10/GPU)
- p5.48xlarge (H100 × 8):  $98.34/hr ($12.29/GPU)

NeoCloud Pricing:
- Lambda (A100):            ~$1.50/hr
- CoreWeave (H100):         ~$2.50/hr
- Nebius (H100):            ~$2.00/hr

Savings: 50-80% using NeoClouds for GPU workloads
```

---

### 7. Multi-GPU Optimization

For distributed training and large inference:

#### Communication Optimization
```bash
# Check GPU topology
nvidia-smi topo -m

# Optimal: All GPUs connected via NVLink
# Suboptimal: GPUs crossing PCIe switches

# Set NCCL environment variables for better performance
export NCCL_ALGO=Ring
export NCCL_NET_GDR_LEVEL=3
export CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7
```

#### Gradient Accumulation
```python
# Instead of large batch on one GPU, accumulate gradients
accumulation_steps = 4

for i, batch in enumerate(dataloader):
    outputs = model(batch)
    loss = criterion(outputs, labels)
    loss = loss / accumulation_steps  # Normalize loss
    loss.backward()
    
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

**Benefits:**
- Effective batch size without memory pressure
- Can use smaller/cheaper GPUs
- Better gradient stability

---

## Platform-Specific Optimizations

### AWS

**Use Graviton + GPU combination:**
```yaml
# Run data preprocessing on Graviton (cheap)
# Use GPU only for model inference
Resources:
  PreprocessingFleet:
    InstanceType: c7g.xlarge  # Graviton3, $0.14/hr
  InferenceFleet:
    InstanceType: g6.xlarge   # L4 GPU, $0.54/hr
```

**Inferentia for Inference:**
```bash
# AWS Inferentia2 chips for LLM inference
# Up to 4x better price-performance than GPUs
inf2.xlarge: $0.66/hr (supports models up to 20B params)
inf2.8xlarge: $5.30/hr (supports models up to 175B params)
```

### GCP

**TPU vs GPU Decision:**
```
Use TPU when:
- Training large models (faster time-to-solution)
- Batch inference workloads
- Models supported by JAX/PyTorch XLA

Use GPU when:
- Low-latency inference
- Custom operations not supported on TPU
- Smaller models where TPU advantage is minimal
```

**GKE GPU Sharing:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-shared
spec:
  containers:
  - name: app
    image: my-gpu-app
    resources:
      limits:
        nvidia.com/gpu: 1  # Full GPU
        nvidia.com/gpu-memory: 8Gi  # Or specify memory only
```

### Azure

**ND-series Optimization:**
```bash
# ND A100 v4 series optimized for AI
# Use InfiniBand for multi-node training
az vm list-skus --location eastus --size Standard_ND96asr_v4

# Combine with Azure ML for automatic optimization
az ml compute create --name gpu-cluster --type AmlCompute \
  --vm-size Standard_ND96asr_v4 --min-instances 0 --max-instances 10
```

---

## Monitoring & Alerting

### Prometheus + Grafana Setup

```yaml
# prometheus-rules.yaml
groups:
- name: gpu-alerts
  rules:
  - alert: LowGPUUtilization
    expr: avg(DCGM_FI_DEV_GPU_UTIL) < 30
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "GPU utilization below 30%"
      description: "GPU {{ $labels.gpu }} averaging {{ $value }}% utilization"
  
  - alert: HighGPUMemory
    expr: avg(DCGM_FI_DEV_MEM_COPY_UTIL) > 90
    for: 15m
    labels:
      severity: critical
    annotations:
      summary: "GPU memory bandwidth saturated"
```

### Cost Dashboards

Track these metrics daily:
- **Cost per inference**: Total GPU cost / number of inferences
- **Cost per training step**: GPU hours / training steps completed
- **GPU utilization trend**: Are we getting more efficient?
- **Spot interruption rate**: How often are we losing spot instances?

---

## Real-World Case Studies

### Case Study 1: LLM Inference Startup

**Problem:** Running Llama-2-70B inference on AWS p4d instances
- 8 × A100 GPUs @ $4.10/hr each = $32.80/hr
- Utilization: 35% average
- Monthly cost: $23,616

**Optimizations Applied:**
1. Migrated to CoreWeave: $2.00/hr per H100
2. Implemented vLLM with continuous batching
3. Quantized to INT8 (minimal quality loss)
4. Added request coalescing

**Results:**
- New cost: $16.00/hr (8 × H100 @ $2.00)
- Utilization: 78% average
- Throughput: 4.2x higher
- Monthly cost: $11,520
- **Savings: $12,096/month (51%)**

---

### Case Study 2: Computer Vision Training

**Problem:** Training object detection models on Azure
- Using NCASv4 (A100 × 8)
- Long training times, frequent out-of-memory
- Monthly cost: $18,000

**Optimizations Applied:**
1. Switched to gradient accumulation (smaller effective batch per GPU)
2. Used mixed precision (AMP)
3. Implemented checkpointing + spot instances
4. Right-sized to 4 × A100 instead of 8

**Results:**
- Training time: Same (better utilization compensated for fewer GPUs)
- Spot savings: 65% discount
- GPU count: 8 → 4
- Monthly cost: $6,300
- **Savings: $11,700/month (65%)**

---

## Quick Wins Checklist

Start here for immediate impact:

- [ ] **Enable GPU monitoring** (nvidia-smi, DCGM exporter)
- [ ] **Identify underutilized GPUs** (< 50% average utilization)
- [ ] **Implement dynamic batching** for inference workloads
- [ ] **Test quantization** on your models (INT8/INT4)
- [ ] **Evaluate NeoCloud providers** for GPU workloads
- [ ] **Set up spot/preemptible instances** for training jobs
- [ ] **Configure autoscaling** to scale GPUs to zero when idle
- [ ] **Review GPU family selection** (L4 vs A10 vs A100 vs H100)
- [ ] **Implement checkpointing** for fault tolerance
- [ ] **Create cost-per-inference metric** and track weekly

---

## Tools & Resources

### Monitoring
- **NVIDIA DCGM**: Data Center GPU Manager
- **NVIDIA Nsight**: Profiling and debugging
- **Prometheus + DCGM Exporter**: Kubernetes GPU monitoring
- **Grafana GPU Dashboards**: Visualization

### Optimization
- **NVIDIA Triton**: Inference server with batching
- **vLLM**: High-throughput LLM serving
- **TensorRT**: Model optimization and deployment
- **DeepSpeed**: Distributed training optimization
- **bitsandbytes**: LLM quantization

### Cost Management
- **Kubecost/OpenCost**: Kubernetes cost allocation with GPU support
- **CloudHealth**: Multi-cloud cost management
- **Infracost**: Infrastructure cost estimation

---

## See Also

- [LLM Serving Economics](../../neoclouds/economics/llm-serving-costs.md)
- [GPU Scheduling in Kubernetes](../../kubernetes/gpu-sharing/)
- [AI Infrastructure Observability](../observability/)
- [NeoCloud GPU Providers](../../neoclouds/gpu-clouds/)
- [Spot Instance Strategies](../../aws/compute/spot-instances.md)
