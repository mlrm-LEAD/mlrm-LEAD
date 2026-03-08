# LEAD: Latent Entropy-Aware Decoding

[![arXiv](https://img.shields.io/badge/arXiv-xxxx.xxxxx-b31b1b.svg)](https://arxiv.org/abs/xxxx.xxxxx) [![Conference](https://img.shields.io/badge/CVPR-2026-blue.svg)]() [![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

This repository implements **LEAD** (Latent Entropy-Aware Decoding), a training-free decoding strategy that mitigates hallucinations in Multimodal Large Reasoning Models (MLRMs) by adaptively switching between soft and hard decoding modes based on real-time latent entropy signals.

![LEAD Framework](/root/autodl-tmp/mlrm-LEAD/figure/method.pdf)

*Overview of the LEAD framework. LEAD monitors the entropy of token probability distributions during autoregressive generation. When entropy rises above a threshold, the decoder switches to **soft mode** — blending probability-weighted embeddings with anchor tokens to encourage exploratory reasoning. When entropy drops, it reverts to **normal mode** — using standard discrete token selection for precise, committed generation. This entropy-driven soft/hard switching mechanism reduces hallucinations without any additional training or external tools.*

## Setup

1. Clone the repository:

```bash
git clone https://github.com/Zhongxing-XU/LEAD.git
cd LEAD
```

2. Install dependencies:

```bash
pip install -r requirements.txt
```

3. Prepare model weights:

Download [Qwen2.5-VL-7B-Instruct](https://huggingface.co/Qwen/Qwen2.5-VL-7B-Instruct) from Hugging Face, or specify the model name directly to load from the Hub:

```bash
# Option A: Use Hugging Face model name (auto-download)
--model_name Qwen/Qwen2.5-VL-7B-Instruct

# Option B: Use local path
--model_name /path/to/Qwen2.5-VL-7B-Instruct
```

## Quick Start

### Basic Usage

Run inference using the provided script:

```bash
bash script/run.sh
```

### Try with Demo

Run a quick demo on a single sample:

```bash
python main.py \
    --model_name Qwen/Qwen2.5-VL-7B-Instruct \
    --dataset data/demo.jsonl \
    --method lead \
    --max_new_tokens 2048
```

### Custom Configuration

You can modify `script/run.sh` or run directly with Python:

```bash
python main.py \
    --model_name Qwen/Qwen2.5-VL-7B-Instruct \
    --dataset data/physunibench.jsonl \
    --output_dir output \
    --method lead \
    --alpha 0.6 \
    --max_switch_count 5 \
    --temperature 0.6 \
    --top_p 0.95 \
    --top_k 20 \
    --max_new_tokens 25600 \
    --seed 42
```

## Configuration

### Key Parameters

#### Dataset & Model

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--model_name` | `Qwen/Qwen2.5-VL-7B-Instruct` | HuggingFace model name or local weight path |
| `--dataset` | `data/physunibench.jsonl` | Path to dataset JSONL file |
| `--output_dir` | `output/` | Directory to save results |
| `--limit` | `None` | Only run first N samples (for debugging) |

#### Decoding Method

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--method` | `lead` | Decoding method: `lead` / `cot` / `cot_greedy` |
| `--alpha` | `0.6` | LEAD soft-mode blending coefficient (\(\alpha_0\)); higher values bias toward probability-weighted embeddings over anchor tokens |
| `--max_switch_count` | `5` | Maximum number of soft→normal mode switches before triggering convergence injection |

#### Sampling

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--temperature` | `0.6` | Sampling temperature |
| `--top_p` | `0.95` | Nucleus sampling threshold |
| `--top_k` | `20` | Top-k filtering |
| `--min_p` | `0.0` | Minimum probability filtering |
| `--max_new_tokens` | `25600` | Maximum tokens to generate |
| `--do_sample` / `--no-do_sample` | `True` | Enable/disable stochastic sampling |
| `--seed` | `42` | Random seed for reproducibility |

#### Data Filtering

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--subtopics` | `None` | Filter by subtopics, e.g. `--subtopics Mechanics Optics` |
| `--language` | `None` | Filter by language: `english` / `chinese` |
| `--min_difficulty` | `1` | Minimum difficulty level (inclusive) |
| `--max_difficulty` | `5` | Maximum difficulty level (inclusive) |

#### Evaluation Only

Re-evaluate existing results without running inference:

```bash
python main.py --eval_only --results output/results.jsonl
```

## Available Scripts

| Script | Description |
|--------|-------------|
| `script/run.sh` | Full evaluation with LEAD method |
| `script/run_cot.sh` | Full evaluation with CoT baseline |
| `script/run_debug.sh` | Debug mode: 5 samples, short generation |
| `script/run_eval.sh` | Evaluate existing results only |

## Datasets

Place your dataset JSONL files in the `data/` directory. Expected format:

```json
{"id": 1, "image": "path/to/image.jpg", "question": "What is shown?", "options": "A. ...\nB. ...\nC. ...\nD. ...", "answer": "A"}
```

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Sample identifier |
| `image` | Yes | Absolute or relative path to the image |
| `question` | Yes | Question text |
| `options` | No | MCQ options (leave empty for open-ended questions) |
| `answer` | No | Ground truth answer (for evaluation) |

The repository includes the following benchmark datasets:

- `physunibench.jsonl` — PhysUniBench physics reasoning
- `math_vision.jsonl` — MathVision mathematical reasoning
- `math_vista.jsonl` — MathVista visual math
- `mmvp.jsonl` — MMVP visual perception
- `realworldqa.jsonl` — RealWorldQA real-world reasoning
- `visulogic.jsonl` — VisuLogic visual logic
- `vstar.jsonl` — V* benchmark
- `demo.jsonl` — Quick demo (1 sample)

## Output

Results are saved in the specified `output_dir`:

- `config.json` — Run configuration and hyperparameters
- `results.jsonl` — Per-sample predictions
- `eval_report.json` — Evaluation report with accuracy breakdown
- `logs/` — Timestamped log files

### Results JSONL Structure

```json
{
  "id": 1,
  "image": "path/to/image.jpg",
  "question": "...",
  "options": "A. ...\nB. ...",
  "answer": "A",
  "model_answer": "The answer is A because ...",
  "extracted_answer": "A",
  "is_correct": true
}
```

### Evaluation Report Structure

```json
{
  "accuracy": 0.75,
  "correct": 75,
  "total": 100,
  "failed_extraction": 0,
  "by_subtopic": { "Mechanics": { "accuracy": 0.80, "correct": 40, "total": 50 } },
  "by_difficulty": { "3": { "accuracy": 0.70, "correct": 35, "total": 50 } },
  "by_language": { "english": { "accuracy": 0.75, "correct": 75, "total": 100 } },
  "config": { "..." : "..." }
}
```

## Project Structure

```
LEAD/
├── main.py                    # Main entry point
├── lead/
│   ├── __init__.py            # Package exports
│   ├── generation_utils.py    # Core LEAD & CoT generation algorithms
│   ├── inference.py           # Input construction & single-sample inference
│   ├── data.py                # Dataset loading & preprocessing
│   ├── evaluator.py           # Answer evaluation & accuracy statistics
│   ├── prompts.py             # Prompt template management
│   ├── logger.py              # Logging system
│   └── utils.py               # General utility functions
├── data/                      # Dataset JSONL files
├── example/                   # Demo images
├── script/
│   ├── run.sh                 # LEAD evaluation script
│   ├── run_cot.sh             # CoT baseline script
│   ├── run_debug.sh           # Debug mode script
│   └── run_eval.sh            # Evaluation-only script
├── tests/                     # Unit tests
├── requirements.txt           # Python dependencies
├── setup.py                   # Package installation config
├── LICENSE                    # MIT License
└── CONTRIBUTING.md            # Contribution guidelines
```

## Citation

If you find this code useful in your research, please cite:

```bibtex
@inproceedings{xu2026lead,
      title={Thinking in Uncertainty: Mitigating Hallucinations in MLRMs with Latent Entropy-Aware Decoding},
      author={Zhongxing Xu},
      booktitle={Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)},
      year={2026}
}
```

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
