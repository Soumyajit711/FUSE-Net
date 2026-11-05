## Rethinking Skin Lesion Segmentation Without Labels: From Visual Cues to Reliable Mask Generation

FUSE-Net is a two-stage, fully unsupervised framework for dermoscopic skin-lesion segmentation. It generates confidence-aware pseudo-masks by fusing complementary appearance, semantic, and geometric edge cues, and then trains a segmentation network to refine those masks. No manual segmentation masks are used during training; ground-truth masks are reserved for evaluation.

Paper: Rethinking Skin Lesion Segmentation Without Labels: From Visual Cues to Reliable Mask Generation
Authors: Soumyajit Gayen, Rajdeep Pal, Jotiaditya Banerjee, Bala Chakravarthy Neelapu, Olga Brikova, Dmitrii Kaplun, and Ram Sarkar

------------------------------------------------------------------------

## Abstract
Annotating dermoscopic lesions for segmentation is slow and subjective, and unsupervised alternatives suffer from noisy pseudo-masks that trained networks tend to memorize. We propose FUSE-Net, a two-stage, mask-free method. Stage 1 is training-free: an appearance cue, a semantic cue, and an edge cue are fused by agreement and converted into a pseudo-mask with a per-pixel confidence derived from cue agreement. Stage 2 trains a feature encoder with a residual coarse-to-fine decoder using a confidence-weighted noise-robust loss together with image-affinity and edge alignment regularization, using the pseudo-mask only as a training target, never as network input. Averaged over three independent runs, FUSE-Net reaches $0.7617{\pm}0.0104$ Dice /$0.6798{\pm}0.0121$ IoU on ISIC 17, $0.8370{\pm}0.0098$ Dice / $0.7602{\pm}0.0100$ IoU on ISIC 18, and $0.8897{\pm}0.0274$ Dice / $0.8242{\pm}0.0246$ IoU on PH 2. These results show that treating pseudo-masks purely as training targets, rather than network inputs, allows a single unsupervised model to generalize across datasets without any manual annotation.

## Key Contributions

1.  Training-free, confidence-aware pseudo-mask generation. Appearance, self-supervised semantic, and geometric edge cues are fused through pixel-wise agreement. Cue agreement and distance from the pseudo-mask boundary provide a per-pixel confidence weight.
2.  Residual coarse-to-fine segmentation. A DINOv2-based encoder and residual decoder refine the initial mask using confidence-weighted generalized cross-entropy and Dice losses, together with image-affinity and edge-alignment regularization.
3.  Pseudo-masks are targets, not inputs. The segmentation network receives the image as input and learns to predict the generated pseudo-mask. This design is intended to reduce identity-map learning and direct copying of noisy pseudo-labels.
4.  Systematic evaluation. The paper reports comparisons with unsupervised segmentation methods, cue-level pseudo-mask ablations, and cross-dataset evaluation without target-set retraining.

## Method Overview

FUSE-Net has two stages:

(i) Stage 1: Training-free pseudo-mask generation: Generate an appearance cue, a semantic cue, and an edge cue; fuse their evidence; and produce an initial pseudo-mask Y0, confidence map C0, and edge map E.
(ii) Stage 2: Confidence-aware mask refinement: Train a DINOv2-based encoder-decoder using the original image as input and Y0 / C0 as supervision. A residual coarse-to-fine decoder progressively refines the predicted lesion mask.

Important: Y0 is a training target, not an input channel to the segmentation network. Ground-truth masks are not used for training.

<img width="1205" height="275" alt="image" src="https://github.com/user-attachments/assets/c600bc0f-9361-4ad1-a3cb-52255bca0eb5" />
**Figure 1:** FUSE-Net overview: Stage 1 (training-free) fuses appearance, semantic, and geometric cues to generate a confidence-guided initial pseudo-mask Y0. Stage 2 (trained) uses a feature encoder and residual coarse-to-fine decoder to predict the refined mask ˆY.

### Stage 1: Cue-Agreement Pseudo-mask Generation

Given an unlabeled dermoscopic image \(I\), a training-free generator produces an initial pseudo-mask \(Y_0\), a pixel-wise confidence map \(C_0\), and an edge map \(E\): \[(Y_0,C_0,E)=g(I).\]

Three complementary cues are fused through pixel-wise agreement: **(i) Appearance Cue**, capturing lesion–background color differences; **(ii) Semantic Cue**, exploiting self-supervised visual similarities; and **(iii) Geometric Edge Cue**, capturing lesion boundary information. Their agreement determines the initial pseudo-mask and its relative pixel confidence.

### Stage 2: Residual Coarse-to-Fine Segmentation Network

The segmentation network combines a **DINOv2 encoder** with a residual coarse-to-fine decoder.

**Encoder and Feature Fusion:** Features from four encoder blocks are projected into a common channel space. A lightweight convolutional detail stem supplements the coarse ViT representations with high-resolution features for the two finest decoder stages.

**Residual Coarse-to-Fine Decoding:** Starting from the deepest encoder features, each decoder stage progressively upsamples and fuses the preceding decoder representation with the corresponding encoder features and, where applicable, high-resolution detail features. A lightweight prediction head generates residual logits that refine the upsampled previous prediction, progressively improving lesion localization and boundary details rather than predicting an entirely new mask at each stage.

## Datasets and Experimental Protocol

The paper evaluates FUSE-Net on three dermoscopic datasets. The splits are reported as follows:

| Dataset | Training Images | Validation Images | Testing Images |
|:---|---:|---:|---:|
| ISIC 2017 | 2,000 | 150 | 600 |
| ISIC 2018 | 2,594 | 100 | 1,000 |
| PH2 | 800 | 100 | 100 |                  800                 100           100

## Experimental protocol:

-   Images are resized to 256 × 256.
-   Validation images are used only for best-checkpoint selection, not for training.
-   Ground-truth masks are reserved for evaluation.
-   The paper reports Dice, IoU, HD95, and ASSD; HD95 and ASSD are measured in pixels at the 256 × 256 resolution.
-   Three separate models are trained, one for each dataset, without ground-truth masks.

## Quantitative Results

## Quantitative Results

**Table 1.** Comparison with unsupervised segmentation methods. FUSE-Net results are reported as mean ± standard deviation over three independent runs. Baseline values are reproduced as reported in the original papers.

| Method | ISIC 17 Dice | ISIC 17 IoU | ISIC 18 Dice | ISIC 18 IoU | PH2 Dice | PH2 IoU |
|:---|---:|---:|---:|---:|---:|---:|
| A2S-v2 | 0.7501 | 0.6596 | 0.6135 | 0.5110 | 0.7821 | 0.6854 |
| SpecWRSC | 0.6897 | 0.5841 | 0.6101 | 0.5052 | 0.7812 | 0.6854 |
| NCut | 0.6213 | 0.5184 | 0.6911 | 0.5878 | 0.8017 | 0.7051 |
| K-means | 0.6778 | 0.5811 | 0.7146 | 0.6157 | 0.8614 | 0.7778 |
| SGSCN | 0.5995 | 0.5004 | 0.7097 | 0.6181 | 0.8789 | 0.8015 |
| SLED | 0.7390 | 0.6445 | 0.7768 | 0.6935 | **0.9034** | **0.8350** |
| RPI-Net | 0.7584 | 0.6621 | 0.8288 | 0.7414 | 0.8931 | 0.8198 |
| FUSE-Net (Training-free, Y₀) | 0.7206 ± 0.0102 | 0.6354 ± 0.0184 | 0.8168 ± 0.0008 | 0.7224 ± 0.0006 | 0.8442 ± 0.0242 | 0.7649 ± 0.0124 |
| **FUSE-Net (Ours)** | **0.7617 ± 0.0104** | **0.6798 ± 0.0121** | **0.8370 ± 0.0098** | **0.7602 ± 0.0100** | 0.8897 ± 0.0274 | 0.8242 ± 0.0246 |

The trained network improves over its own training-free pseudo-mask baseline on all three datasets. The paper also reports that the relative importance of the three cues varies by dataset; no single cue is sufficient across all datasets.

**Table 2.** Ablation study of individual cues in the Stage-1 pseudo-mask generation module. Results are reported as Dice / IoU (mean ± standard deviation over three independent seeds).

| Variant | ISIC 17 (Dice / IoU) | ISIC 18 (Dice / IoU) | PH2 (Dice / IoU) |
|:---|:---:|:---:|:---:|
| w/o appearance cue | 0.6393 ± 0.0144 / 0.5249 ± 0.0209 | 0.7615 ± 0.0107 / 0.6665 ± 0.0192 | 0.6092 ± 0.0347 / 0.4484 ± 0.04077 |
| w/o semantic cue | 0.6519 ± 0.0271 / 0.5506 ± 0.0239 | 0.7672 ± 0.0197 / 0.6711 ± 0.0202 | 0.6040 ± 0.0320 / 0.4784 ± 0.0398 |
| w/o geometric cue | 0.6269 ± 0.0234 / 0.5186 ± 0.0258 | 0.7487 ± 0.0223 / 0.6584 ± 0.0259 | 0.7671 ± 0.0012 / 0.6607 ± 0.0010 |
| w appearance cue | 0.6307 ± 0.0267 / 0.5209 ± 0.0209 | 0.7654 ± 0.0268 / 0.6708 ± 0.0297 | 0.7282 ± 0.0192 / 0.6187 ± 0.0182 |
| w semantic cue | 0.5416 ± 0.0440 / 0.4370 ± 0.0398 | 0.6313 ± 0.0387 / 0.5244 ± 0.0400 | 0.5752 ± 0.0245 / 0.4478 ± 0.0253 |
| w geometric cue | 0.6064 ± 0.0278 / 0.4897 ± 0.0302 | 0.7300 ± 0.0198 / 0.6296 ± 0.0304 | 0.5705 ± 0.0371 / 0.4804 ± 0.0364 |
| **FUSE-Net (Training-free, Y₀)** | **0.7206 ± 0.0102 / 0.6354 ± 0.0184** | **0.8168 ± 0.0008 / 0.7224 ± 0.0006** | **0.8442 ± 0.0242 / 0.7649 ± 0.0124** |

The paper’s ablation discussion reports that removing the geometric cue has the largest effect on ISIC 2017 and ISIC 2018. On PH2, appearance and semantic cues are more influential, while the geometric cue is nearly redundant. The single-cue configurations underperform the fused pseudo-mask on all three datasets.

**Table 3.** Cross-dataset evaluation of FUSE-Net. Each model is trained exclusively on the source dataset and evaluated on the target dataset without retraining. `n` denotes the number of target images.

| Source Model | Target Evaluation Set | n | Dice | IoU | HD95 | ASSD |
|:---|:---|---:|---:|---:|---:|---:|
| ISIC 17 | ISIC 18 (official test) | 1,000 | 0.8199 | 0.7440 | 27.15 | 11.60 |
| ISIC 17 | ISIC 18 (full dataset, de-duplicated) | 1,894 | 0.8163 | 0.7381 | 28.95 | 12.82 |
| ISIC 17 | PH2 (full dataset) | 200 | **0.8872** | **0.8182** | **21.27** | **8.27** |
| ISIC 18 | ISIC 17 (official test, de-duplicated) | 71 | 0.6782 | 0.5883 | 47.10 | 23.99 |
| ISIC 18 | ISIC 17 (full dataset, de-duplicated) | 300 | 0.7169 | 0.6353 | 34.42 | 16.82 |
| ISIC 18 | PH2 (full dataset) | 200 | **0.8874** | **0.8187** | **21.80** | **8.34** |
| PH2 | ISIC 17 (official test) | 600 | 0.7072 | 0.6110 | 38.70 | 17.63 |
| PH2 | ISIC 18 (official test) | 1,000 | 0.7715 | 0.6821 | 38.35 | 18.66 |

These results show asymmetric transfer across datasets. In particular, models trained on either ISIC 2017 or ISIC 2018 transfer strongly to PH2, while transfer in the reverse direction is more challenging.

## Qualitative Analysis and Failure Cases

The paper’s qualitative figure presents a best-case and a worst-case example:

-   Best case: Appearance, semantic, and edge cues provide complementary, spatially consistent evidence. Their agreement produces a coherent initial pseudo-mask that is close to the refined prediction.
-   Worst case: Low lesion-skin contrast and illumination artifacts weaken cue reliability and introduce misleading edge responses. Cue disagreement produces an inaccurate initial pseudo-mask that the refinement network cannot fully correct.

<img width="760" height="275" alt="image" src="https://github.com/user-attachments/assets/f376de7a-c9a5-4875-9214-1393a8d3b0e3" />
**Figure 2:** Results on best-case and worst-case test case.


The paper identifies three recurring failure sources: low contrast and uneven illumination; cross-cue disagreement that fragments foreground evidence and causes under- or over-segmentation; and error propagation from unreliable initial boundaries.

## Conclusion

FUSE-Net combines training-free cue-agreement pseudo-mask generation with confidence-weighted learning and residual coarse-to-fine refinement. Its central design choices are the fusion of complementary visual cues, explicit pixel-level confidence weighting, and the use of pseudo-masks only as training targets. The experiments cover ISIC 2017, ISIC 2018, and PH2, including cue ablations and cross-dataset evaluation.

The paper identifies future directions, including additional self-supervised cues, joint pseudo-mask refinement during training, and evaluation on mask-scarce medical imaging modalities beyond dermoscopy.

## Ethical and Funding Statements

-   Ethics: The paper uses open-access data and states that no ethical approval was required.
-   Funding: No funding was received.
-   Conflicts of interest: The authors declare no conflicts of interest.

## Citation

    Soumyajit Gayen, Rajdeep Pal, Jotiaditya Banerjee, Bala Chakravarthy Neelapu,
    Olga Brikova, Dmitrii Kaplun, and Ram Sarkar,
    "Rethinking Skin Lesion Segmentation Without Labels: From Visual Cues to
    Reliable Mask Generation."

------------------------------------------------------------------------



<img width="2000" height="409" alt="image" src="https://github.com/user-attachments/assets/43c1923b-ea29-4f8a-98f7-519b2aa58c5a" />
Figure 1: FUSE-Net overview. Stage 1 (training-free): appearance, semantic, and geometric cues are extracted from the input image and combined by confidence-weighted fusion into a fused map, from which a cue-agreement map and a confidence-guided initial pseudo-mask Y0 are derived. Stage 2 (trained): a feature encoder and a residual coarse-to-fine decoder learn to predict a refined mask ˆY.

