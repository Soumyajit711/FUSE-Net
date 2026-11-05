FUSE-Net: Rethinking Skin Lesion Segmentation Without Labels

From Visual Cues to Reliable Mask Generation

FUSE-Net is a two-stage, fully unsupervised framework for dermoscopic skin-lesion segmentation. It generates confidence-aware pseudo-masks by fusing complementary appearance, semantic, and geometric edge cues, and then trains a segmentation network to refine those masks. No manual segmentation masks are used during training; ground-truth masks are reserved for evaluation.

Paper: Rethinking Skin Lesion Segmentation Without Labels: From Visual Cues to Reliable Mask Generation
Authors: Soumyajit Gayen, Rajdeep Pal, Jotiaditya Banerjee, Bala Chakravarthy Neelapu, Olga Brikova, Dmitrii Kaplun, and Ram Sarkar.
Code: github.com/Soumyajit711/FUSE-Net

------------------------------------------------------------------------

Abstract

Annotating dermoscopic lesions for segmentation is slow and subjective, while unsupervised alternatives can produce noisy pseudo-masks that a trained network may simply memorize. FUSE-Net addresses these problems through two stages. Stage 1 is training-free: appearance, semantic, and edge cues are fused according to their pixel-wise agreement to produce an initial pseudo-mask and a confidence map. Stage 2 trains a DINOv2-based feature encoder with a residual coarse-to-fine decoder. The network uses a confidence-weighted noise-robust objective together with image-affinity and edge-alignment regularization. The pseudo-mask is used only as a training target, never as a network input.

Across three dermoscopic benchmarks, FUSE-Net reports the following Dice / IoU results in the paper’s abstract and comparison table:

  Dataset                  Dice               IoU
  ----------- ----------------- -----------------
  ISIC 2017     0.7617 ± 0.0104   0.6798 ± 0.0121
  ISIC 2018     0.8370 ± 0.0098   0.7602 ± 0.0100
  PH2           0.8897 ± 0.0274   0.8242 ± 0.0246

Results are averaged over three independent runs.

Key Contributions

1.  Training-free, confidence-aware pseudo-mask generation. Appearance, self-supervised semantic, and geometric edge cues are fused through pixel-wise agreement. Cue agreement and distance from the pseudo-mask boundary provide a per-pixel confidence weight.
2.  Residual coarse-to-fine segmentation. A DINOv2-based encoder and residual decoder refine the initial mask using confidence-weighted generalized cross-entropy and Dice losses, together with image-affinity and edge-alignment regularization.
3.  Pseudo-masks are targets, not inputs. The segmentation network receives the image as input and learns to predict the generated pseudo-mask. This design is intended to reduce identity-map learning and direct copying of noisy pseudo-labels.
4.  Systematic evaluation. The paper reports comparisons with unsupervised segmentation methods, cue-level pseudo-mask ablations, and cross-dataset evaluation without target-set retraining.

Method Overview

FUSE-Net has two stages:

-   Stage 1 — Training-free pseudo-mask generation: Generate an appearance cue, a semantic cue, and an edge cue; fuse their evidence; and produce an initial pseudo-mask Y0, confidence map C0, and edge map E.
-   Stage 2 — Confidence-aware mask refinement: Train a DINOv2-based encoder-decoder using the original image as input and Y0 / C0 as supervision. A residual coarse-to-fine decoder progressively refines the predicted lesion mask.

Overall Pipeline

    flowchart TD
        A[Unlabeled dermoscopic image] --> B[Stage 1: Training-free cue generation]
        B --> C1[Appearance cue\nLab border-background model]
        B --> C2[Semantic cue\nFrozen self-supervised encoder + affinity graph]
        B --> C3[Geometric edge cue\nGradient magnitude + morphology]
        C1 --> D[Pixel-wise cue agreement and fusion]
        C2 --> D
        C3 --> D
        D --> E[Initial pseudo-mask Y0]
        D --> F[Confidence map C0]
        D --> G[Edge map E]
        A --> H[Stage 2: DINOv2 feature encoder]
        H --> I[Residual coarse-to-fine decoder]
        I --> J[Predicted lesion probability map]
        E --> K[Confidence-weighted learning objective]
        F --> K
        G --> K
        J --> K
        K --> L[Refined binary lesion mask]

Important: Y0 is a training target, not an input channel to the segmentation network. Ground-truth masks are not used for training.

Stage 1: Cue-Agreement Pseudo-mask Generation

Given an unlabeled dermoscopic image I, the training-free generator produces:

    (Y0, C0, E) = g(I)

where Y0 is the initial pseudo-mask, C0 is the relative pixel-confidence map, and E is the edge map. Three complementary cues are used.

1. Appearance Cue

The background colour is modeled from valid border pixels in CIELAB (Lab) colour space. The method estimates a background mean μ and covariance Σ. For a pixel with Lab value xp, its colour deviation is measured using a regularized Mahalanobis distance:

    d_app(p) = sqrt((xp - μ)^T (Σ + λI)^(-1) (xp - μ))

An image-adaptive sigmoid/threshold converts this distance into an appearance score C_app(p) ∈ [0, 1]. Pixels that differ more from the surrounding skin receive stronger foreground evidence.

2. Semantic Cue

A frozen self-supervised encoder extracts patch-level features. The features are L2-normalized and used to construct an affinity graph:

    W_ij = max(0, f_hat_i^T f_hat_j)
    L = I - D^(-1/2) W D^(-1/2)
    D_ii = sum_j W_ij

The Fiedler vector of the normalized graph Laplacian separates the patches into two groups. Its sign is fixed using a border-background prior, and the resulting partition is upsampled to form the semantic cue C_sem. This cue groups visually similar regions using learned features rather than colour alone.

3. Geometric Edge Cue

The gradient magnitude E is computed from a smoothed image. It is thresholded, enclosed regions are filled, and morphological cleanup is applied. A Gaussian filter smooths the resulting edge cue:

    C_edg = G_sigma * Post(Fill(1[E > t_E]))

4. Cue Fusion, Pseudo-mask, and Confidence

Let the cue set be:

    K = {app, sem, edg}

The fused score is the mean of the cue scores:

    P(x) = (1 / |K|) * sum_{k in K} C_k(x)

The fraction of cues voting foreground is:

    m(x) = (1 / |K|) * sum_{k in K} 1[C_k(x) > tau_v]

The agreement score and initial pseudo-mask are:

    A(x)  = (2m(x) - 1)^2
    Y0(x) = 1[P(x) > tau_P]

Pixel confidence combines cue agreement with the distance d_Y0(x) to the nearest opposite-label pixel:

    C0(x) = min(1, d_Y0(x) / r) * [alpha + (1 - alpha) A(x)]

Here, r is the boundary radius and alpha is the minimum weight. Pixels close to the pseudo-mask boundary or with conflicting cue evidence receive lower confidence. As specified in the paper, C0 is a relative weight, not a calibrated probability.

Stage 2: Residual Coarse-to-Fine Segmentation Network

The segmentation network combines a DINOv2 feature encoder with a residual coarse-to-fine decoder.

Encoder and Feature Fusion

-   Features from four encoder blocks are projected to a common channel width.
-   Since ViT features are spatially coarse, a lightweight convolutional detail stem supplies high-resolution features to the two finest decoder stages.
-   Decoding begins from the deepest encoder representation.

Residual Coarse-to-Fine Decoding

At each decoder stage, upsampled decoder features are fused with the corresponding encoder skip features and, at the finer stages, the high-resolution detail features. A lightweight prediction head produces a residual logit map. The new stage corrects the upsampled previous prediction instead of predicting a completely new mask.

The final probability map is:

    p = sigmoid(z_S)

Intermediate decoder logits receive deep supervision. The final binary mask is obtained by thresholding the probability map and applying the paper’s post-processing operation:

    Y_hat = Post(1[p >= tau])

Reliability-Aware Learning Objective

The segmentation network is trained using the generated pseudo-mask Y0 and confidence map C0; manual masks are not used. Let p be the predicted lesion probability and define the probability assigned to the pseudo-label as:

    p_t = p * Y0 + (1 - p) * (1 - Y0)

The objective combines four terms.

Confidence-Weighted Generalized Cross-Entropy

    L_GCE = sum_x C0(x) * (1 - p_t(x)^q)
            ---------------------------------
            q * sum_x C0(x) + epsilon

The confidence map reduces the influence of uncertain pseudo-label pixels.

Confidence-Weighted Dice Loss

    L_Dice = 1 - [2 * sum_x C0(x) * p(x) * Y0(x) + epsilon]
                 / [sum_x C0(x) * (p(x) + Y0(x)) + epsilon]

Image-Affinity Regularization

The affinity loss encourages similar-coloured neighbouring pixels to have similar predictions:

    w_ij = exp(-||c_i - c_j||^2 / (2 * sigma_c^2))

where c_i and c_j are Lab colour vectors. The loss penalizes prediction differences between neighbouring pixels in proportion to their colour affinity.

Edge-Alignment Regularization

The predicted boundary is defined as:

    B_p = min(1, |grad p|_1)

The edge loss encourages this predicted boundary to align with the generated edge cue E.

The total objective is:

    L = L_GCE + L_Dice + lambda_a * L_aff + lambda_e * L_edge

Datasets and Experimental Protocol

The paper evaluates FUSE-Net on three dermoscopic datasets. The splits are reported as follows:

  Dataset       Train images   Validation images   Test images
  ----------- -------------- ------------------- -------------
  ISIC 2017            2,000                 150           600
  ISIC 2018            2,594                 100         1,000
  PH2                    800                 100           100

Experimental protocol:

-   Images are resized to 256 × 256.
-   Validation images are used only for best-checkpoint selection, not for training.
-   Ground-truth masks are reserved for evaluation.
-   The paper reports Dice, IoU, HD95, and ASSD; HD95 and ASSD are measured in pixels at the 256 × 256 resolution.
-   Three separate models are trained, one for each dataset, without ground-truth masks.

Implementation Settings Reported in the Paper

  Setting                  Value
  ------------------------ -----------------------------
  Batch size               16
  Optimizer                AdamW
  Learning-rate schedule   Cosine decay
  Training duration        20 epochs
  Compute                  Single Kaggle NVIDIA T4 GPU
  Independent runs         3

The paper states that hyperparameters are tuned on the validation split. It does not provide all numerical hyperparameter values in the paper; consult the implementation for exact values.

Quantitative Results

Table 1. Comparison with Unsupervised Segmentation Methods

FUSE-Net’s reported results are mean ± standard deviation over three runs. Baseline values are reproduced as reported in the paper.

  ------------------------------------------------------------------------------------------------------------------------------------------
  Method                              ISIC 17 Dice       ISIC 17 IoU      ISIC 18 Dice       ISIC 18 IoU          PH2 Dice           PH2 IoU
  ------------------------------ ----------------- ----------------- ----------------- ----------------- ----------------- -----------------
  A2S-v2                                    0.7501            0.6596            0.6135            0.5110            0.7821            0.6854

  SpecWRSC                                  0.6897            0.5841            0.6101            0.5052            0.7812            0.6854

  NCut                                      0.6213            0.5184            0.6911            0.5878            0.8017            0.7051

  K-means                                   0.6778            0.5811            0.7146            0.6157            0.8614            0.7778

  SGSCN                                     0.5995            0.5004            0.7097            0.6181            0.8789            0.8015

  SLED                                      0.7390            0.6445            0.7768            0.6935            0.9034            0.8350

  RPI-Net                                   0.7584            0.6621            0.8288            0.7414            0.8931            0.8198

  FUSE-Net (training-free, Y0)     0.7206 ± 0.0102   0.6354 ± 0.0184   0.8168 ± 0.0008   0.7224 ± 0.0006   0.8442 ± 0.0242   0.7649 ± 0.0124

  FUSE-Net                         0.7617 ± 0.0104   0.6798 ± 0.0121   0.8370 ± 0.0098   0.7602 ± 0.0100   0.8897 ± 0.0274   0.8242 ± 0.0246
  ------------------------------------------------------------------------------------------------------------------------------------------

The trained network improves over its own training-free pseudo-mask baseline on all three datasets. The paper also reports that the relative importance of the three cues varies by dataset; no single cue is sufficient across all datasets.

Table 2. Stage-1 Pseudo-mask Cue Ablation

Values are Dice / IoU, reported as mean ± standard deviation over three seeds.

  -------------------------------------------------------------------------------------------------------------------------------------------
  Variant                                       ISIC 17 (Dice / IoU)                ISIC 18 (Dice / IoU)                     PH2 (Dice / IoU)
  ------------------------------ ----------------------------------- ----------------------------------- ------------------------------------
  w/o appearance cue               0.6393 ± 0.0144 / 0.5249 ± 0.0209   0.7615 ± 0.0107 / 0.6665 ± 0.0192   0.6092 ± 0.0347 / 0.4484 ± 0.04077

  w/o semantic cue                 0.6519 ± 0.0271 / 0.5506 ± 0.0239   0.7672 ± 0.0197 / 0.6711 ± 0.0202    0.6040 ± 0.0320 / 0.4784 ± 0.0398

  w/o geometric cue                0.6269 ± 0.0234 / 0.5186 ± 0.0258   0.7487 ± 0.0223 / 0.6584 ± 0.0259    0.7671 ± 0.0012 / 0.6607 ± 0.0010

  w appearance cue                 0.6307 ± 0.0267 / 0.5209 ± 0.0209   0.7654 ± 0.0268 / 0.6708 ± 0.0297    0.7282 ± 0.0192 / 0.6187 ± 0.0182

  w semantic cue                   0.5416 ± 0.0440 / 0.4370 ± 0.0398   0.6313 ± 0.0387 / 0.5244 ± 0.0400    0.5752 ± 0.0245 / 0.4478 ± 0.0253

  w geometric cue                  0.6064 ± 0.0278 / 0.4897 ± 0.0302   0.7300 ± 0.0198 / 0.6296 ± 0.0304    0.5705 ± 0.0371 / 0.4804 ± 0.0364

  FUSE-Net (training-free, Y0)     0.7206 ± 0.0102 / 0.6354 ± 0.0184   0.8168 ± 0.0008 / 0.7224 ± 0.0006    0.8442 ± 0.0242 / 0.7649 ± 0.0124
  -------------------------------------------------------------------------------------------------------------------------------------------

The paper’s ablation discussion reports that removing the geometric cue has the largest effect on ISIC 2017 and ISIC 2018. On PH2, appearance and semantic cues are more influential, while the geometric cue is nearly redundant. The single-cue configurations underperform the fused pseudo-mask on all three datasets.

Table 3. Cross-Dataset Evaluation

Each model is trained on the source dataset only and evaluated on the target set without retraining. n is the number of target images.

  --------------------------------------------------------------------------------------------------------------
  Source model   Target evaluation set                             n       Dice        IoU       HD95       ASSD
  -------------- ---------------------------------------- ---------- ---------- ---------- ---------- ----------
  ISIC 17        ISIC 18 (official test)                       1,000     0.8199     0.7440      27.15      11.60

  ISIC 17        ISIC 18 (full dataset, de-duplicated)         1,894     0.8163     0.7381      28.95      12.82

  ISIC 17        PH2 (full dataset)                              200     0.8872     0.8182      21.27       8.27

  ISIC 18        ISIC 17 (official test, de-duplicated)           71     0.6782     0.5883      47.10      23.99

  ISIC 18        ISIC 17 (full dataset, de-duplicated)           300     0.7169     0.6353      34.42      16.82

  ISIC 18        PH2 (full dataset)                              200     0.8874     0.8187      21.80       8.34

  PH2            ISIC 17 (official test)                         600     0.7072     0.6110      38.70      17.63

  PH2            ISIC 18 (official test)                       1,000     0.7715     0.6821      38.35      18.66
  --------------------------------------------------------------------------------------------------------------

These results show asymmetric transfer across datasets. In particular, models trained on either ISIC 2017 or ISIC 2018 transfer strongly to PH2, while transfer in the reverse direction is more challenging.

Qualitative Analysis and Failure Cases

The paper’s qualitative figure presents a best-case and a worst-case example:

-   Best case: Appearance, semantic, and edge cues provide complementary, spatially consistent evidence. Their agreement produces a coherent initial pseudo-mask that is close to the refined prediction.
-   Worst case: Low lesion-skin contrast and illumination artifacts weaken cue reliability and introduce misleading edge responses. Cue disagreement produces an inaccurate initial pseudo-mask that the refinement network cannot fully correct.

The paper identifies three recurring failure sources: low contrast and uneven illumination; cross-cue disagreement that fragments foreground evidence and causes under- or over-segmentation; and error propagation from unreliable initial boundaries.

Conclusion

FUSE-Net combines training-free cue-agreement pseudo-mask generation with confidence-weighted learning and residual coarse-to-fine refinement. Its central design choices are the fusion of complementary visual cues, explicit pixel-level confidence weighting, and the use of pseudo-masks only as training targets. The experiments cover ISIC 2017, ISIC 2018, and PH2, including cue ablations and cross-dataset evaluation.

The paper identifies future directions including additional self-supervised cues, joint pseudo-mask refinement during training, and evaluation on mask-scarce medical imaging modalities beyond dermoscopy.

Ethical and Funding Statements

-   Ethics: The paper uses open-access data and states that no ethical approval was required.
-   Funding: No funding was received.
-   Conflicts of interest: The authors declare no conflicts of interest.

Citation

    Soumyajit Gayen, Rajdeep Pal, Jotiaditya Banerjee, Bala Chakravarthy Neelapu,
    Olga Brikova, Dmitrii Kaplun, and Ram Sarkar,
    "Rethinking Skin Lesion Segmentation Without Labels: From Visual Cues to
    Reliable Mask Generation."

For the implementation and additional results, see: FUSE-Net GitHub repository.

------------------------------------------------------------------------

Reproducibility note: The README reproduces the main performance values from the paper’s abstract and Table 1, and the corresponding ablation and cross-dataset values from Tables 2 and 3. The paper’s prose discussion includes slightly different headline values for ISIC 2018 and PH2 than its abstract/Table 1; verify those values against the authors’ final experimental records before using them in another publication.



<img width="2000" height="409" alt="image" src="https://github.com/user-attachments/assets/43c1923b-ea29-4f8a-98f7-519b2aa58c5a" />
Figure 1: FUSE-Net overview. Stage 1 (training-free): appearance, semantic, and geometric cues are extracted from the input image and combined by confidence-weighted fusion into a fused map, from which a cue-agreement map and a confidence-guided initial pseudo-mask Y0 are derived. Stage 2 (trained): a feature encoder and a residual coarse-to-fine decoder learn to predict a refined mask ˆY.

