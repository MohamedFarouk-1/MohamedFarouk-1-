# SparseOpt

**A PyTorch computation graph optimizer that automatically fuses operators, eliminates dead nodes, and reorders execution — with zero changes to your model code.**

Most ML engineers treat model inference as a black box. SparseOpt opens the graph, applies a compiler-style optimization pipeline via `torch.fx`, verifies numerical correctness to `1e-4` tolerance, and hands back a faster model.

---

## Why I Built This

ML inference optimization is almost always left on the table.

Engineers spend months tuning training pipelines but ship inference code that's essentially unchanged from the research prototype — full of redundant dropout ops that do nothing at eval time, operator sequences that stall the hardware pipeline, and Linear+Activation patterns that could be a single fused kernel.

The tools to fix this exist inside PyTorch (`torch.fx`, `torch.nn.intrinsic`) but they're compiler-level APIs that most practitioners never touch. **SparseOpt wraps them into a pipeline anyone can run in one command.**

I built this because I wanted to understand what happens between `model.forward()` and the hardware — and because inference latency is a real product problem. A 4% reduction on BERT-base sounds small, but it compounds: 36 fusion passes firing across 12 transformer layers, each one reducing memory bandwidth at the kernel boundary. On GPU, the gains are larger because fused kernels eliminate kernel-launch overhead entirely.

**What I learned:**
- How `torch.fx` represents a model as a symbolic computation graph
- Which optimization passes are safe (numerically verified to `1e-4`) vs. which require careful correctness checking
- Why some models (LLMs with dynamic control flow) resist whole-graph tracing and need layer-by-layer fallback strategies
- The difference between what a model *does* and how it *executes* — and why that gap matters for product decisions about latency, cost, and hardware selection

---

## Benchmark Results

Measured on CPU (100 runs, 10 warmup iterations).

| Model | Params | Baseline (ms) | SparseOpt (ms) | Speedup | Optimizations Applied |
|-------|--------|:---:|:---:|:---:|---|
| **MLP** | 6.3M | 0.33 | 0.33 | **1.01×** | Dropout elimination ×2, Linear+ReLU fusion ×3, 11→6 graph nodes |
| **ResNet-50** | 25.6M | 14.90 | 14.59 | **1.02×** | Node reordering (layer-by-layer) |
| **BERT-base** | 109.5M | 18.35 | 17.58 | **1.04×** | Linear+GELU fusion ×36 across 37 traced submodules |

> CPU gains are conservative. GPU workloads see larger improvements because fused kernels reduce kernel-launch overhead. BERT's 4.2% reduction compounds across all 12 transformer layers — the same fusion pass fires 36 times.

---

## How It Works

SparseOpt uses `torch.fx` to trace a model into a symbolic computation graph, then runs a pipeline of optimization passes:

```
nn.Module
    │
    ▼
torch.fx.symbolic_trace()
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  Optimization Pipeline                                   │
│                                                         │
│  1. DropoutElimination      remove no-op inference nodes│
│  2. DeadNodeElimination     prune unreachable subgraphs  │
│  3. NodeReordering          schedule lightweight ops first│
│  4. Operator Fusion         collapse sequential patterns │
│     ├─ Conv2d + BN + ReLU → ConvBnReLU2d               │
│     ├─ Conv2d + ReLU      → ConvReLU2d                  │
│     ├─ Linear + ReLU      → FusedLinearReLU             │
│     ├─ Linear + GELU      → FusedLinearGELU             │
│     └─ LayerNorm + Linear + LayerNorm → fused block     │
└─────────────────────────────────────────────────────────┘
    │
    ▼
Numerical Verification  (torch.allclose, rtol=1e-4, atol=1e-4)
    │
    ▼
Benchmarking  (10 warmup + 100 timed iterations)
    │
    ▼
Optimized nn.Module
```

For models with dynamic control flow that resist whole-graph tracing, SparseOpt falls back to layer-by-layer tracing, applying passes to each traceable submodule independently.

---

## Installation

```bash
git clone https://github.com/MohamedFarouk-1/SparseOpt.git
cd SparseOpt
pip install -r requirements.txt
pip install -e .
```

---

## Usage

### CLI

```bash
python optimize.py --model bert-base-uncased --device cuda
```

| Flag | Default | Description |
|------|---------|-------------|
| `--model` | required | HuggingFace model name or local path |
| `--device` | `cuda` | `cuda` or `cpu` |
| `--num-runs` | `10` | Benchmark iterations |
| `--warmup` | `3` | Warmup runs before timing |
| `--text` | `"Hello, this is SparseOpt!"` | Input text |
| `--max-length` | `128` | Tokenizer max sequence length |

### Python API

```python
from sparseopt.huggingface import optimize_hf_model, benchmark_model

optimized_model, stats = optimize_hf_model("bert-base-uncased", device="cuda")
result = benchmark_model(optimized_model, inputs, device="cuda", num_runs=100, warmup=10)
print(f"Speedup: {baseline / result['mean_latency']:.2f}x")
```

### Compose passes manually

```python
from sparseopt.graph.passes.dead_node import DeadNodeEliminationPass
from sparseopt.graph.passes.linear_fusion import LinearGELUFusion
from sparseopt.graph.optimizer_pipeline import OptimizerPipeline

pipeline = OptimizerPipeline(
    passes=[DeadNodeEliminationPass(), LinearGELUFusion()],
    verify_correctness=True,
    benchmark=True,
)
optimized_gm, stats = pipeline.optimize(gm, example_inputs=inputs)
```

---

## Supported Fusion Patterns

| Pattern | Target Architecture |
|---------|---|
| `Conv2d → BatchNorm → ReLU` | CNNs (ResNet, EfficientNet) |
| `Conv2d → ReLU` | CNNs |
| `Linear → ReLU` | MLPs, classifier heads |
| `Linear → GELU` | Transformers (GPT, BERT) |
| `LayerNorm → Linear → LayerNorm` | Transformer blocks |

---

## Tested Models

`bert-base-uncased` · `distilbert-base-uncased` · `facebook/opt-350m` · `gpt2` · ResNet-18/50 · Custom MLP and GNN models

---

## Project Structure

```
sparseopt/
├── graph/
│   ├── passes/
│   │   ├── dead_node.py
│   │   ├── reordering.py
│   │   ├── dropout_elimination.py
│   │   ├── linear_fusion.py
│   │   └── conv_fusion.py
│   ├── optimizer_pipeline.py
│   └── base.py
├── huggingface.py
├── optimize.py
└── utils/benchmark.py
```

---

## License

MIT
