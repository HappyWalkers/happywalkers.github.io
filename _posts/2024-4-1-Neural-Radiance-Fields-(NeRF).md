---
title: Neural Radiance Fields (NeRF)
description: A compact NeRF-inspired reconstruction study using COLMAP camera poses, sine coordinate features, a small MLP, and differentiable ray compositing.
tags:
  - Robotics
cover: /assets/images/covers/NeRF.png
thumbnail: /assets/images/thumbnails/NeRF.jpg
---

## Abstract

This project implements a compact NeRF-inspired pipeline for reconstructing a scene from calibrated photographs. The system combines camera parameters recovered by COLMAP, ray generation, sine-based coordinate features, a multilayer perceptron, and differentiable ray compositing. The final model was trained on images downsampled to `32 x 32` pixels for 2,000 iterations. Qualitative comparisons show that the implementation recovers aspects of the coarse appearance and scene layout. However, the code differs materially from the canonical NeRF formulation in its coordinate encoding, output activations, and transmittance calculation. The experiment is therefore presented as an implementation study rather than a validation or quantitative evaluation of standard NeRF or novel-view synthesis.

## Problem Formulation

The intended objective follows [Neural Radiance Fields](https://arxiv.org/abs/2003.08934) (Mildenhall et al., 2020): represent a static scene as a continuous function rather than as an explicit mesh or a collection of stored images. For a three-dimensional location, the learned function returns four raw values interpreted as RGB color and density. A pixel is reconstructed by sampling this function along the corresponding camera ray and combining the samples with an opacity-based rule.

The implementation deliberately uses a restricted formulation. Its network receives only a three-dimensional position and does not receive the viewing direction. Consequently, the predicted color at a point is invariant to camera direction, and the model cannot explicitly reproduce view-dependent appearance such as specular reflection. In addition, the network returns unconstrained color and density values because the sigmoid color activation and positive density activation are disabled in the recorded code.

## Methodology

### Camera Calibration and Ray Construction

[COLMAP](https://openaccess.thecvf.com/content_cvpr_2016/html/Schonberger_Structure-From-Motion_Revisited_CVPR_2016_paper.html) was used to estimate the camera intrinsics and extrinsics from photographs of one scene (Schonberger and Frahm, 2016). The recovered parameters were converted to a JSON representation containing the camera field of view and one transformation matrix for each image. During data loading, each image was resized to `32 x 32`, its color values were normalized, and its focal length was adjusted to account for the change in resolution.

For each pixel, the system constructs a ray through the image plane and transforms both its origin and direction into the world coordinate frame. Each ray is sampled at 64 uniformly spaced depths between a near bound of `2` and a far bound of `6`. A single image therefore requires `32 x 32 x 64` spatial evaluations before the samples can be combined into rendered pixels.

### Neural Scene Representation

Each sampled three-dimensional position is represented by its raw coordinates together with sine features at powers-of-two frequencies. The cosine terms used in the standard NeRF positional encoding are present only as commented code and are not included in the recorded run. The encoded vector is processed by a multilayer perceptron with three hidden layers, each of width 128, and ReLU activations. Its four unconstrained outputs are interpreted as RGB color and density at the queried location.

The pointwise predictions are converted into pixels through a differentiable compositing rule. For successive samples on a ray, the distance between samples and the predicted density determines opacity. The implementation computes transmittance with an inclusive cumulative product of `(1 - opacity)` and weights each color by that transmittance and its opacity. Canonical NeRF instead uses an exclusive product over samples preceding the current sample. The distinction is important: the implemented operation remains differentiable, but it is not the standard NeRF volume-rendering equation.

## Experimental Setup

The final configuration was optimized for 2,000 iterations with Adam and a learning rate of `5e-3`. At each iteration, four training images were selected at random, rendered from their associated recovered poses, and compared with the source images using mean-squared RGB error. Random seeds were fixed for repeatability, and the trained weights were saved so that rendering could be performed without retraining.

Memory consumption was governed primarily by the number of spatial queries. The positional encodings were therefore divided into chunks before network evaluation. Their predictions were subsequently concatenated and restored to the tensor shape `[height, width, samples, 4]` for volume compositing. Downsampling the images to `32 x 32` further reduced memory and iteration time, at the cost of spatial detail.

## Results

The report presents the training loss and qualitative comparisons between source photographs and reconstructions at their corresponding camera poses. Figure 1 shows one such comparison. The saved model was also rendered along the available sequence of camera transformations to produce a video. These outputs show that the recorded implementation connected camera calibration, coordinate transformations, neural inference, and differentiable compositing sufficiently to produce low-resolution renderings; they do not verify the canonical NeRF formulation.

![Ground-truth scene and the corresponding low-resolution NeRF reconstruction](/assets/images/posts/nerf/ground-truth-and-reconstruction.jpg)

*Figure 1. Ground truth on the left and the corresponding `32 x 32` reconstruction on the right.*

The reconstruction preserves coarse scene structure and color, but the resolution is insufficient for fine boundaries and texture. The presented comparison is qualitative. Because the project did not define a held-out test set or report image-quality metrics, the results should not be interpreted as a controlled novel-view-synthesis benchmark.

## Limitations and Discussion

Five limitations determine the scope of the results. First, the `32 x 32` training resolution removes much of the information needed to assess detailed reconstruction. Second, the position-only network cannot model view-dependent radiance. Third, the coordinate features omit the cosine terms used by canonical NeRF. Fourth, the raw color and density outputs are not constrained by the intended sigmoid and positive-density activations. Fifth, the inclusive transmittance product differs from the standard exclusive formulation. The evaluation also emphasizes reconstruction from recovered camera poses and does not quantify generalization to held-out viewpoints. The output video demonstrates rendering over a sequence of camera transformations, but it does not by itself establish novel-view accuracy.

Despite these restrictions, the experiment illustrates that coordinate-based rendering depends on the consistency of several numerical components, not only on the neural network. Camera conventions, focal-length scaling, ray directions, tensor dimensions, output constraints, and compositing rules all affect the resulting optimization. The observed deviations also define concrete corrections required before evaluating the implementation as NeRF rather than as a NeRF-inspired reconstruction pipeline.

## Conclusion

The project establishes an end-to-end, NeRF-inspired reconstruction experiment from calibrated images to rendered output. Under its low-resolution setting, the recorded code produces coarse same-view reconstructions, but its noncanonical encoding, unconstrained outputs, and inclusive transmittance rule preclude treating the result as validation of standard NeRF. A corrected evaluation would restore sine-and-cosine features, apply appropriate color and density activations, use exclusive transmittance, incorporate viewing direction, reserve camera poses for testing, and report quantitative reconstruction metrics.

## References

- B. Mildenhall, P. P. Srinivasan, M. Tancik, J. T. Barron, R. Ramamoorthi, and R. Ng. [“NeRF: Representing Scenes as Neural Radiance Fields for View Synthesis.”](https://arxiv.org/abs/2003.08934) *European Conference on Computer Vision*, 2020.
- J. L. Schonberger and J.-M. Frahm. [“Structure-from-Motion Revisited.”](https://openaccess.thecvf.com/content_cvpr_2016/html/Schonberger_Structure-From-Motion_Revisited_CVPR_2016_paper.html) *IEEE Conference on Computer Vision and Pattern Recognition*, 2016.

## Project Materials

### Report

The NeRF implementation is documented as Solution 3 in the embedded homework report. The same report contains the particle-filter SLAM project discussed in the preceding post.

<div class="row">
    <iframe src="/assets/files/reports/particle-filter-slam-and-nerf.pdf" title="Particle Filter SLAM and NeRF project report" loading="lazy" style="width:100%; height:550px"></iframe>
</div>

[Open the Particle Filter SLAM and NeRF report as a PDF](/assets/files/reports/particle-filter-slam-and-nerf.pdf)
