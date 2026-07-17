---
title: Work Completed
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Implementation

## Architecture and general approach

### Research objective

The central question of this project is whether RL fine-tuning can enable **out-of-distribution (OOD) generalization** in Vision-Language-Action (VLA) models — that is, allowing a model to succeed on tasks that are maximally distant from its SFT training distribution. A secondary question is whether the primary bottleneck to RL improvement is **exploration** (the policy cannot discover reward signal) or **optimization** (the policy cannot improve given a signal).

### Pipeline overview

The project went through two major phases:

**Phase 1 — On-policy RL (Weeks 1–7):** Direct RL fine-tuning using GRPO/PPO on top of OpenVLA-OFT and π₀ in LIBERO and RoboCasa environments. This phase established the exploration bottleneck hypothesis and the limits of on-policy approaches without SFT warmup.

**Phase 2 — ExpoFT residual policy (Week 8–present):** Reproduction and adaptation of the [ExpoFT architecture](https://pd-perry.github.io/expo-ft/) into a dedicated, ManiSkill-simulation-driven codebase (`expo-ft`), replacing ExpoFT's original real-robot (DROID) data pipeline while initially keeping the algorithm itself untouched, before later diverging from it in a targeted way once instability was diagnosed (see below). The VLA (π₀.₅) is kept **frozen**, and a small residual correction policy is trained on top of it via a Q-function, using a combination of off-policy RL and motion-planning demonstrations. This design avoids two limitations of Phase 1: it does not require valid action log-probabilities (which flow-matching VLAs like π₀/π₀.₅ cannot provide, ruling out standard policy-gradient RL on the base model directly) and it bypasses the cold-start exploration problem by seeding the replay buffer with demonstrations. On-policy baselines (PPO, GRPO) and an off-policy baseline without the residual/argmax scheme (SAC) were later added to the same codebase for direct comparison (see below).

The chosen methodology is **progressive component stripping**: start from the full ExpoFT recipe (SFT warmup + demonstrations + off-policy RL) and validate each stage rigorously before removing components (dataset size, SFT warmup itself) to identify the minimum supervision actually needed.


### Working environment

- **Cluster:** Mila HPC (SLURM), A100/A100L GPUs (40GB/80GB)
- **Models tested:** OpenVLA-OFT (7B), π₀ (3B) (Phase 1); π₀.₅ (`pi05_droid_jointpos` checkpoint, frozen) + residual policy (Phase 2)
- **Environments:** LIBERO (Spatial, Long), RoboCasa (CloseDrawer) — Phase 1; ManiSkill (StackCube-v1, PushCube-v1, PickCube-v1) — Phase 2
- **RL algorithms:** GRPO, PPO (Phase 1); ExpoFT (frozen VLA + residual policy, off-policy Q-learning via a REDQ-style critic ensemble initially, later a categorical/distributional critic — see below), plus PPO, GRPO, and SAC baselines added directly to the same codebase for comparison (Phase 2)
- **Framework:** RLinf (`montrealrobotics/RLinf`) for Phase 1; standalone `expo-ft` repository (own fork of `openpi` and `ManiSkill` as git submodules) for Phase 2
- **Key infrastructure:** Four separate Python venvs to manage dependency conflicts; repository living on `$HOME` with `logs/` and `demos/` symlinked to `$SCRATCH`; large package caches relocated to `$SCRATCH`

---

## Experiments and analyses performed

### Phase 1 — On-policy RL baselines

#### LIBERO-90 GRPO (OpenVLA-OFT)
First experiment. 1000 steps (~31h40). `success_once` showed occasional spikes (~0.13) with no stable upward trend. Initially attributed to small batch size; later confirmed as an exploration bottleneck.

![LIBERO-90 GRPO Training Results Success Once]({{ config.site_url }}/images/libero90_grpo_training1_success_once.png)
![LIBERO-90 GRPO Training Results Approx KL]({{ config.site_url }}/images/libero90_grpo_training1_approx_kl.png)

#### RoboCasa CloseDrawer PPO (π₀)
150-step run with LoRA (rank 32). Final metrics: `success_once` ~0.82, `critic/value_loss` ~0.005, `approx_kl` ~0.036. Demonstrated that RL can improve performance when the task is within distribution — but CloseDrawer was too easy (SFT base already ~0.6–0.7), leaving limited room for RL contribution.

![RoboCasa CloseDrawer PPO Training Results Success Once]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_success_once.png)
![RoboCasa CloseDrawer PPO Training Results Approx KL]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_approx_kl.png)
![RoboCasa CloseDrawer PPO Training Results Value Loss]({{ config.site_url }}/images/robocasa_close_drawer_ppo_training1_value_loss.png)

<div style="text-align: center">
    <video width="480" controls poster="{{ config.site_url }}/images/robocasa_close_drawer_thumbnail.png">
        <source src="{{ config.site_url }}/images/robocasa_close_drawer_training1_ppo_step_150.mp4" type="video/mp4">
    </video>
</div>

*π₀ + RoboCasa CloseDrawer — episode at step 150 (after PPO training)*

#### LIBERO-Spatial GRPO (OpenVLA-OFT)
~38-step run. `success_once` oscillated between 0.22 and 0.875 without clear convergence. Identified as a learning rate misconfiguration following epoch reduction.

![LIBERO-Spatial GRPO Training Results Success Once]({{ config.site_url }}/images/libero_spatial_grpo_training1_success_once.png)
![LIBERO-Spatial GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_spatial_grpo_training1_approx_kl.png)

#### LIBERO-Long GRPO (OpenVLA-OFT)
~150 steps across 4 resumed sessions (~37h total). `success_once` oscillated between 0.06 and 0.19 with no upward trend. `approx_kl` remained stable at ~0.035–0.048 throughout — policy updates were controlled, confirming the bottleneck is **exploration**, not optimization instability.

![LIBERO-Long GRPO Training Results Success Once]({{ config.site_url }}/images/libero_long_grpo_training1_success_once.png)
![LIBERO-Long GRPO Training Results Approx KL]({{ config.site_url }}/images/libero_long_grpo_training1_approx_kl.png)

#### StackCube-v1 PPO and GRPO (OpenVLA base, on-policy, no SFT)
Both runs produced 0% success rate. Without SFT warmup, the base policy cannot produce any task-relevant behavior, so on-policy exploration cannot bootstrap a learning signal. This result directly motivated the pivot to ExpoFT. Here are the results for the PPO and GRPO runs on StackCube-v1:

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-ppo-nosft.mp4" type="video/mp4">
    </video>
</div>

*OpenVLA on maniskill StackCube-v1 after PPO*


<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-grpo-nosft.mp4" type="video/mp4">
    </video>
</div>

*OpenVLA on maniskill StackCube-v1 after GRPO*


### OOD dataset selection and visualization

- Created a Jupyter notebook for endpoint distribution analysis across Open X-Embodiment datasets ([Colab](https://colab.research.google.com/github/Josh012006/OpenX-Embodiment-Datasets-Visualization/blob/main/colabs/Open_X_Embodiment_Datasets.ipynb), [Repo](https://github.com/Josh012006/OpenX-Embodiment-Datasets-Visualization))
- A shared UMAP fit was used across all datasets (rather than a per-dataset fit), since per-dataset fits would make coordinates incomparable across datasets
- Identified ManiSkill as the target OOD benchmark: tasks like StackCube-v1 and PushCube-v1 are maximally distant from the VLA's SFT/pretraining data (DROID/OXE)
- Note: spatial L2 distance across datasets can be misleading due to differing coordinate frames between datasets

### Hyperparameter exploration — training time reduction (Phase 1)

Tested increasing the replay ratio (`rollout_epoch` 4→8 for GRPO, `update_epoch` 2→4 for PPO) to reduce wall-clock time. Result: the bottleneck shifted from rollout collection to gradient updates, making training *slower* (~27 min/step vs ~14 min/step). Effective approach is instead to reduce `max_episode_steps` and `max_epochs` directly.

---

### Phase 2 — ExpoFT: repository setup

The ExpoFT algorithm was reproduced in a dedicated, ManiSkill-driven repository (`~/projects/expo-ft`) rather than inside RLinf, to keep the original algorithm's code as untouched as possible while only swapping out the environment/data layer. Two dependencies — `openpi` (the π₀.₅ implementation) and `ManiSkill` — were forked and wired in as proper editable git submodules (rather than untracked/`pip`-installed dependencies), enabling direct, trackable modifications instead of runtime monkeypatches. `.gitmodules` uses HTTPS URLs for public clone compatibility, with SSH remotes configured locally for pushing.

Storage layout follows Mila's cluster guidance: the repository itself lives on `$HOME` (backed up, 100GB), while large, regenerable artifacts (`logs/`, `demos/`) are symlinked to `$SCRATCH` (5TB, auto-cleaned). Large package caches (`uv`, `openpi`, `huggingface`, `pip`, `jax`, ~90GB total) were relocated to `$SCRATCH/cache/` with symlinks back into `~/.cache`, freeing up the `$HOME` quota.

### Phase 2 — SFT stage: action-space diagnosis and validation

The first SFT warmup run (Week 8), on StackCube-v1 demonstrations converted to the DROID/LeRobot format, produced a near-0% success rate — the arm only approached the cubes without completing the stack. This was initially suspected to be a genuine task-difficulty ceiling, but a systematic investigation traced it to a **fundamental action-space mismatch**: π₀.₅'s DROID pretraining expects joint-position actions in physical units, while the pipeline was using ManiSkill's `pd_ee_delta_pose` cartesian control, causing the majority of demonstration actions to be silently clipped out of range.

The fix — switching the full pipeline to joint-space control end-to-end (`control_mode=pd_joint_delta_pos`, `pi05_droid_jointpos` checkpoint, `observation/joint_position` state, `normalize_action=True`, `output_action_dim=8`, DROID norm stats from GCS) — immediately moved the large majority of actions back in-range during replay validation.

This triggered a broader systematic review ("MAJOR RESET") that surfaced a cluster of further silent bugs before the SFT numbers could be trusted:

- An eval script unconditionally overriding the official DROID normalization stats, even for baseline evaluation
- A leftover cartesian-space action rescaling artifact from the abandoned control mode, corrupting a meaningful fraction of the now-joint-space actions
- A truncation-length mismatch between the environment and the eval loop
- A simulation backend mismatch preventing correct control-mode conversion

To regain confidence in the data pipeline, `validate_demos_full_pipeline.py` was built as a rigorous end-to-end replay tool (using the real normalization stats, the real environment wrapper, and the exact recorded seeds). After the fixes above, this tool confirmed **100% demo replay success on StackCube**, giving real confidence that the action space and data conversion were finally correct.

Proper evaluation tooling was also built at this stage: fixed episode seeds (200 held-out seeds, generated once from a master seed and reused across checkpoints) for apples-to-apples comparison between checkpoints, standard-error bars (`SE = sqrt(p(1-p)/n)`, a normal approximation rather than a Wilson interval, per the team's explicit preference), and a resumable checkpoint-sweep script that rebuilds its plot after every checkpoint.

Full SFT checkpoint sweeps on these 200 fixed seeds confirmed **real, validated generalization** (not memorization) on StackCube, with success rates increasing across SFT iterations; PushCube-v1 (added as a second, easier task for sanity-checking) performed even better.

Here are the SFT checkpoint sweep curves (success rate vs. training step/iteration, with SE error bars, 200 held-out seeds) for **StackCube-v1** and **PushCube-v1**.

![SFT Checkpoint Sweep Curves PushCube-v1]({{ config.site_url }}/images/curve-sft-pushcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pushcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 on maniskill PushCube-v1 after SFT warmup*


![SFT Checkpoint Sweep Curves StackCube-v1]({{ config.site_url }}/images/curve-sft-stackcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 on maniskill StackCube-v1 after SFT warmup*



And these are episode recordings of the best SFT checkpoint's behavior on StackCube-v1 and PushCube-v1, to visually accompany the sweep curves above (same style as the RoboCasa CloseDrawer video in Phase 1).

### Phase 2 — Task suite: StackCube, PushCube, and PickCube (dropped, then reintegrated)

PickCube-v1 was added as a third task to further diversify the OOD suite, but initially showed a persistent 0% success rate. This was traced to a **task-design incompatibility** rather than a policy or exploration problem: ManiSkill's PickCube-v1 hides the goal marker (`goal_site`) from the sensor cameras by default, and while `obs['extra']['goal_pos']` is technically accessible in the simulator, the DROID-mimicking observation schema used throughout this project (image + proprioception only, matching what a real robot would have) had no channel for it. The policy therefore had zero access to the per-episode goal position — visually or numerically — making the task appear fundamentally unsolvable.

PickCube-v1 was dropped in a first pass rather than worked around with a privileged state channel, since that would not match what a real robot could perceive. However, a monkeypatch unhiding `goal_site` from the sensor camera itself — giving the policy genuine *visual* access to the goal position, exactly as a real camera pointed at a visible target would — was subsequently validated successfully. This resolved the observability gap without compromising the DROID-mimicking observation schema, and **PickCube-v1 was reintegrated into the active task suite**.

So I also ran an SFT-warmup on PickCube-v1. Here are the performances :

![SFT Checkpoint Sweep Curves PickCube-v1]({{ config.site_url }}/images/curve-sft-pickcube.png)

<div style="text-align: center">
    <video width="480" controls>
        <source src="{{ config.site_url }}/images/pickcube-post-sft.mp4" type="video/mp4">
    </video>
</div>

*π₀.5 on maniskill PickCube-v1 after SFT warmup*


**Active task suite going forward: StackCube-v1, PushCube-v1, and PickCube-v1.**

### Phase 2 — RL stage: code review and infrastructure fixes

Before trusting any RL result, the RL training stage received the same level of line-by-line review as the SFT stage. This surfaced another cluster of real bugs:

- A norm-stats override bug identical to the one previously found in SFT evaluation
- Roughly 15 CLI flags being passed to the training script that it silently never read (orphaned hyperparameters — present in configs, never actually consumed by the code)
- A hardcoded checkpoint-overwrite flag that crashed every fresh (non-resumed) run
- A mismatch between two different copies of a key training flag, causing a crash whenever it was set to a non-default value
- Several RL hyperparameters defined in the task YAML configs that were never actually read by the training script, silently falling back to different defaults defined elsewhere in the codebase
- One hyperparameter's value being silently parsed as text instead of a float by the config reader (PyYAML parses bare scientific notation like `3e-4` as a string, not a float)

Separately, `eval_policy.py` was found to default to `only_base_actions=True`, meaning it had been silently evaluating only the frozen base VLA policy — never the trained residual policy — on every RL checkpoint evaluated so far. Proper support for loading and evaluating the full trained RL agent (residual policy and critic together) was built and used for all evaluations from this point forward.

Two distinct causes of repeated out-of-memory crashes during RL training were also diagnosed and fixed:
- A JAX recompilation issue triggered by an inconsistent input shape partway through training (an `actor_batch` `None`→dict pytree structure flip across a `jax.jit` boundary caused JAX to silently compile and keep a second XLA program resident)
- JAX's default GPU memory allocator reserving only 75% of the card by default, rather than the full card — resolved with `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95`

Finally, training-time TensorBoard logging was found to be silently dropping most training metrics (only success rate and loop time were visible, while wandb showed everything correctly) due to a type-filtering issue — JAX `jnp.float32` scalars fail Python's `isinstance(v, (int, float))` checks. Fixed with an explicit `float()` cast.

The paper's exact training data augmentation recipe was reproduced from its description and confirmed to already be correctly implemented in the codebase.

### Phase 2 — RL stage: the central instability finding

With the infrastructure and evaluation bugs fixed, and with PickCube-v1 reintegrated after its observability fix, the first real RL fine-tuning runs were launched from validated SFT checkpoints on all three active tasks. Full evaluation curves (200 held-out fixed seeds per checkpoint) were then run across a battery of comparison experiments varying the **update-to-data (UTD) ratio** and the proportion of each training batch drawn from demonstration data, including a run reproducing the paper's own exact hyperparameters within the paper's own validated training length.

**Key result:** every RL configuration tried so far — across all three active tasks (StackCube, PushCube, PickCube) and every UTD ratio / demonstration-ratio setting tested — **degrades the SFT-pretrained policy's success rate rather than improving it**:

- `utd_ratio=10`: catastrophic collapse within ~2–5k steps
- `utd_ratio=2` + `offline_ratio=0.5`: delays collapse to ~35–40k steps, eventually degrading to ~0.3–0.4 success by ~85–90k steps
- `utd_ratio=20` (the paper's exact value, run within the paper's own validated step range of ~20–24k steps): collapsed **faster and more severely** than the long `utd_ratio=2` run — this refutes the hypothesis that simply training within the paper's shorter real-robot budget would avoid the problem; `utd_ratio` itself, not total training duration, appears to be the dominant driver of the instability

The degradation is consistently preceded and accompanied by **growing instability in the critic's training loss** (progressive critic loss spikes), suggesting the root cause is a progressively destabilizing value function rather than a pipeline bug — this was further confirmed by verifying that evaluation is demonstrably using the trained residual policy and critic, not silently falling back to the frozen base policy (see the `eval_policy.py` fix above).

Confirmed hyperparameter matching against the original paper: `offline_ratio=0` (demonstrations are placed into the single online replay buffer, not a separate 50/50 offline buffer — that convention belongs to RLPD, not this paper), `hidden_dims=3×256`, `init_temperature=1.0`, `num_qs=10` / `num_min_qs=2` (REDQ-style critic ensemble), `tau=0.005`. Known deviations from the paper: `batch_size=24` (paper uses 64; constrained by available GPU memory) and a flat `edit_scale=0.2` (paper uses 0.05–0.2, scaled by task difficulty).

Here is a comparison of the training metrics across the three UTD ratio configurations described above, for PushCube-v1, the most easy task of the three :

*Critic loss evolution during training* :

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=2` + `offline_ratio=0.5*
        </td>
    <tr>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-critic-loss-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


*Residual actor loss evolution during training* :
<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-residual-actor-loss-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-residual-actor-loss-pushcube.png" alt="Curve">

            *utd_ratio=2` + `offline_ratio=0.5*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-residual-actor-loss-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


*Evaluation success rate evolution during training* :
<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config1-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=10*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=2` + `offline_ratio=0.5*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config3-eval-success-pushcube.png" alt="Curve">

            *utd_ratio=20*
        </td>
    </tr>
</table>


And this is a video showing the degradation of the policy's behavior on StackCube-v1 after ~120k steps of RL fine-tuning:
<table>
    <tr>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-sft.mp4" type="video/mp4">
                </video>
            </div>

            *Before RL fine tuning with expo*
        </td>
        <td>
            <div style="text-align: center">
                <video width="480" controls>
                    <source src="{{ config.site_url }}/images/stackcube-post-rl-degradation.mp4" type="video/mp4">
                </video>
            </div>

            *After RL fine tuning with expo*
        </td>
    </tr>
</table>

**Open research question**, raised to the group for discussion: is this a fundamental limitation of high-UTD off-policy critic training that the paper's own short real-robot runs simply don't run long enough to expose, or is it specific to this simulated setup (smaller dataset, the frozen-VLA + residual-policy architecture, or the memory-constrained batch size)? This bears directly on the project's central research question, since it currently blocks demonstrating any positive RL contribution on top of the validated SFT baseline.

**Untested leads identified for next steps:** a smaller `edit_scale` (e.g. 0.05) specifically for StackCube, since it is the most precision-demanding task and collapsed most dramatically; `replan_steps=4` for StackCube (currently a flat 8, while the paper uses 4–8 depending on task); and `batch_size=32` as a low-risk next test (never yet tried).

### Phase 2 — RL stage: isolating critic instability from a structural selection bias

The next phase of work took the leads above and systematically tested them one at a time, while also stepping back to precisely characterize *why* the critic's own training loss was destabilizing in the first place, rather than continuing to search hyperparameters.

**Mechanism identified.** ExpoFT selects the executed action by sampling several candidate actions (the base VLA's own output, plus several residual-edited variants), scoring all of them with the critic, and taking the highest-scoring one — and that same critic is then used to bootstrap its own training target for whichever candidate was picked. This is a textbook maximization/self-reference bias (closely related to the reason Double Q-learning exists): the argmax will systematically prefer whichever candidate the critic happens to overestimate, and that overestimated value then becomes part of the critic's own next training target, reinforcing the error. This mechanism was identified as the most likely root cause of the growing critic-loss instability described above, and became the organizing hypothesis for the rest of this phase of work.

**First mitigations tried, individually — none resolved the collapse:**

- Fixed-temperature ablation, freezing the critic's image encoder, gradient clipping on the critic, and critic weight decay were each tested in isolation as candidate stabilizers. None changed the qualitative outcome (`target_q_max` still grew without bound, `critic_loss` still grew increasingly spiky, `eval/success_rate` still collapsed).
- A smaller `edit_scale` (0.05, following the paper's own guidance for precision-demanding tasks) was tested on StackCube specifically. Same outcome.
- Critic pretraining — a warm-start phase, prior to any RL, that trains only the critic (never the residual policy) on demonstration data via the same Bellman-backup mechanism used later — was implemented as a YAML-toggleable option, following a similar idea proposed in the XQCfD paper (arXiv 2605.10734) for critic/actor coherence. Tested in isolation: did not resolve the collapse. Notably, `target_q_max` during this offline-only pretraining phase *itself* began climbing again partway through, on demonstration data alone — evidence that the instability is not primarily about the online data the critic sees, but about the argmax/bootstrap mechanism itself.

**Rebuilding the critic architecture (XQC / XQCfD).** To test whether the critic's own training dynamics (as opposed to the selection mechanism) were the primary driver, the critic was rebuilt from scratch following XQC (arXiv 2509.25174) and XQCfD (arXiv 2605.10734): a **categorical (C51-style, fixed bounded support)** critic head replacing the original unbounded scalar regression target, with **batch normalization and weight normalization** on the critic's MLP, and **no ensemble** (matching XQCfD's own reported setup). The reward signal is normalized by a running estimate of its own scale before being used, so the fixed value bounds stay meaningful across tasks without hand-tuning them individually. The original architecture was preserved, unmodified, in a separate module rather than overwritten, specifically so the two could be run side by side under otherwise identical conditions.

**Result:** the categorical critic's own value estimates behave in a qualitatively different, healthier way — they converge and stabilize within a bounded range instead of climbing indefinitely, and the critic's training loss stays smooth rather than growing increasingly spiky. But `eval/success_rate` still collapses in essentially the same way. This was an important negative result: it rules out critic-training instability itself as the primary driver of the policy degradation.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-target-q-max.png" alt="Curve">

            *target_q_max — original (yellow) vs. categorical (brown) critic architecture*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-critic-loss.png" alt="Curve">

            *critic_loss — original (yellow) vs. categorical (brown) critic architecture*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/critic-architecture-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — original (yellow) vs. categorical (brown) critic architecture*
        </td>
    </tr>
</table>

**A second, independent bug found and fixed: reward/termination timing.** A closer review of the online training loop found that the environment's reward and episode-termination signal were being read *before* the action for that step was actually taken, rather than after — meaning every stored transition was paired with the outcome of the *previous* action rather than the one it was actually meant to represent. This was confirmed to be a genuine, structural mismatch (not just a labeling convention) via a small controlled reproduction outside the main codebase, and traced to affect only the online RL data-collection loop — supervised fine-tuning, demonstration generation, and critic pretraining all use separate data pipelines that were never affected. The tasks used in this project rely on a shaped (dense), not sparse, reward signal, which limits how much of the smoothly-varying part of the reward this timing shift could have disrupted in practice — but the discrete success bonus layered on top of that shaping term is exactly the kind of value such a shift would misattribute most.

The fix was applied, and a second controlled before/after comparison was run under otherwise identical conditions. **Result:** the same pattern as the architecture change above — `target_q_max` stabilizes noticeably compared to the unfixed version, but `eval/success_rate` again collapses in essentially the same way regardless.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/reward-timing-fix-comparison-target-q-max.png" alt="Curve">

            *target_q_max — before (green) vs. after (red) the reward-timing fix*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/reward-timing-fix-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — before (green) vs. after (red) the reward-timing fix*
        </td>
    </tr>
</table>

**Why doesn't the original paper report this?** Revisiting the paper's own real-robot setup for an explanation: it relies on a human operator actively correcting the robot in real time during training (most heavily early on), which acts as a continuous circuit breaker on exactly the failure mode described above — a bad, overestimated action never gets the chance to be reinforced, since it gets corrected before it can be recorded. Their training runs are also considerably shorter (on the order of 8–20k steps, versus 100k+ here), and this project's own logs show the instability only becoming severe well past that range. The underlying mechanism in the paper's own formulation is identical to the one used here — nothing about the algorithm itself appears to make it immune — which points to the paper's human-in-the-loop safety net and shorter training horizon as the more likely explanation, rather than a fundamental difference in the method.

**Current leading hypothesis and next steps.** With two independent, real, and confirmed fixes each improving something measurable about the critic without resolving the actual symptom, the selection mechanism itself (the same critic proposing *and* judging its own preferred candidate) is the leading remaining explanation, by elimination. Two complementary directions from the XQCfD paper are being tested in order, to keep results attributable to a single, citable methodology before considering an alternative fix from a different line of research (decoupling the selection and evaluation steps, in the style of Double DQN, using the online and target critic networks already present in the architecture):

- **KL regularization** against the pretrained policy, replacing the generic exploration bonus used by default — computed in closed form between the residual policy's own (pre-squashing) distribution and a fixed reference distribution representing "no correction." An initial test (not yet a fully isolated one — see below) already showed the most encouraging result of this entire investigation: success rate held consistently 5–10 percentage points above a comparable run without it, for most of training, and the regularization term itself remained an active, non-trivial constraint throughout rather than being drowned out by the exploration bonus running alongside it in that first test. An isolated test (with the exploration bonus fully disabled, matching the paper's own formulation exactly) is in progress to confirm how much of that gain is attributable to the regularization term specifically.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/kl-regularization-comparison-success-rate.png" alt="Curve">

            *eval/success_rate — with (in pink) vs. without (in red) KL regularization (initial, non-isolated test)*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/kl-regularization-kl-penalty.png" alt="Curve">

            *The KL regularization term itself over the course of training*
        </td>
    </tr>
</table>

- **A "stationary" policy architecture** (not yet implemented), the remaining piece of the XQCfD recipe, intended to make the policy default to high-uncertainty, conservative behavior when acting outside the region its demonstrations actually cover, rather than continuing to act confidently.

The original, scalar-critic architecture was also made directly selectable (not just preserved as a passive reference) so it can be re-run under the exact same conditions as the categorical one on request, for further comparison.

### Phase 2 — Extending the same rigor to the PPO/GRPO baselines

On-policy baselines (PPO, GRPO) were added to the same codebase to provide additional points of comparison, sharing the same training loop and infrastructure as ExpoFT. Exercising this code for the first time surfaced several independent, confirmed numerical bugs — none previously caught, since this code path had never actually been run to completion before:

- A network-construction incompatibility between the value-function network and a shared piece of infrastructure that assumed a different (ensemble) network shape
- An action post-processing step that padded the action's dimensionality twice, once too many
- An incorrect fallback check for a quantity (the policy's entropy) that has no closed form for this project's action distribution, silently taking the wrong code path and crashing instead of using the sample-based approximation that was already implemented for exactly this situation
- An unclipped ratio term in the PPO/GRPO objective that could, after enough policy drift, silently reach floating-point infinity and corrupt an entire batch's loss
- A numerical instability at the exact boundary of the (tanh-squashed) action space, encountered when replaying certain stored actions, which could similarly corrupt a gradient step

Each of these was confirmed with a small, targeted, reproducible test built specifically to isolate it, rather than being fixed from code inspection alone. A defensive safeguard was also added on top of all of these: if a computed gradient is ever found to be non-finite for any reason, that particular update is now skipped entirely (leaving the model and optimizer state untouched) rather than being allowed to silently corrupt training, with diagnostic logging added at each stage of the computation so any future occurrence is immediately traceable rather than presenting as a bare crash much later.

**A more fundamental finding.** Investigating why these baselines showed no learning signal at all from the very start of training (rather than the "starts well, then degrades" pattern seen with ExpoFT) revealed that their trainable policy network had never actually been connected to the supervised-pretrained checkpoint in the first place: it is a small, separately and randomly initialized network, with the pretrained checkpoint used only to convert its inputs and outputs to and from the right format — not to give it any of the pretrained policy's own behavior to start from. This is a reasonable consequence of using a flow-matching base model, which does not expose the kind of action likelihood these two algorithms need directly, but it meant these baselines were effectively being asked to relearn manipulation from scratch, reproducing this project's own Phase 1 finding (on-policy RL cannot bootstrap without a competent starting policy) despite the intent being to test on-policy fine-tuning *from* the SFT checkpoint. A behavior-cloning warm-start option, mirroring the critic-pretraining idea used for ExpoFT, was added for these baselines to close this gap before drawing further conclusions from them.

---

## Major difficulties encountered

### Exploration bottleneck (Phase 1, fundamental)
On-policy algorithms (PPO, GRPO) cannot generate a learning signal when the base policy has ~0% success. This applies to LIBERO-Long (11.9% SFT base) and directly to StackCube with the base OpenVLA model. No exploration strategy (temperature, entropy bonus) is sufficient without a prior SFT warmup or demonstrations — this directly motivated the pivot to ExpoFT.

### Action-space mismatch and the MAJOR RESET (Phase 2)
The single largest source of wasted effort in Phase 2: a cartesian-vs-joint-space control mismatch produced symptoms (near-0% success, arm approaching but not completing the task) that looked identical to a genuine task-difficulty ceiling. Diagnosing it required building rigorous, ground-truth replay validation tooling rather than trusting training curves alone. Several of the further bugs found during the systematic review that followed were silent — they did not crash anything, they simply produced plausible-looking but wrong numbers — which is what made a full systematic review necessary rather than a targeted fix.

### PickCube task-design incompatibility (resolved)
A failure mode that initially looked like a policy or exploration problem (0% success, no improvement) turned out to be a fundamental observability gap between the task's design (hidden goal marker) and the project's DROID-mimicking observation schema. Required camera-level diagnostics to trace correctly, and PickCube-v1 was dropped in a first pass as a result. A subsequent monkeypatch unhiding the goal marker from the sensor camera resolved the gap without compromising the observation schema, and the task was reintegrated. Lesson generalized: verify VLA camera/observability alignment with a given ManiSkill task's design (hidden goal markers, fixed camera angle, wrist camera presence) before assuming poor results reflect task difficulty — and before assuming the incompatibility is unfixable.

### RL critic instability (Phase 2, central difficulty — substantially narrowed down since first reported)
Every RL configuration tried so far degrades rather than improves the validated SFT policy, with the degradation initially correlated with growing critic loss instability. Two independent, real fixes — a rebuilt critic architecture with structurally bounded value estimates, and a corrected reward/termination timing bug in the data-collection loop — were each confirmed to meaningfully stabilize the critic's own training dynamics, but neither changed the collapse in policy performance. This shifted the leading hypothesis away from critic-training instability itself and toward the action-selection mechanism (the same critic proposing and judging its own preferred action), which is now being tested directly rather than inferred by elimination. A useful, if initially counter-intuitive, lesson from this stretch of work: a healthier-looking critic is not the same thing as a better policy, and the two need to be checked separately rather than assumed to move together.

### Confounded first regularization test (resolved, process lesson)
The first test of KL regularization against the pretrained policy was run with the default exploration bonus still active alongside it, rather than disabled as the source paper's own formulation implies — meaning the encouraging result from that run could not yet be attributed to the regularization term specifically. Caught before drawing a conclusion from it; a properly isolated follow-up test is in progress. General lesson: when adding a term to an existing objective, explicitly check every other term it could interact with before reporting a result, rather than assuming a new option is independent of what is already there.

### PPO/GRPO baselines silently disconnected from the pretrained policy (resolved)
These baselines were assumed to be continuing from the supervised-pretrained checkpoint, the same way the primary ExpoFT method does — they were not. Their trainable policy network was a separate, randomly-initialized network the entire time, with the pretrained checkpoint used only for input/output formatting. This was not caught by any test passing or any run crashing; it only surfaced once the *results* from these baselines were examined critically (persistent near-zero performance from the very first step) rather than the code being assumed correct because it ran without error.

### Infrastructure — cluster, Ray, and storage (Phase 1 and 2)
- Model loaded twice into GPU memory (RolloutGroup + ActorGroup) → OOM on L40S (44GB) → switched to A100L (80GB) with `enable_offload: True` (Phase 1)
- Residual Ray clusters from other users blocked initialization on multiple nodes → resolved with `unset RAY_ADDRESS` and node exclusion via `#SBATCH --exclude` (Phase 1)
- `$HOME` directory too small for checkpoints (~50GB each) → migrated project to `$SCRATCH`, later re-migrated to `$HOME` with only large regenerable artifacts symlinked to `$SCRATCH`, per updated Mila storage guidance (Phase 2)
- `robosuite` dependency conflict between LIBERO and RoboCasa → resolved with two separate venvs (Phase 1)
- JAX GPU out-of-memory crashes during RL training, traced to a recompilation/pytree-structure issue and to JAX's default 75% memory pre-allocation (Phase 2)

---

## Adjustments and key decisions

| Decision | Rationale |
|---|---|
| Dropped L40S for A100L (80GB) | L40S VRAM insufficient for models loaded twice in memory |
| Dropped multi-GPU for single-GPU + `enable_offload` | NCCL timeout issues on multi-GPU; single GPU sufficient with offloading |
| Dropped LIBERO-90 | Too many simultaneous tasks; unfocused learning signal |
| Dropped OpenVLA-OFT as primary model | Pivoted to π₀ then π₀.₅ for ExpoFT (better suited to the research question) |
| Dropped LIBERO / RoboCasa as primary benchmarks | Partially saturated by SFT; limited OOD potential |
| Dropped on-policy RL from the base model | 0% success on StackCube; on-policy cannot bootstrap without SFT or demonstrations |
| Selected ManiSkill (StackCube, PushCube, PickCube) | Maximally OOD relative to π₀.₅'s DROID pretraining data |
| Pivoted to ExpoFT residual architecture | Freezes VLA, trains small residual policy via a Q-function; avoids the log-probability limitation of flow-matching models; bypasses the cold-start exploration problem via demonstrations |
| Reproduce ExpoFT in a dedicated ManiSkill repo, not inside RLinf | Keeps the original algorithm's code close to untouched; only the environment/data layer is swapped |
| `openpi` and `ManiSkill` as git submodules, not `pip` installs | Enables direct, trackable modifications instead of runtime monkeypatches |
| Full ExpoFT attempted first, components stripped afterward | Progressive stripping methodology to identify the true minimum supervision needed, rather than guessing upfront |
| Switched to joint-space control end-to-end | Matches π₀.₅'s DROID pretraining action space; cartesian control silently clipped most demonstration actions |
| Built `validate_demos_full_pipeline.py` | Ground-truth, seed-exact replay validation needed after the action-space bug showed training curves alone can't be trusted |
| Standard error (not Wilson interval) for all error bars | Per the team's (Jesse's) explicit preference |
| PickCube-v1 dropped (first pass) | Hidden-goal-marker task design initially appeared incompatible with the DROID-mimicking (image + proprioception only) observation schema |
| PickCube-v1 reintegrated | A monkeypatch unhiding the goal marker from the sensor camera restored genuine visual access to the goal position without adding a privileged state channel — the incompatibility was fixable, not fundamental |
| Repository moved from `$SCRATCH`-only to `$HOME` + symlinked artifacts | Follows updated Mila storage guidance; keeps only large, regenerable data on auto-cleaned scratch |
| `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95` adopted as standing default | Resolves recurring RL training OOM crashes tied to JAX's default 75% memory pre-allocation |
| Working default of `utd_ratio=2` (vs. paper's 20) | Paper's own ratio destabilizes training even faster than the reduced one in this simulated setup |
| Deleted catastrophic pre-fix RL runs from `$SCRATCH` | Frees disk space; keeps only validated SFT checkpoints and post-fix RL runs |
| Rebuilt the critic as a categorical (bounded-support) architecture, per XQC/XQCfD | Directly tests whether unbounded critic-value growth was the root cause of policy degradation, rather than continuing to search hyperparameters around the original architecture |
| Preserved the original critic architecture in a separate, still-runnable module rather than overwriting it | Keeps a clean A/B comparison against the categorical rewrite available at any time, rather than a one-way migration |
| Adaptive (running-estimate) reward normalization instead of fixed, hand-tuned value bounds | Keeps the categorical critic's fixed support meaningful across tasks without per-task tuning, and without needing to reproject the support mid-training |
| Testing the XQCfD paper's own mitigations (critic pretraining, categorical critic, KL regularization, stationary architecture) in order, before an alternative fix from a different line of research | Keeps results attributable to a single, citable methodology rather than mixing approaches from unrelated papers |
| KL regularization implemented as an additional, toggleable term rather than a hard replacement of the existing exploration bonus | Preserves the ability to reproduce the paper's exact formulation (by disabling the other term) while keeping both available independently for ablation |
| Added a behavior-cloning warm-start option for the PPO/GRPO baselines | Addresses the discovery that their trainable policy had no connection to the pretrained checkpoint otherwise |
| Added a defensive non-finite-gradient guard to the PPO/GRPO update step | Prevents a future, as-yet-unknown numerical issue from silently corrupting a training run, independent of whether its root cause has been found |

---

## Figures, visuals, and demonstrations

Key training curves, episode videos, and checkpoint weights for Phase 1 are embedded inline in each experiment section above. Final weights for the RoboCasa CloseDrawer PPO run are available on HuggingFace: [`josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO`](https://huggingface.co/josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO).

Phase 2 figures and videos are also integrated in the experiments descriptions above. Post-SFT weights for the Pi0.5 on the maniskill tasks are available on HuggingFace: 

- [`josh11234/ExpoFT-Pi05-StackCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-StackCube-v1-SFT) (41 %success on 200 held-out seeds)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-62p) (62 %success on 200 held-out seeds)
- [`josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p`](https://huggingface.co/josh11234/ExpoFT-Pi05-PushCube-v1-SFT-86p) (86 %success on 200 held-out seeds)
- [`josh11234/ExpoFT-Pi05-PickCube-v1-SFT`](https://huggingface.co/josh11234/ExpoFT-Pi05-PickCube-v1-SFT) (22 %success on 200 held-out seeds)

The critic-architecture comparison, reward-timing-fix comparison, and KL-regularization figures referenced in the latest updates above are still to be added — placeholders use the same `images/` convention as the rest of this page.