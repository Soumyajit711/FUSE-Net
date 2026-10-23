# RETHINKING SKIN LESION SEGMENTATION WITHOUT LABELS: FROM VISUAL CUES TO RELIABLE MASK GENERATION

*Submitted to #ISBI2027*
---

## 🎯 Abstract
Annotating dermoscopic lesions for segmentation is slow and subjective, and unsupervised alternatives suffer from noisy pseudo-masks that trained networks tend to memorize. We propose FUSE-Net, a two-stage, mask-free method. Stage 1 is training-free: an appearance cue, a semantic cue, and an edge cue are fused by agreement, and converted into a pseudo-mask with a per-pixel confidence derived from cue agreement. Stage 2 trains a feature encoder with a residual coarse-to-fine decoder using a confidence-weighted noise-robust loss together with image-affinity and edge alignment regularization, using the pseudo-mask only as a training target, never as network input. Averaged over three independent runs, FUSE-Net reaches 0.7617±0.0104 Dice 0.6798±0.0121 IoU on ISIC 17, 0.8370±0.0098 Dice 0.7602±0.0100 IoU on ISIC 18, and 0.8897±0.0274 Dice  0.8242±0.0246 IoU on PH 2. These results show that treating pseudo-masks purely as training targets, rather than network inputs, allows a single unsupervised model to generalize across datasets without any manual annotation.


<img width="2000" height="409" alt="image" src="https://github.com/user-attachments/assets/43c1923b-ea29-4f8a-98f7-519b2aa58c5a" />
Figure 1: FUSE-Net overview. Stage 1 (training-free): appearance, semantic, and geometric cues are extracted from the input image and combined by confidence-weighted fusion into a fused map, from which a cue-agreement map and a confidence-guided initial pseudo-mask Y0 are derived. Stage 2 (trained): a feature encoder and a residual coarse-to-fine decoder learn to predict a refined mask ˆY.

