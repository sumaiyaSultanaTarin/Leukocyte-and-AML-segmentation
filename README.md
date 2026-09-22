# AML Blast Cell Segmentation: Ratio-Consistency Loss and Ratio-Feedback Gating

Code for the letter **"An Ablation of Ratio-Aware Supervision for Leukocyte and AML Segmentation."**

Leukocyte segmentation models are usually scored by pixel overlap (Dice), but morphological assessment of acute myeloid leukaemia (AML) relies on measurements such as the nucleus-to-cytoplasm (N:C) ratio. This repository implements and evaluates two mechanisms that target the N:C ratio directly, rather than relying on overlap metrics as a proxy:

- **Ratio-consistency loss** — a Huber penalty on the log N:C area ratio between the predicted and reference masks.
- **Ratio-Feedback Gate (RFG)** — a lightweight decoder module that estimates the network's own N:C ratio and uses it to recalibrate decoder features before the segmentation head.

Both are evaluated in a controlled ablation on the source-domain **WBCAtt+** dataset (Attention U-Net with scSE decoder attention, and a plain U-Net control), then transferred to a pseudo-labeled, human-corrected target-domain subset of **AML-Cytomorphology_LMU**.

## Key finding

Explicit ratio supervision does not reliably improve ratio agreement over a strong baseline, and on the Attention U-Net it is marginally worse. The clearer result is orthogonal to the proposed mechanisms: transferring a WBCAtt+-pretrained checkpoint to AML images significantly improves Dice (+0.09, *p* = 0.007) but leaves N:C ratio error statistically unchanged (*p* = 0.78–0.94) — a controlled demonstration that overlap-based segmentation quality and morphometric measurement reliability are separable outcomes that must be validated independently.

## Repository structure

```
.
├── AML_Phase_I.ipynb    # Phase I: WBCAtt+ ablation (Attention U-Net vs plain U-Net, ratio loss + RFG)
├── AML_Phase_II.ipynb   # Phase II: AML-Cytomorphology_LMU transfer + target-domain RFG ablation
├── requirements.txt
├── README.md
└── LICENSE
```

Each notebook is self-contained (Colab-style): environment setup, data loading, model/loss definitions, training, and evaluation all live in one file, organized into sequential sections (`AML_Phase_I.ipynb`: Phase-A setup/data → Phase-B dataset/augmentation → Phase-C model/losses → Phase-D training → Phase-F evaluation and statistics). `AML_Phase_II.ipynb` reuses the same model and loss definitions from Phase I.

**Before running:** both notebooks mount Google Drive and will prompt you to enter your project folder path (e.g. `/content/drive/MyDrive/AML_segmentation_project`, with `data/`, `checkpoints/`, and `logs/` subfolders) — no need to edit the code. Datasets and trained checkpoints are hosted on Drive, not in this repository (see Datasets below for the public sources).

## Datasets

| Dataset | Role | Access |
|---|---|---|
| **WBCAtt+** | Source-domain pretraining and Phase I ablation (10,298 image–mask pairs, official 6,169/1,030/3,099 split) | [Hugging Face](https://huggingface.co/datasets/apple2373/wbcattplus) · [GitHub](https://github.com/apple2373/wbcattplus) |
| **AML-Cytomorphology_LMU** | Target-domain fine-tuning and evaluation (Phase II); no expert compartment masks — pseudo-labeled, with a 50-image human-corrected gold subset | [The Cancer Imaging Archive](https://www.cancerimagingarchive.net/collection/aml-cytomorphology_lmu/) · DOI: [10.7937/TCIA.2019.36F5O9LD](https://doi.org/10.7937/tcia.2019.36f5o9ld) |

## Requirements

Both notebooks were run on Google Colab, which ships with PyTorch, OpenCV, scikit-image, SciPy, and pandas preinstalled; the notebooks additionally install:

- segmentation-models-pytorch
- albumentations
- huggingface_hub
- pingouin (for ICC computation)

To run outside Colab, install the full stack:

```bash
pip install -r requirements.txt
```

with `requirements.txt` containing at minimum: `torch`, `segmentation-models-pytorch`, `albumentations`, `huggingface_hub`, `opencv-python`, `scikit-image`, `scipy`, `pandas`, `statsmodels`, `pingouin`.

## Method summary

Both backbones use a ResNet-34 encoder with a U-Net decoder. The primary backbone adds scSE (spatial and channel squeeze-and-excitation) recalibration within the decoder; the control backbone has no decoder attention. The total training loss is

```
L = L_Dice + λ1·L_Focal + λ2·L_ratio + λ3·L_probe
```

with λ1 = 0.5 fixed, and λ2 (ratio-consistency loss) / λ3 (RFG's auxiliary probe supervision) ramped linearly from 0 to 0.3 over the first 25% of training whenever the corresponding component is enabled. Models are trained with AdamW, cosine annealing, and mixed precision; the checkpoint with the lowest validation N:C ratio error is retained.

## Ablation configurations

**Phase I (WBCAtt+, Attention U-Net):** baseline, ratio-loss only, RFG only, and the full method (both), each across 5 seeds — plus a plain U-Net control with RFG on/off (ratio loss held on), 5 seeds each.

**Phase II (AML-Cytomorphology_LMU):** training from random initialization vs. fine-tuning from the best Phase I checkpoint; and RFG on/off from random initialization on both backbones. All Phase II models are evaluated on the AML validation split and the independently human-reviewed 50-image gold subset.

## Citation

If you use this code, please cite:

```bibtex
@article{tarin2026ratioaware,
  author  = {Tarin, Sumaiya Sultana and Mahdi, Mahamodul Hasan and Mobin, MD Iftekharul},
  title   = {An Ablation of Ratio-Aware Supervision for Leukocyte and {AML} Segmentation},
  year    = {2026}
}
```
## Contact

MD Iftekharul Mobin — iftekhar.mobin@aiub.edu
Department of Computer Science, American International University-Bangladesh (AIUB)
