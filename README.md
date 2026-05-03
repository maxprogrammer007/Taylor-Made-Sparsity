# Taylor-Made Sparsity

> **Anonymous Submission — NeurIPS 2026**

> A fast, hardware-aware pruning method that selects functionally important channels using a theoretically grounded Taylor proxy, enabling reliable high-sparsity compression without breaking model structure.

This repository contains the official implementation of **Taylor-Made Sparsity**, a structural neural network pruning method that uses gradient-weighted activation importance (Grad-Act) to select which filters to remove, while preserving spatially localized, task-relevant activations (validated via Grad-CAM).

---

## Requirements

Python 3.10+ and a CUDA-capable GPU (≥8 GB VRAM recommended).

```bash
pip install -r requirements.txt
```

Key dependencies: PyTorch 2.x, torchvision, torch-pruning, pycocotools.

---

## Data Download

**CIFAR-10** is downloaded automatically by torchvision on first run.

**MS-COCO 2017** (~20 GB) must be downloaded manually:

```bash
bash data/download_data.sh ./datasets/coco
```

This downloads and extracts annotations, `val2017`, and `train2017` into `./datasets/coco/`.

> **Note:** By default COCO is expected at `/dev/shm/coco` (RAM disk) for I/O performance.
> Update `configs/default.yaml` → `data.coco_root` if you use a different path.

> **Note on COCO baseline:** The MobileNetV2-SSD baseline uses a lightweight training
> configuration optimised for fast, reproducible experimentation rather than
> state-of-the-art detection accuracy. All methods share this identical baseline,
> so pruning comparisons are internally consistent.

---

## Reproducing the Main Results

### Full pipeline (baseline training + pruning, both tasks)

```bash
bash run_experiment.sh
```

This sequentially runs:
1. `scripts/run_baseline.py` — trains ResNet-50/CIFAR-10 and MobileNetV2-SSD/COCO baselines
2. `scripts/run_pruning.py` — prunes + fine-tunes all methods × all sparsity levels

Results are saved as CSV and JSON to `outputs/results/`.

### Ablation studies (Table 2 in paper)

```bash
python scripts/run_ablations_v2.py --seed 42
```

### Grad-CAM visualizations (Appendix A)

```bash
python scripts/run_gradcam.py
```

---

## Expected Results

Running `bash run_experiment.sh` should reproduce the following key numbers from the paper.
Small variations of ±0.2% Top-1 / ±0.3 mAP may occur due to non-determinism in CUDA kernels.

### ResNet-50 / CIFAR-10

| Method       | Prune % | Top-1 Acc (%) | GFLOPs | FPS  |
|-------------|---------|--------------|--------|------|
| Baseline    | 0%      | ~97.1        | 1.311  | —    |
| L1-Norm     | 30%     | ~95.5        | 0.611  | ~225 |
| Grad-Act (Ours) | 30% | ~95.7       | 0.611  | ~230 |
| Grad-Act (Ours) | 50% | ~94.2       | 0.331  | ~235 |

### MobileNetV2-SSD / COCO (mAP @ [0.50:0.95])

| Method       | Prune % | mAP (%)  | GFLOPs | FPS  |
|-------------|---------|---------|--------|------|
| Baseline    | 0%      | ~21.0   | 1.224  | ~148 |
| Grad-Act (Ours) | 30% | ~12.5  | 1.023  | ~70  |
| DepGraph    | 30%     | ~12.6   | 1.023  | ~149 |

> FPS measured on NVIDIA RTX 4090, batch size 1, 200 iterations after 50 warm-up iterations.

---

## Method Summary

We estimate channel importance using a squared gradient–activation interaction:

```
I(k, l) = Σ_x (∂L/∂a(k,l,x)  ⊙  a(k,l,x))²
```

summed over spatial positions `x` and calibration samples. This formulation:

- **Avoids gradient sign cancellation** across spatial locations (e.g., Σ (g⁺ − g⁻) ≈ 0 in dense prediction tasks)
- **Approximates second-order sensitivity** via the Fisher Information Matrix diagonal
- **Enables stable estimation** in multi-task settings (e.g., detection with multi-scale heads)

We also evaluate an absolute variant |∂L/∂a ⊙ a|; however, the squared formulation provides lower variance and more stable importance estimates across calibration samples and seeds (see ablation results in `outputs/results/ablation_results_v2.csv`).

Structural consistency is enforced using dependency-graph-based grouping (DepGraph), ensuring all pruned models remain architecturally valid and hardware-efficient.

---

## Reproducibility Notes

This repository supports full reproducibility of all claims in the paper:

- ✔ Dataset download scripts provided (`data/download_data.sh`)
- ✔ All hyperparameters exposed in `configs/default.yaml`
- ✔ Random seeds fixed at 42, 123, 2024 (multi-seed ablations in Table 3)
- ✔ Full pipeline executable with a single command (`bash run_experiment.sh`)
- ✔ Results saved as CSV and JSON in `outputs/results/` for independent verification

---

## Compute Requirements

Approximate runtimes on a single **NVIDIA RTX 4090**:

| Stage | ResNet-50 / CIFAR-10 | MobileNetV2-SSD / COCO |
|---|---|---|
| Baseline training | ~2 hours | ~18 hours |
| Importance scoring | ~1.5 seconds | ~3 seconds |
| Dependency graph construction | ~0.8 seconds | ~1 seconds |
| Fine-tuning (post-pruning) | ~8 min (5 epochs) | ~45 min (20 epochs) |
| FPS benchmark (one config) | ~1 min | ~1 min |

**Total compute used in the paper: ~72 GPU-hours.**

---

## Repository Structure

```
.
├── configs/           # All hyperparameters (configs/default.yaml)
├── data/              # Dataset loaders + download script
├── evaluation/        # FLOPs counter, FPS benchmark, CIFAR/COCO eval
├── explainability/    # Grad-CAM implementation
├── models/            # ResNet-50/CIFAR and MobileNetV2-SSD definitions
├── pruning/           # ★ Core contribution
│   ├── importance.py         # Grad-Act importance scoring
│   └── structural_pruner.py  # Filter removal + channel correction
├── scripts/           # Experiment runner scripts
├── training/          # Training loops and losses
├── outputs/results/   # Generated CSVs and JSON result files
├── requirements.txt
└── run_experiment.sh  # One-command full pipeline
```

---

## Pretrained Checkpoints

Pretrained checkpoints are hosted anonymously at:

> 🔗 **[Anonymous Zenodo link — to be added upon de-anonymisation]**

Download and place them in `outputs/checkpoints/` to skip training and run evaluation only.

---

## Configuration

All hyperparameters are in [`configs/default.yaml`](configs/default.yaml). Key settings:

| Setting | ResNet-50 / CIFAR-10 | MobileNetV2-SSD / COCO |
|---|---|---|
| Baseline epochs | 50 | 100 |
| Batch size | 512 | 128 |
| Base LR | 0.04 | 0.04 |
| LR schedule | Cosine | Cosine |
| Fine-tune epochs | 5 | 20 |
| Fine-tune LR | 0.004 | 0.004 |
| Calibration samples | 2,000 | 1,000 |
| Pruning ratios | 5–50% | 5–50% |

---

## Troubleshooting

**COCO download fails or is incomplete:**
Ensure ~22 GB of free disk space and a stable connection. The script uses `wget -c` and
can be safely re-run to resume an interrupted download.

**CUDA out-of-memory during training:**
Reduce `batch_size` in `configs/default.yaml` under `training.resnet` or `training.ssd`.

**FPS measurements are inconsistent between runs:**
Ensure no other GPU workloads are running concurrently. The benchmark discards 50
warm-up iterations to reduce variance, but shared-GPU environments will still cause noise.

**`torch_pruning` import error:**
Run `pip install torch-pruning` (note the hyphen). The package name differs from the import name.

**Mismatched state_dict when loading pruned checkpoints:**
Structural pruning physically changes layer dimensions. Always rebuild the pruned
architecture first (by re-applying pruning to a fresh baseline copy) before calling
`load_state_dict`. See `scripts/run_gradcam.py` for a reference implementation.
