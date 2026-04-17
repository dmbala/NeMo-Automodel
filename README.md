# nemo_autotune

A collection of examples for autotuning, training, and deploying Large Language Models (LLMs), Vision Language Models (VLMs), and diffusion models using the [NVIDIA NeMo Framework](https://github.com/NVIDIA/NeMo).

---

## Overview

`nemo_autotune` provides ready-to-use scripts and configurations that leverage NeMo's built-in autotuning capabilities to find optimal training hyperparameters (micro-batch size, tensor parallelism, pipeline parallelism, etc.) for your hardware cluster, and then launch fully-optimised training and inference runs.

---

## Features

- **Automatic hyperparameter search** – uses NeMo's autotuner to sweep over parallelism strategies and batch sizes to maximize GPU utilization and throughput.
- **LLM support** – examples covering GPT-style models (e.g., Llama, Mistral, Mixtral).
- **VLM support** – examples for vision-language models (e.g., CLIP, LLaVA).
- **Diffusion model support** – examples for latent-diffusion and score-based generative models.
- **Multi-node ready** – configurations tested on single-node and multi-node SLURM/Kubernetes clusters.
- **NeMo 2.x API** – examples use the modern `nemo.collections` and `nemo.lightning` APIs.

---

## Prerequisites

| Requirement | Version |
|---|---|
| Python | ≥ 3.10 |
| CUDA | ≥ 12.1 |
| PyTorch | ≥ 2.2 |
| NVIDIA NeMo Framework | ≥ 2.0 |
| NVIDIA Apex | latest |
| Megatron-LM | bundled with NeMo |

The easiest way to satisfy all prerequisites is to use the official NeMo container:

```bash
docker pull nvcr.io/nvidia/nemo:25.02
```

---

## Installation

```bash
# Clone the repository
git clone https://github.com/dmbala/nemo_autotune.git
cd nemo_autotune

# (Optional) create a virtual environment
python -m venv .venv && source .venv/bin/activate

# Install NeMo with all extras
pip install "nemo_toolkit[all]"
```

---

## Quick Start

### 1. Autotune a Llama-3 8B model

```bash
python examples/llm/llama3_autotune.py \
    --model_size 8b \
    --num_nodes 1 \
    --num_gpus_per_node 8 \
    --max_minutes_per_run 5 \
    --output_dir ./autotune_results
```

The autotuner will:
1. Profile a range of parallelism/batch-size combinations.
2. Select the configuration that achieves the highest training throughput.
3. Write the optimal `trainer_config.yaml` to `output_dir`.

### 2. Launch training with the tuned configuration

```bash
python examples/llm/llama3_train.py \
    --config ./autotune_results/trainer_config.yaml
```

---

## Repository Structure

```
nemo_autotune/
├── examples/
│   ├── llm/          # LLM training & autotuning examples
│   ├── vlm/          # Vision-language model examples
│   └── diffusion/    # Diffusion model examples
├── configs/          # Base YAML configurations
└── README.md
```

---

## Configuration

Key autotuning parameters (set via CLI or YAML):

| Parameter | Description | Default |
|---|---|---|
| `num_nodes` | Number of compute nodes | `1` |
| `num_gpus_per_node` | GPUs per node | `8` |
| `max_minutes_per_run` | Time budget per trial | `5` |
| `model_size` | Model size shorthand (e.g., `7b`, `13b`, `70b`) | — |
| `tensor_model_parallel_size` | Tensor parallelism degree (autotuned if not set) | auto |
| `pipeline_model_parallel_size` | Pipeline parallelism degree (autotuned if not set) | auto |
| `micro_batch_size` | Per-GPU micro batch size (autotuned if not set) | auto |

---

## Contributing

Contributions are welcome! Please open an issue or submit a pull request.

1. Fork the repository.
2. Create a feature branch: `git checkout -b feature/my-feature`.
3. Commit your changes: `git commit -m "Add my feature"`.
4. Push and open a PR against `main`.

---

## License

This project is licensed under the [Apache 2.0 License](LICENSE).

---

## References

- [NVIDIA NeMo Framework](https://github.com/NVIDIA/NeMo)
- [NeMo Documentation](https://docs.nvidia.com/nemo-framework/user-guide/latest/)
- [Megatron-LM](https://github.com/NVIDIA/Megatron-LM)
