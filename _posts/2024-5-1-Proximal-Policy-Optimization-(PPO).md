---
title: Proximal Policy Optimization (PPO)
description: Implementing PPO for DeepMind Control's planar walker with GAE, clipped objectives, mini-batch updates, entropy regularization, and training diagnostics.
tags:
  - Robotics
cover: /assets/images/covers/PPO.png
thumbnail: /assets/images/thumbnails/PPO.jpg
---

## Abstract

This project implements Proximal Policy Optimization (PPO) for the continuous-control `walker/walk` task in DeepMind Control Suite. A PyTorch PPO baseline was adapted to the environment and extended with Generalized Advantage Estimation, mini-batch optimization, learning-rate annealing, gradient clipping, approximate-KL early stopping, checkpointing, and return diagnostics. The trained policy produced walking behavior in simulation. In the recorded comparison, the run with a zero entropy coefficient did not exhibit the mid-training return decline observed in the initial run. Because this comparison used single runs rather than a multi-seed ablation, it identifies a configuration-specific observation rather than a general causal result.

## Objective

The objective was to train a planar simulated walker directly from on-policy interaction while examining the implementation choices required for stable [Proximal Policy Optimization](https://arxiv.org/abs/1707.06347) (PPO; Schulman et al., 2017). Although PPO has a compact mathematical objective, practical performance depends on the interface between the simulator and algorithm, the construction of rollout batches, the estimation of advantages, and safeguards that constrain each policy update.

The implementation began with a small PyTorch PPO codebase and introduced the components required for DeepMind Control. The resulting system separates environment adaptation, trajectory collection, policy and value estimation, optimization, checkpointing, and evaluation. This organization permitted individual training behaviors, including the effect associated with the entropy coefficient, to be inspected from logged returns and saved policies.

## Methodology

### Environment and Policy Representation

DeepMind Control and Gym expose different environment interfaces. A wrapper was therefore implemented to flatten the walker's orientation, height, and velocity observations into a single vector, convert actions and rewards into the Gym-style format expected by the training loop, and expose the continuous observation and action dimensions.

The policy and value functions were represented by separate feed-forward networks with 64-unit hidden layers. The actor parameterized the mean of a multivariate Gaussian action distribution, while the critic estimated the scalar value of the current observation. During interaction, actions were sampled from the Gaussian. Each rollout retained the observations, actions, old action log probabilities, rewards, value estimates, episode boundaries, and episode lengths required for the subsequent update.

### PPO Objective and Optimization

The training configuration collected 2,048 time steps per batch and limited each episode to 1,024 steps. Future rewards were discounted by `0.99`. Advantages were estimated using [Generalized Advantage Estimation](https://arxiv.org/abs/1506.02438) (Schulman et al., 2015) with `lambda = 0.98` and were normalized before optimization.

For each stored action, the PPO update computes the ratio between its probability under the current policy and its probability under the behavior policy that generated the rollout. The actor optimizes the minimum of the ordinary surrogate objective and the surrogate with the probability ratio clipped to `[0.8, 1.2]`. This clipping limits the benefit of a large change to the policy on a fixed rollout. The critic minimizes mean-squared error with respect to the computed return target.

The rollout indices were shuffled and divided into two mini-batches, with four optimization passes over each rollout. The initial learning rate of `1e-3` was annealed during training. The gradient norm was clipped at `0.5`, and an update pass terminated early when the approximate Kullback–Leibler divergence exceeded `0.02`. These mechanisms were intended to reduce abrupt optimization steps and to provide additional checks on policy drift.

## Experimental Setup

The principal experiment trained the policy on the DeepMind Control `walker/walk` task and logged episodic return against environment steps. Actor and critic checkpoints were saved during training, and the learned actor was subsequently loaded without the optimizer for evaluation. The report includes the resulting walking behavior.

An additional configuration comparison examined entropy regularization. The initial actor objective subtracted the entropy coefficient multiplied by mean policy entropy from the clipped surrogate loss. The initial training trace exhibited a substantial return decline during training. A subsequent configuration set the entropy coefficient to zero while retaining the remainder of the final training setup. This second trace continued to improve without the same observed decline.

## Results

The trained policy generated walking behavior in the simulator, establishing that the adapted environment interface, rollout implementation, advantage computation, and PPO update operated together successfully. The saved checkpoints also separated training from policy evaluation and made the qualitative behavior reproducible from a fixed learned actor. Figure 1 compares the two recorded entropy configurations.

![PPO return curves before and after disabling entropy regularization](/assets/images/posts/ppo/entropy-comparison.jpg)

*Figure 1. Individual PPO training traces with entropy regularization enabled (left) and disabled (right). The first trace declines during training, whereas the second continues to improve.*

The trace with a zero entropy coefficient did not exhibit the return decline seen in the initial trace. This observation supports that setting for the reported run, but the single-run comparison does not isolate the effect of entropy regularization. Entropy may be beneficial under different coefficients, random seeds, policy initializations, or task conditions.

## Limitations and Discussion

The main limitation is the absence of repeated trials. The entropy comparison consists of one trace for each configuration, so stochastic variation cannot be separated from the effect of the entropy coefficient. The experiment also reports simulator return and qualitative walking behavior rather than a broader evaluation across tasks or perturbations. Accordingly, the results characterize one implementation and one environment rather than PPO performance in general.

Within this scope, the training behavior illustrates that PPO's clipped objective is only one component of a complete control system. Observation conversion, episode termination, rollout bookkeeping, advantage targets, batch construction, exploration pressure, gradient control, and KL monitoring all influence the data presented to the optimizer. The mid-training decline was therefore treated as a diagnostic observation, and disabling entropy was a configuration change supported by the reported run rather than a universal prescription.

## Conclusion

The project produced an operational PPO pipeline for continuous planar locomotion in DeepMind Control. Its primary contribution is an end-to-end implementation that integrates simulator adaptation, on-policy data collection, GAE, clipped policy optimization, mini-batch updates, and multiple optimization safeguards. The reported entropy comparison provides a useful implementation-specific diagnosis, but its single-run design precludes a general conclusion. A stronger evaluation would repeat each configuration across random seeds, report aggregate return statistics, and vary the entropy coefficient while holding the remaining hyperparameters fixed.

## References

- J. Schulman, F. Wolski, P. Dhariwal, A. Radford, and O. Klimov. [“Proximal Policy Optimization Algorithms.”](https://arxiv.org/abs/1707.06347) arXiv:1707.06347, 2017.
- J. Schulman, P. Moritz, S. Levine, M. Jordan, and P. Abbeel. [“High-Dimensional Continuous Control Using Generalized Advantage Estimation.”](https://arxiv.org/abs/1506.02438) *International Conference on Learning Representations*, 2016.

## Project Materials

### Report

The PPO implementation and experiment are documented as Solution 2 in the embedded report. The same report also contains short demonstrations of MuJoCo Model Predictive Control.

<div class="row">
    <iframe src="/assets/files/reports/ppo.pdf" title="Proximal Policy Optimization project report" loading="lazy" style="width:100%; height:550px"></iframe>
</div>

[Open the Proximal Policy Optimization report as a PDF](/assets/files/reports/ppo.pdf)
