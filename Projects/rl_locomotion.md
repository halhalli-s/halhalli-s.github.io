---
layout: default
title: Quadruped Locomotion with Deep RL
---

# Quadruped Locomotion with Deep RL — SAC vs PPO on MuJoCo Ant-v5

**CS5180 Reinforcement Learning · Northeastern University · Jan 2026 – Apr 2026**  
**Stack:** Python · MuJoCo · Gymnasium · Stable-Baselines3 · SAC · PPO · NVIDIA RTX 5070 Ti

---

## What This Is

A comparative study of two deep RL algorithms — Soft Actor-Critic (SAC, off-policy) and Proximal Policy Optimization (PPO, on-policy) — on the task of quadruped locomotion using the Gymnasium Ant-v5 environment. I implemented the SAC side; my partner Ahilesh Vadivel implemented PPO.

The project goes beyond a baseline comparison: we ran reward shaping ablations with multiple variants per algorithm, diagnosed an emergent reward exploitation failure, and extended PPO to terrain adaptation via domain randomization across six custom environments.

---

## The Task

The Ant-v5 agent has a 105-dimensional state space (torso pose, joint angles/velocities, contact forces) and an 8-dimensional continuous action space (torque commands at hip and ankle joints per leg). The base reward is:

```
r(s,a) = v_x − 0.5‖a‖² − c_contact + 1_alive
```

Forward velocity rewarded, high torques and contact forces penalized, +1 survival bonus per step. Episodes terminate when torso height exits [0.2, 1.0]m or 1000 steps elapse.

---

## SAC Implementation

<div style="display:grid; grid-template-columns:1fr 1fr; gap:1rem; margin:1.5rem 0;">
  <div>
    <p style="text-align:center; font-weight:bold; margin-bottom:0.3rem;">Baseline</p>
    <img src="../Assets/images/rl_baseline.gif" alt="Baseline gait" style="width:100%;">
  </div>
  <div>
    <p style="text-align:center; font-weight:bold; margin-bottom:0.3rem;">Speed Focused</p>
    <img src="../Assets/images/rl_speed.gif" alt="Speed focused gait" style="width:100%;">
  </div>
  <div>
    <p style="text-align:center; font-weight:bold; margin-bottom:0.3rem;">Energy Efficient</p>
    <img src="../Assets/images/rl_energy.gif" alt="Energy efficient gait" style="width:100%;">
  </div>
  <div>
    <p style="text-align:center; font-weight:bold; margin-bottom:0.3rem;">Symmetric Gait</p>
    <img src="../Assets/images/rl_symmetric.gif" alt="Symmetric gait" style="width:100%;">
  </div>
</div>

SAC (Haarnoja et al. 2018) is an off-policy maximum entropy actor-critic. It optimizes:

```
J(π) = Σ E[r(s,a) + α·H(π(·|s))]
```

The entropy term α·H encourages exploration and prevents premature convergence. Two Q-critics (minimum of both used during updates) mitigate overestimation bias.

**Architecture:** Actor MLP 256×256 with tanh-squashed Gaussian output. Each Q-critic takes concatenated state-action (113D) with 256×256 hidden layers.  
**Training:** 1M timesteps, 3 seeds, RTX 5070 Ti (~3 hrs/seed). One gradient update per environment step from a replay buffer of 10⁶ transitions.

### Baseline Results (3 seeds)

| Metric | Value |
|---|---|
| Mean ± std reward | 3446 ± 733 |
| Peak reward | 3795 @ 930k steps |
| Steps to stable walking | ~180k |
| Final episode length | 868.9 / 1000 steps |

Seed variance was substantial (±733). Seed 0 had a catastrophic reward collapse at ~850k steps but recovered to the highest final reward of 4460 — the entropy term kept the stochastic policy exploring diverse actions during the collapse rather than locking into a failed mode.

---

## Reward Shaping Ablations (SAC)

Three variants, 1M timesteps each:

**Speed-focused:** 3× forward velocity weight  
**Energy-efficient:** 4× control cost penalty  
**Symmetric gait:** Added bilateral asymmetry penalty across left-right joint pairs (λ=0.5)

| Variant | Final Reward | vs. Baseline |
|---|---|---|
| Baseline | 3446 | — |
| Energy Efficient | 4068 | +18% |
| Symmetric Gait | 5166 | +50% |
| Speed Focused | 10065 | +192%* |

*Speed Focused's raw number reflects a 3× velocity weight scaling artifact, not proportionally faster locomotion.

**Symmetric Gait was the most principled gain.** A hand-designed biomechanical constraint that was expected to *constrain* performance instead guided SAC toward a more stable gait strategy. SAC's entropy maximization made it highly responsive to this additional structure.

---

## PPO Comparison

PPO (Schulman et al. 2017) collects a full 2048-step rollout before any gradient update, then discards all experience. This on-policy discipline ensures unbiased gradient estimates at the cost of sample efficiency.

**Architecture:** 64×64 MLP for both actor and critic (significantly smaller than SAC's 256×256).

| Metric | 1M steps |
|---|---|
| Final reward | 2198 |
| Steps to stable walking | ~300k |

SAC reached stable walking 1.7× faster than PPO, directly attributable to continuous replay buffer reuse.

---

## Reward Exploitation Failure — Goodhart's Law in RL

The most interesting result: PPO's symmetric gait variant (Eq. 10 — exponential symmetry bonus) achieved numerically high reward but **learned to walk upside-down**.

The policy discovered it could collect the symmetry bonus, forward velocity reward, and survival bonus simultaneously while inverted, because the reward function lacked an orientation constraint. This is a textbook instance of Goodhart's Law: the policy optimized the specified reward correctly, but the reward failed to capture the intended behavior.

**Diagnosis:** Video analysis of rollouts.  
**Fix:** Quaternion-based orientation constraint — harsh penalty (−10.0) when qw < 0.9 caused complete learning paralysis; calibrated penalty (−0.5) successfully prevented inversion while preserving locomotion.

This required iterative reward engineering: specify → train → diagnose → fix → retrain.

---

## Terrain Adaptation (PPO Domain Randomization)

Six custom MuJoCo environments built by modifying ground geometry in the Ant XML: flat, slope 5°, slope 10°, slope 15°, rough terrain (44 scattered box obstacles), stairs (5 steps, 0.05m rise).

**Flat-only policy zero-shot transfer:**

| Terrain | Reward |
|---|---|
| Flat | 1844 |
| Slope 5° | 576 |
| Slope 10° | 202 |
| Slope 15° | 114 |
| Rough | 2060 |
| Stairs | 1484 |

Sharp degradation on slopes — gravity vector shift requires qualitatively different joint torque strategies absent from flat-ground training. Surprisingly good transfer to rough terrain and stairs.

**Fine-tuned terrain policy:** Achieved ~960 reward uniformly across all terrains — but video showed the agent learned to *stand still*, collecting the survival bonus without attempting locomotion. Conservative fine-tuning hyperparameters (lr=1e-4, clip=0.1) intended to prevent catastrophic forgetting instead prevented meaningful adaptation entirely.

---

## Key Takeaways

**SAC vs PPO:** The replay buffer is the decisive factor for sample efficiency in continuous control. SAC's 1.7× advantage over PPO at stable-walking threshold is directly explained by continuous buffer reuse.

**Reward shaping matters more than architecture:** Both algorithms are highly sensitive to objective design. An underspecified constraint (symmetry bonus without orientation check) caused a complete behavioral failure. A well-specified constraint (bilateral symmetry penalty in SAC) produced the strongest improvement.

**Sim-to-sim transfer is non-trivial:** A policy that handles rough terrain and stairs zero-shot completely fails on 15° slopes. Environment geometry that changes the effective gravity direction requires fundamentally different control strategies, not just generalization.
