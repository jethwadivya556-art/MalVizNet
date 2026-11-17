# MalVizNet: Malware Family Classification from 32×32 Images

Comparison of 12 CNN architectures for classifying malware into 25 families, using grayscale images of malware binaries (the Malimg dataset at 32×32). The best models reach about **94% test accuracy**. The project also documents what held accuracy back at first and what the dataset's limits are.

![Test accuracy of the 12 architectures](results/accuracy_comparison_32x32.png)

## Overview

- **Idea:** a malware file is a sequence of bytes (0-255). Each byte becomes one pixel, so a file becomes a grayscale image. Variants of the same family share code, so their images share texture patterns that a CNN can learn.
- **Task:** given a 32×32 image, predict one of 25 malware families.
- **What this project does:** trains and compares 12 architectures (M1-M12) under one protocol, then checks how reliable the results are (multiple seeds, per-class scores, a duplicate-leakage check).

## Dataset

- **Malimg**, loaded from the pre-packaged `malimg.npz` in the [Hydrogenion/Malimg](https://github.com/Hydrogenion/Malimg) repository: 9,339 images, 25 families, already 32×32. The notebook downloads it automatically.
- Only the 32×32 version was available, so the project uses a single image scale with no resizing.
- Class sizes range from 80 to 2,949 images (about 37 : 1). The two Allaple families (IDs 2 and 3) make up 48.6% of the data.
- About 21% of the images are exact duplicates (7,378 distinct images). Some families are almost entirely copies: all 800 Yuner.A (ID 24) images and all 106 Autorun.K (ID 5) images are identical.
- **About the family names.** The data file stores only numeric labels (IDs 0-24), so the **ID is the ground truth** in this project. Names are added only to make the results easier to read, for example to say which families the models confuse.
  - **How the names were assigned:** the class sizes in the file, in ID order, match the sizes in the published Malimg table (families listed alphabetically) in all 25 cases. The names are therefore very likely correct, but they are inferred and not stored in the file.
  - **Convention:** a family is written as its name followed by its ID, for example Allaple.A (ID 2). If a name is ever found to be wrong, only the name changes; the IDs and all numbers stay valid.

<details>
<summary>The 25 families (label ID, name, type, number of images)</summary>

| ID | Family | Type | Images |
|---|---|---|---|
| 0 | Adialer.C | Dialer | 122 |
| 1 | Agent.FYI | Backdoor | 116 |
| 2 | Allaple.A | Worm | 2,949 |
| 3 | Allaple.L | Worm | 1,591 |
| 4 | Alueron.gen!J | Trojan | 198 |
| 5 | Autorun.K | Worm | 106 |
| 6 | C2LOP.gen!g | Trojan | 200 |
| 7 | C2LOP.P | Trojan | 146 |
| 8 | Dialplatform.B | Dialer | 177 |
| 9 | Dontovo.A | Trojan downloader | 162 |
| 10 | Fakerean | Rogue (fake antivirus) | 381 |
| 11 | Instantaccess | Dialer | 431 |
| 12 | Lolyda.AA1 | Password stealer | 213 |
| 13 | Lolyda.AA2 | Password stealer | 184 |
| 14 | Lolyda.AA3 | Password stealer | 123 |
| 15 | Lolyda.AT | Password stealer | 159 |
| 16 | Malex.gen!J | Trojan | 136 |
| 17 | Obfuscator.AD | Trojan downloader | 142 |
| 18 | Rbot!gen | Backdoor | 158 |
| 19 | Skintrim.N | Trojan | 80 |
| 20 | Swizzor.gen!E | Trojan downloader | 128 |
| 21 | Swizzor.gen!I | Trojan downloader | 132 |
| 22 | VB.AT | Worm | 408 |
| 23 | Wintrim.BX | Trojan downloader | 97 |
| 24 | Yuner.A | Worm | 800 |

</details>

## Method

| Item | Setting |
|---|---|
| Split | Stratified 70/15/15 (train / validation / test = 6,537 / 1,401 / 1,401), `random_state=42` |
| Model selection | Best checkpoint chosen on **validation** accuracy; the test set is used only for the final evaluation |
| Augmentation | Training only: random shift (reflect-pad 3 px, random 32×32 crop). No zoom, brightness, contrast or noise |
| Class imbalance | No class weights or resampling. Reported through macro-F1 and per-class precision/recall |
| Training | EarlyStopping on validation accuracy (patience 15), best checkpoint saved |
| Batch size | 32 for M1-M8, 64 for M9-M12 |
| Optimizers | SGD with momentum for M1-M8, Adam for M9-M12 (per-model settings are in the notebook) |

**Architectures** (layer details are in the notebook)

| Models | Type |
|---|---|
| M1-M4, M7, M8 | Custom CNNs of different depths |
| M5, M6 | Custom CNNs with global max pooling |
| M9-M11 | VGG-style CNNs with BatchNorm (M10 and M11 add dropout) |
| M12 | ResNet-50 trained from scratch |

## Results

Test set (1,401 images), single run with seed 42:

| Model | Accuracy (%) | Weighted F1 | Macro F1 | Parameters |
|---|---|---|---|---|
| M3 | 94.50 | 0.941 | 0.895 | 1,070,745 |
| M8 | 94.43 | 0.942 | 0.907 | 87,705 |
| M4 | 94.08 | 0.938 | 0.895 | 1,061,497 |
| M7 | 93.79 | 0.936 | 0.893 | 275,065 |
| M1 | 93.72 | 0.933 | 0.880 | 1,061,497 |
| M2 | 92.93 | 0.926 | 0.881 | 1,052,249 |
| M11 | 92.58 | 0.912 | 0.829 | 1,446,873 |
| M10 | 92.08 | 0.904 | 0.806 | 1,446,873 |
| M9 | 89.58 | 0.876 | 0.784 | 1,446,873 |
| M6 | 82.80 | 0.816 | 0.787 | 17,017 |
| M12 | 77.66 | 0.756 | 0.700 | 26,348,441 |
| M5 | 70.95 | 0.690 | 0.576 | 7,769 |

Three seeds (42, 43, 44) for M3, M8 and M4, chosen from an earlier run's validation results (test set):

| Model | Accuracy (%) mean ± std | Range | Macro F1 mean ± std |
|---|---|---|---|
| M8 | 94.27 ± 0.18 | 94.08-94.43 | 0.901 ± 0.008 |
| M4 | 93.91 ± 0.15 | 93.79-94.08 | 0.899 ± 0.004 |
| M3 | 93.81 ± 0.96 | 92.72-94.50 | 0.888 ± 0.012 |

### Key findings

- **The top models are within noise of each other.** M3, M8, M4, M7 and M1 span about 0.8 points in a single run, which is smaller than the seed-to-seed variation. M8 is the best trade-off: it matches the top accuracy with 88k parameters and has the lowest spread.
- **Global pooling and ResNet-50 do poorly at 32×32.** M5 and M6 (global max pooling) and M12 (ResNet-50, whose feature maps shrink to 1×1 at this input size) are clearly behind.
- **Errors concentrate in six small families:** C2LOP.gen!g (ID 6), C2LOP.P (ID 7), Malex.gen!J (ID 16), Swizzor.gen!E (ID 20), Swizzor.gen!I (ID 21) and Wintrim.BX (ID 23), each with only 14-30 test images. The three best models struggle on the same families. This is why macro-F1 (about 0.90) is below accuracy (about 94%). Thirteen of the 25 families are found with recall 1.00 by all three.
- **Batch size has no consistent effect.** A check on M4 and M7 (3 seeds each) found validation differences of 0.3-0.45 points between batch 32 and 64, well inside the seed spread.
- **BatchNorm models (M9-M11) trained unstably.** Validation accuracy swung widely between epochs, though the saved checkpoints behaved consistently on the test set. A high learning rate is my best guess; I did not confirm it.

![Per-class recall and precision](results/per_class_32x32.png)

### Duplicate-leakage check

313 of the 1,401 test images (22.3%) have an exact copy in the training set, and every model classifies them correctly. On the remaining 1,088 images:

| Model | All test images (%) | Images with no training copy (%) | Macro F1 (no copy) |
|---|---|---|---|
| M3 | 94.50 | 92.92 | 0.807 |
| M8 | 94.43 | 92.83 | 0.822 |
| M4 | 94.08 | 92.37 | 0.805 |

Headline accuracy is therefore about 1.6-1.7 points higher than accuracy on unseen images. Only exact copies are counted, so near-duplicates may add to this.

## What changed along the way

The first version of the notebook scored below 80%. Fixing these brought it to about 94%:

- Zoom plus brightness, contrast and noise augmentation was replaced by plain shifts. At 32×32, interpolation blurs the texture and pixel values are the data.
- Balanced class weights were removed. They made training unstable and cost overall accuracy.
- The learning rates of two models (M1, M2) were far too small for them to learn.
- Checkpoints had been chosen on the test set. A separate validation split was added.

## Limitations

- **Resolution:** only 32×32 was available, and some families look alike at this size. Published results of 96% or more probably use larger images or splits where duplicates leak between train and test; I did not verify this.
- **Duplicates:** 22.3% of test images have a copy in training (see above).
- **Small classes:** several families have under 30 test images, so per-class scores are noisy.
- **Run-to-run variation:** GPU training is not exactly repeatable. The same seed gave results about 1 point apart across runs, so differences of 1-1.5 points between models are not meaningful.
- **Epoch limits:** M1, M2, M5, M6 and M9-M12 ran to their epoch limit (50 or 30), so some may not have fully converged.
- **Seed-run selection:** M3, M8 and M4 were chosen using an earlier run's validation scores. In the final run, M7 ranks above M3 and M4 on validation, but all four are within about 0.5 points.
- **Family names are inferred** from class sizes (see Dataset); the numeric IDs are the ground truth.
- **Closed-set classifier:** the models know only these 25 families. They have no "benign" or "unknown" class, and Malimg is a collection of older Windows malware. This is a research project, not a malware detector.

## Repository contents

```
malviznet-32x32.ipynb        # full notebook: data loading, training, evaluation
results/
  results_32x32.csv          # test results for all 12 models
  precision_recall_32x32.csv # weighted and macro precision / recall, all 12 models
  seed_runs_32x32.csv        # per-seed test results (M3, M8, M4)
  seed_summary_32x32.csv     # mean / std / min / max over seeds
  batch_size_check_32x32.csv
  per_class_32x32.csv        # per-class precision / recall / F1 (M3, M8, M4)
  duplicate_check_32x32.csv
  accuracy_comparison_32x32.png
  confusion_matrices_32x32.png   # M4, M7, M11, M12
  per_class_32x32.png
```

Trained model files are not included; running the notebook recreates them.

## How to run

1. Open the notebook in Kaggle Notebooks (where it was developed and run) with **GPU** and **Internet** turned on. The notebook downloads the dataset itself.
2. Run all cells in order. The 12 models train in about 15 minutes in total (from the logged training times); the batch-size check and the seed runs add more.
3. The cell that defines `make_datasets` and `train_model` also clears old checkpoints, so every run starts clean. Do not re-run that cell after training has started, or the saved models are deleted.

Requirements: Python, TensorFlow/Keras, scikit-learn, NumPy, pandas, Matplotlib, Seaborn.

## References

- L. Nataraj, S. Karthikeyan, G. Jacob and B. S. Manjunath, "Malware Images: Visualization and Automatic Classification," VizSec 2011 (the original Malimg dataset).
- Hydrogenion/Malimg on GitHub (the 32×32 `.npz` version used here).
- A. Pinhero et al., "Malware detection employed by visualization and deep neural network," *Computers & Security*, 105, 102247, 2021 (the architectures M1-M12 follow this prior work).
