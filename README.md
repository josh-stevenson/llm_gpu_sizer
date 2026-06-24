# vLLM Kubernetes Cluster Sizer & Profiler ☸️

## Overview
The **vLLM K8s Cluster Sizer & Profiler** is a capacity planning tool designed for Platform Architects and AI/ML Engineers deploying Large Language Models (LLMs) on Kubernetes. 

While basic VRAM calculators only look at model weights, deploying LLMs in production requires navigating a complex web of hardware topologies, dynamic memory growth, and Kubernetes scheduling constraints. This tool bridges the gap by simulating how the **NVIDIA GPU Operator** allocates hardware (Passthrough, MIG, vGPU) and validating whether your multi-model workload will physically fit and perform efficiently on your cluster.

It runs entirely in your browser using WebAssembly (`stlite`), requiring no backend servers or Python environments to host.

---

## Features
* **Multi-Model Support:** Plan capacity for up to 4 concurrent endpoints running different models.
* **K8s Scheduling Diagnostics:** Validates node affinity, Tensor Parallelism (TP) boundaries, and hardware interconnect limits.
* **GPU Operator Simulation:** Accurately models the VRAM boundaries and capabilities of GPU Passthrough, Multi-Instance GPU (MIG), and vGPU (Time-Slicing).
* **Latency Profiling:** Estimates Time-To-First-Token (TTFT), Inter-Token Latency (ITL), and End-to-End (E2E) Response Time based on your hardware's peak memory bandwidth.

---

## How to Use the Tool

### 1. Cluster Topology (Global Settings)
Define the physical reality of your Kubernetes cluster and how the GPU Operator exposes it:
* **Worker Nodes & GPUs per Node:** Defines the physical layout of your hardware. This is critical because a model's Tensor Parallelism group must fit on a single node to utilize high-speed NVLink.
* **GPU Architecture:** Select your specific hardware (e.g., H100, RTX Pro 6000) to set the baseline VRAM and memory bandwidth limits.
* **GPU Allocation Strategy:**
  * **GPU Passthrough:** Pods get exclusive access to full, physical GPUs.
  * **MIG (Multi-Instance GPU):** Physically partitions GPUs into isolated hardware slices. *(Note: MIG slices lack NVLink, so Tensor Parallelism > 1 is unsupported).*
  * **vGPU (Time-Slicing):** Uses software-level time-slicing to share a GPU. You define the number of "Replicas," which multiplies your schedulable GPUs but proportionally divides the VRAM limit per virtual GPU.

### 2. Model Configurations (Tabs)
For each active model you intend to host, configure its specific profile:
* **Model Specs:** Enter the parameters (e.g., 8B, 70B) and weight precision (FP16, FP8, INT4).
* **Workload & I/O:** * Define your average **Input Tokens (Prompt)** and **Output Tokens (Generation)**.
  * Set the **Concurrency** (peak simultaneous users).
  * Adjust the **Assumed MBU (Model Bandwidth Utilization)** to estimate how efficiently your batch size utilizes the GPU's memory bandwidth.
* **K8s Scheduling:** Assign a specific **Tensor Parallelism** degree (the number of GPUs to shard this model across).

### 3. Verdict & Diagnostics
At the bottom of the app, the engine aggregates your configuration and runs a placement simulation:
**Production Ready:** All models fit within VRAM limits, TP boundaries, and cluster topology.
**Deployment Risky:** Models will schedule, but are dangerously close to hardware limits (e.g., VRAM usage > 90%), risking Out-of-Memory (OOM) crashes under load, or using highly inefficient software topologies.
 **Deployment Failed:** The Kubernetes scheduler will fail to place the pods. The engine will list specific constraints violated (e.g., Anti-Affinity Failures, MIG Conflicts, or hard VRAM limits).

---

## Mathematical Foundations

This tool uses industry-standard performance engineering formulas to calculate memory footprints and latency:

### Memory (VRAM) Calculations
* **Static Weights:** Calculated based on parameter count and quantization level. 
  * *Formula:* `Parameters * Bytes per Parameter`
* **Dynamic KV Cache:** The memory required to store key and value tensors from past tokens to avoid redundant calculation.
  * *Formula:* $M_{KV} = 2 \times C_{users} \times T_{seq} \times L_{layers} \times \frac{N_{attn\_heads}}{g} \times D_{head} \times Z_{bytes}$.
* **Engine Overhead:** A conservative baseline of ~4 GB per GPU is reserved for CUDA context, the vLLM runtime, and PyTorch activation buffers.

### Latency Profiling
* **Inter-Token Latency (ITL):** Because the decoding phase is heavily memory-bandwidth bound, the speed at which subsequent tokens are generated depends on the total weight and cache size divided by the effective Model Bandwidth Utilization (MBU).
* **Time-to-First-Token (TTFT):** The prefill phase is highly parallelized and compute-bound. The tool uses an empirical linear regression model to estimate prefill latency based on the total input prompt size.
* **End-to-End Latency (E2E):** The total round trip, calculated as $TTFT + (ITL \times (\text{Output Tokens} - 1))$.

---

