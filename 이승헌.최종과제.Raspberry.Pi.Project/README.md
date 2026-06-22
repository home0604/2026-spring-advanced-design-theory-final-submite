# RP2-RPS-QAT-Lab: Rock-Paper-Scissors Classification with Quantization-Aware Training & Structured Pruning

A comprehensive research laboratory for implementing **Quantization-Aware Training (QAT)**, **Network Slimming (Structured Pruning)**, and **Model Comparison** on ResNet50 for Rock-Paper-Scissors (RPS) image classification.

## Project Overview

This project explores three key optimization techniques for deploying deep learning models on edge devices (mobile, embedded, Raspberry Pi):

1. **Quantization-Aware Training (QAT)**: Convert float32 models to int8 with minimal accuracy loss
2. **Structured Pruning (Network Slimming)**: Reduce model complexity via L1 regularization on BatchNormalization gamma parameters
3. **Model Comparison**: Evaluate trade-offs between accuracy, file size, FLOPs, and parameters across pruning strategies

## Key Files

### Main Notebook
- **`project_2025324140.ipynb`** - Primary research notebook for ResNet50 QAT + structured pruning
  - **Cell 2d4bea15**: Tier 1 (In-place channel masking) with 30% sparsity
  - **Cell 7a9ab4ab**: Tier 3 (Model rebuild) with reduced channel dimensions
  - **Cell a8cab07d**: Comprehensive 4-model comparison (baseline, Tier1-unstructured, Tier1-structured, Tier3-rebuilt)

### Utility Modules (`utils/`)
- **`pruning_ablation.py`** - Structured pruning with L1 sparsity learning
  - Temporarily unfreezes BatchNormalization during L1 training phase
  - Generates independent channel masks per BN layer
  - Applies Tier 1 in-place masking
  - Converts to TFLite with op version compatibility fix

- **`structured_rebuild.py`** - Tier 3 model reconstruction
  - Rebuilds ResNet50 with pruned channels removed
  - Supports independent channel widths (k1, k2) per bottleneck
  - Transfers weights from original to pruned model
  - Exports to TFLite with full int8 quantization

### Pre-trained Models (`results/`)
**Baseline (QAT)**
- `RPS_PreTrained_ResNet_Augmentation_QAT.h5` (271 MB, float32)
- `RPS_PreTrained_ResNet50_Augmentation_QAT.tflite` (24 MB, int8)

**Tier 1: Unstructured Pruning**
- `pruned_unstructured_sparsity30.h5` (271 MB, float32)
- `pruned_unstructured_sparsity30.tflite` (24 MB, int8)

**Tier 3: Structured Pruning (Rebuilt)**
- `rebuilt_structured_channel30.h5` (172 MB, float32, 37% smaller)
- `rebuilt_structured_channel30.tflite` (15 MB, int8, 37% smaller)

## Methodology

### L1 Sparsity Learning (Structured Pruning)
1. Add L1 regularization to BatchNormalization gamma parameters
2. Temporarily unfreeze BN layers to enable gradient flow during L1 phase
3. Generate channel masks based on gamma magnitude thresholds
4. Apply masks:
   - **Tier 1**: In-place masking (shape preserved, file size unchanged)
   - **Tier 3**: Model rebuild (channels removed, file size reduced by 37%)

### Quantization-Aware Training (QAT)
1. Train model with fake quantization to simulate int8 inference
2. Calibrate quantization ranges using representative dataset
3. Convert to TFLite with full int8 quantization (DEFAULT optimization)
4. Evaluate accuracy on both float32 (Keras) and int8 (TFLite) models

## Key Technical Insights

### Handling BN Freezing
In QAT models, BatchNormalization layers are frozen to stabilize training. For L1 sparsity learning:
- **Problem**: Frozen gamma parameters don't respond to L1 penalty
- **Solution**: Temporarily unfreeze BN during L1 training phase, restore frozen state afterward
- **Validation**: Check gamma gradient flow with `report_gamma_sparsity()`

### Tier 1 vs Tier 3 Trade-offs
| Aspect | Tier 1 | Tier 3 |
|--------|--------|--------|
| Architecture | Original shape + masks | Rebuilt with fewer channels |
| File Size | ~24 MB (unchanged) | ~15 MB (-37%) |
| Inference Speed | Slow (masked computation) | Fast (actual size reduction) |
| Training Complexity | Low | Medium |

### Independent Mask per BN Layer
- **Problem**: Sharing same mask between bn1 and bn2 in bottleneck causes wrong layers to be pruned
- **Solution**: Generate separate masks for each BN layer from its own gamma distribution
- **Result**: More accurate channel selection per layer

## Getting Started

### 1. Environment Setup
```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements-cnn.txt
```

### 2. Running the Lab Notebook
Open `project_2025324140.ipynb` in Jupyter and execute cells:

**Option A: Quick Test (Tier 1 Structured Pruning)**
- Run Cell 2d4bea15 for configuration
- Model comparison results in ~30 minutes

**Option B: Full Experiment (Tier 1 + Tier 3)**
- Run Cell 2d4bea15 (Tier 1 configuration)
- Run Cell 7a9ab4ab (Tier 3 rebuild - ~2 hours)
- Run Cell a8cab07d (comprehensive comparison)

### 3. Important Notes
- **Restart kernel** between runs to reload modified utility modules
- Ensure GPU is available: `nvidia-smi`
- Dataset is auto-loaded from `RPS_Dataset/` directory

## Performance Results

### 4-Model Comparison
```
Model                   | Type      | Accuracy | File Size | Reduction
Baseline (QAT)          | Reference | -        | 24 MB     | -
Tier1-Unstructured      | Masked    | -        | 24 MB     | 0%
Tier1-Structured        | Masked    | -        | 24 MB     | 0%
Tier3-Rebuilt           | Slim      | -        | 15 MB     | 37.5%
```
*Run notebook to populate accuracy metrics (float32 vs int8)*

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| "Op version 'Fully_connected' v12" error | TFLite op mismatch | Restart Jupyter kernel |
| L1 sparsity not working | BN layers frozen | Unfreezing code in `pruning_ablation.py` line 256 |
| File size not reducing (Tier 1) | By design (masks only) | Use Tier 3 rebuild instead |

## Project Structure

```
RP2-RPS-QAT-Lab/
├── README.md                              (this file)
├── project_2025324140.ipynb              (main experiment notebook)
├── requirements-cnn.txt                   (dependencies)
├── utils/
│   ├── pruning_ablation.py               (L1 sparsity + Tier 1 masking)
│   ├── structured_rebuild.py             (Tier 3 model rebuild)
│   └── __init__.py
├── results/                               (model checkpoints & TFLite exports)
│   ├── RPS_PreTrained_ResNet_Augmentation_QAT.h5
│   ├── RPS_PreTrained_ResNet50_Augmentation_QAT.tflite
│   ├── pruned_unstructured_sparsity30.h5/tflite
│   └── rebuilt_structured_channel30.h5/tflite
└── RPS_Dataset/                           (training data)
    ├── 0/ (scissors)
    ├── 1/ (rock)
    └── 2/ (paper)
```

## References

- Network Slimming: Learning Efficient Convolutional Networks via L1 Regularization of Weights (ICLR 2017)
- Quantization-Aware Training (TensorFlow Model Optimization)
- ResNet: He et al., "Deep Residual Learning for Image Recognition" (CVPR 2016)
- TensorFlow Lite Quantization Guide

---

**Research Focus**: Model optimization for edge device deployment  
**Last Updated**: 2025-06-22  
**Target Deployment**: Mobile, embedded systems, Raspberry Pi
