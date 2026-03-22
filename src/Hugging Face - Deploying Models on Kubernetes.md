
Kubernetes cluster running ML workloads is a two-tier system. You have "thinking nodes" (CPU) that handle everything normal — your API servers, databases, queue workers, routing. And you have "crunching nodes" (GPU) that do one thing: tensor math, very fast.


# CPU vs GPU Inferencing: Why It Matters for ML on Kubernetes

Let me build this from the ground up — starting with what's physically happening inside the chips, then connecting it to your K8s deployment decisions.

## The Core Difference

The fundamental thing to understand: inference is mostly **matrix multiplication**. When a model processes your input text, it's doing thousands of multiply-and-add operations across weight matrices. The question is _how_ those operations get scheduled on hardware.

Think of it like a post office analogy. A CPU is like having 8 very skilled postal workers who can each handle any type of package — fragile, oversized, international — but they process them one at a time, sequentially. A GPU is like having 10,000 postal workers who can only do one simple thing — stamp a package — but they all stamp simultaneously. If your job is "stamp 10,000 packages," the GPU finishes in one beat while the CPU takes 1,250 beats.

Neural network inference is overwhelmingly the "stamp 10,000 packages" kind of work.

![](Pasted%20image%2020260321184941.png)

The diagram above shows the architectural difference, but let me explain what's actually happening during inference at each level.

## What Happens During a Single Forward Pass

When your FastAPI server receives `{"text": "I love this product"}` and runs `classifier(text)`, here's the compute pipeline:

**Step 1 — Tokenization** (CPU-only, fast): The text gets split into token IDs: `[101, 1045, 2293, 2023, 4031, 102]`. This is string manipulation — CPU is perfectly fine here, takes microseconds.

**Step 2 — Embedding lookup** (memory-bound): Each token ID indexes into a 30522 x 768 matrix to pull out a 768-dimensional vector. This is a table lookup, not computation — it's limited by how fast you can read from memory.

**Step 3 — Transformer layers** (compute-bound, this is where GPU wins): Each layer does attention (Q, K, V matrix multiplications) and feed-forward (two large matrix multiplications). For distilbert with 6 layers, that's roughly 12 matrix multiplications of shape ~(seq_len, 768) x (768, 768). This is the bulk of the work.

**Step 4 — Classification head** (small, fast): A final linear layer maps the 768-dim representation to 2 classes (positive/negative). Trivial.

The key insight: Step 3 is ~95% of the total compute time. And matrix multiplication is _embarrassingly parallel_ — every element of the output matrix can be computed independently. This is why GPUs dominate: they can compute thousands of those output elements simultaneously.


## Bottlenecks That Determine Inference Speed

Here's the key mental model. Inference speed is limited by whichever of these three is slowest:

**1. Compute throughput** — how many multiply-add operations per second the chip can do. GPUs win massively here: an NVIDIA T4 does ~65 TFLOPS (INT8), while a 4-core CPU does maybe ~0.5 TFLOPS. That's a 130x gap.

**2. Memory bandwidth** — how fast you can read model weights from RAM. This is actually the bottleneck for smaller models and batch size 1. The model weights must flow from memory → compute units every single forward pass. CPU DDR5 offers ~50 GB/s. GPU HBM3 offers ~3000 GB/s. This is why even a "simple" model like distilbert is 10x faster on GPU — at batch=1, you're mostly waiting on memory reads, not compute.

**3. Memory capacity** — can the model even _fit_? CPU RAM is typically 16-128 GB (plenty for most models). GPU VRAM is 8-80 GB. Larger models may not fit in VRAM at all, requiring model parallelism or quantization.



```
# Typical latency numbers for batch_size=1 on CPU vs GPU:

#                          CPU (4-core)    GPU (T4)     Ratio
# distilbert (66M params)    ~50ms           ~5ms       10x
# bert-base (110M params)    ~120ms          ~8ms       15x
# flan-t5-small (80M)        ~200ms          ~15ms      13x
# flan-t5-base (250M)        ~800ms          ~40ms      20x
# llama-2-7b (7B)            ~30s            ~200ms     150x
```


The key insight is that **not every part of an inference request needs a GPU**. Tokenizing text, validating JSON, routing requests, assembling responses — all of that is CPU work. Only the forward pass through the neural network (the matrix multiplications) needs GPU acceleration. So the architecture splits these concerns.

![](Pasted%20image%2020260321174241.png)



The key idea at this level: you split your cluster into two node pools. CPU nodes are cheap ($0.06/hr on AWS for `m5.large`) and handle the 90% of work that doesn't need parallel math. GPU nodes are expensive ($0.52/hr for `g4dn.xlarge` with a T4) and handle only the forward pass. The request flows through the cheap layer first, hits the GPU only for the ~5ms of actual inference, and comes back through the cheap layer for response assembly. This keeps your GPU utilization high and your costs sane.


_How_ K8s knows to send ML pods to GPU nodes and regular pods to CPU nodes. This is where taints, tolerations, node selectors, and the device plugin framework come together.


![](Pasted%20image%2020260321182306.png)


![](Pasted%20image%2020260321182318.png)


![](Pasted%20image%2020260321182339.png)


![](Pasted%20image%2020260321182355.png)

**Taints and tolerations** act as a bouncer. GPU nodes are "tainted" with `nvidia.com/gpu=:NoSchedule`, which means _no pod can land on that node unless it explicitly says "I tolerate this taint."_ This prevents your API server or Redis cache from accidentally landing on your expensive GPU node and wasting resources.

**Node selectors / node affinity** add a positive preference. The ML pod says `nodeSelector: {node-type: gpu}`, which tells the scheduler "only consider nodes with this label." Combined with the toleration, this creates a lock-and-key system.

**Extended resources** make GPU a first-class K8s resource. The NVIDIA device plugin (a DaemonSet running on every GPU node) registers `nvidia.com/gpu` as an allocatable resource with the kubelet. When a pod requests `nvidia.com/gpu: 1` in its resource limits, the scheduler knows to filter out any node that doesn't have a free GPU — just like it would filter out a node without enough CPU or memory.

```yaml
# The ML inference pod
spec:
  nodeSelector:
    node-type: gpu            # Only schedule on GPU-labeled nodes
  tolerations:
    - key: nvidia.com/gpu     # "I'm allowed on tainted GPU nodes"
      operator: Exists
      effect: NoSchedule
  runtimeClassName: nvidia    # Use nvidia container runtime
  containers:
    - name: model-server
      resources:
        limits:
          nvidia.com/gpu: 1   # "I need exactly 1 GPU"
          memory: "8Gi"
        requests:
          cpu: "2"
          memory: "4Gi"
```

The source code for this scheduling logic lives in the Kubernetes scheduler's filter plugins. Specifically, `pkg/scheduler/framework/plugins/noderesources/fit.go` handles the resource filtering — it checks whether a node has enough allocatable extended resources (including `nvidia.com/gpu`) to satisfy the pod's request. The taint/toleration logic is in `pkg/scheduler/framework/plugins/tainttoleration/taint_toleration.go`

![](Pasted%20image%2020260321182528.png)


The forward pass is ~85-95% of total latency. Every other phase — ingress, tokenization, network hop, response serialization — is under 1ms each. This is why the architecture works: you optimize the one thing that matters (moving matrix math to GPU), and you can afford to be "sloppy" about everything else because the overhead is negligible.



Keep GPU pods warm with model in VRAM

```yaml
# startupProbe gives the model time to load into VRAM
startupProbe:
  httpGet:
    path: /readyz
    port: 8000
  failureThreshold: 60    # 60 × 5s = 5 minutes to load
  periodSeconds: 5
```

In source code terms: PyTorch's `model.to('cuda')` copies all weight tensors from CPU RAM to GPU VRAM via `cudaMemcpy()`. This is a one-time cost. After that, `model(input_tensor.to('cuda'))` just reads from VRAM (3 TB/s bandwidth) — no CPU memory involved.

### Trick 2: Batching amortizes the GPU's fixed costs

A GPU has a "launch overhead" for each kernel — about 5-10 microseconds per operation to set up the compute grid. For a single input, this overhead adds up across hundreds of kernel launches in a forward pass. But if you batch 8 inputs together, the matrices get wider (8×768 instead of 1×768) and the GPU's massive parallelism finally gets fully utilized:


```
Batch=1:  5ms  per inference  (GPU at ~10% utilization)
Batch=8:  8ms  total          (GPU at ~60% utilization) → 1ms per inference
Batch=32: 15ms total          (GPU at ~90% utilization) → 0.47ms per inference

```

The inference server (like Triton or the FastAPI app from earlier) implements this by holding incoming requests in a queue for a few milliseconds, then batching them into a single forward pass.



Trick 3 Same-AZ placement eliminates network latency


### The NVIDIA device plugin + containerd make GPU access zero-overhead

When containerd creates the container, the NVIDIA runtime hook (`nvidia-container-runtime`) mounts the GPU device files (`/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-uvm`) directly into the container's mount namespace. The CUDA driver inside the container then talks to the kernel driver via these device files — there's no virtualization layer, no hypervisor tax. The container sees the GPU at bare-metal speed.

The device plugin source code lives at `github.com/NVIDIA/k8s-device-plugin`. The key function is `Allocate()` in `pkg/nvidia/plugin.go` — it returns the device IDs and mounts that kubelet passes to the container runtime.

![](Pasted%20image%2020260321182826.png)


And for the comparision this is Latency numbers for well known models. measured at batch_size=1, single-user. Real production throughput depends heavily on batching, quantization (FP8/INT4/AWQ), inference engine (vLLM, SGLang, TensorRT-LLM), and tensor/expert parallelism strategy. MoE models show "active" parameters because only a subset of experts fire per token — Kimi K2 has 1T total params but only 32B activate per token, making per-token compute comparable to a 32B dense model.


![](Pasted%20image%2020260321190745.png)

This divides the GPU serving into two parts. Below the cliff and above the cliff.

**Above the cliff** (Kimi K2, DeepSeek-V3/R1): these require 8+ H100s with high-bandwidth NVLink interconnect, and often need **expert parallelism** across GPUs. The Kimi K2 training strategy leverages 16-way Expert Parallelism, and storing the model parameters in BF16 with gradient buffers requires approximately 6 TB of GPU memory distributed over 256 GPUs. [arXiv](https://arxiv.org/pdf/2507.20534) For inference it's less extreme, but you still need a full DGX node or equivalent. The quantized version requires over 240GB of memory just for the weights, and for strong performance you want 10+ tokens/s. [Unsloth AI](https://unsloth.ai/docs/models/kimi-k2.5)




![](Pasted%20image%2020260321192541.png)

![](Pasted%20image%2020260321192847.png)