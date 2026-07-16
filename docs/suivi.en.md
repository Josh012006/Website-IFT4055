---
title: Project Tracking
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Project Tracking

## Week 1 (May 4–9, 2026)

### Goals for the period
- Get in touch with the team and set up the work environment
- Launch a first RL training run on the Mila cluster
- Identify and resolve infrastructure issues

### Work completed

!!! abstract "Progress"
    - [x] Connected with the team, joined the lab's Slack
    - [x] Read the RLinf documentation (LIBERO)
    - [x] Connected to the Mila cluster and resolved access issues with IT support
    - [x] Cloned the repository and installed dependencies on an interactive compute node
    - [x] Wrote the `job.sh` script and submitted it via `sbatch`
    - [x] Progressively resolved infrastructure errors (normalization key, CPU/GPU RAM, NCCL timeout)
    - [x] First successful training run reaching 132 steps
    - [x] Migrated project to `$SCRATCH`, created symbolic link from `$HOME`, added automatic checkpoint cleanup script
    - [x] Full 1000-step training completed (31h40)
    - [x] Versioned job script and config on GitHub
    - [ ] Batch size increase — deferred to next week

### Decisions and adjustments

!!! info "Decisions"
    - Dropped L40S GPU (44GB insufficient) in favor of A100L (80GB)
    - Switched to single-GPU setup with `enable_offload: True` instead of multi-GPU (NCCL issues)
    - Added checkpoint cleanup script to avoid filling up storage

### Difficulties encountered

!!! warning "Difficulties"
    - OOM errors due to the model being loaded twice into GPU memory — resolved with `enable_offload: True`
    - Corrupted checkpoint at step 50 due to `$HOME` size limit — resolved by migrating to `$SCRATCH`
    - Ray conflict on assigned node — resolved by excluding contaminated nodes via `#SBATCH --exclude`
    - 1000-step results showed occasional reward spikes with no stable learning trend — batch size hypothesis initially considered, later revised (see Week 2)


## Week 2 (May 11–15, 2026)

### Goals for the period
- Analyze the previous training results and plan next steps
- Switch environment and model based on supervisor recommendations
- Launch a first training run with the new setup

### Work completed

!!! abstract "Progress"
    - [x] Discussion with the team on Week 1 results and next steps
    - [x] Switched to RoboCasa environment with the π₀ model (~3B parameters) using PPO
    - [x] Installed RoboCasa dependencies and downloaded kitchen assets (~5GB)
    - [ ] First successful training run with π₀ — blocked by a Ray initialization hang

### Decisions and adjustments

!!! info "Decisions"
    - Dropped OpenVLA-OFT as the primary model in favor of π₀ (~3B parameters), better suited for research
    - Dropped LIBERO as the primary benchmark (mostly saturated) in favor of RoboCasa
    - LoRA will be enabled to reduce memory footprint in the next training run

### Difficulties encountered

!!! warning "Difficulties"
    - The reward spikes from Week 1 are explained by an exploration bottleneck, not batch size — gradient accumulation has no effect on learning dynamics
    - Ray hang at job startup: the Ray process initializes then blocks with no further output — root cause under investigation


## Week 3 (May 18–22, 2026)

### Goals for the period
- Launch a successful π₀ training run on RoboCasa
- Start a LIBERO-Spatial training run with OpenVLA-OFT
- Document and share experiments with the team

### Work completed

!!! abstract "Progress"
    - [x] Enabled LoRA (rank 32) for π₀ and tuned config for single A100L (80GB)
    - [x] First successful π₀ + RoboCasa CloseDrawer (PPO) run — 24 steps, `success_once` reaching 0.8125 at step 20
    - [x] Resolved `robosuite` version conflict between LIBERO and RoboCasa — created two separate venvs
    - [x] Updated all job scripts to point to the correct venv
    - [x] Opened PR on `montrealrobotics/RLinf` with configs, job scripts, and `EXPERIMENTS_README.md`
    - [x] Started LIBERO-Spatial GRPO run — `success_once` oscillating between 0.22 and 0.875 over 38 steps
    - [x] π₀ + RoboCasa CloseDrawer run completed at 150 steps
    - [x] Visualized results via TensorBoard, saved episode videos
    - [x] Uploaded final weights to HuggingFace (`josh11234/RLinf-Pi0-RoboCasa-CloseDrawer-PPO`)
    - [x] Shared results and videos with the group via Slack
    - [ ] LIBERO-Spatial run still ongoing

### Decisions and adjustments

!!! info "Decisions"
    - LIBERO and RoboCasa require separate venvs due to dependency conflicts — documented in the README
    - RoboCasa CloseDrawer is too easy as a starting task (SFT base already ~0.6–0.7) — limited room for RL contribution
    - LIBERO-Long identified as the next target task — harder SFT base rate (11.9%), better candidate for demonstrating RL gains
    - Added `unset RAY_ADDRESS` to job scripts to prevent residual Ray cluster conflicts

### Difficulties encountered

!!! warning "Difficulties"
    - Job crashed at step 24 with `FileNotFoundError: omron_mobile_base.xml` — resolved by downloading RoboCasa assets separately
    - Recurring Ray version conflicts on multiple nodes — resolved with `unset RAY_ADDRESS` and node exclusion
    - LIBERO-Spatial run preempted after ~19h — resumed from last checkpoint


## Week 4 (May 25–29, 2026)

### Goals for the period
- Analyze LIBERO-Spatial results and plan next steps
- Launch LIBERO-Long training
- Deepen the literature review

### Work completed

!!! abstract "Progress"
    - [x] Analyzed LIBERO-Spatial run results — insufficient performance, suspected learning rate not adapted after reducing the number of epochs
    - [x] Read the full OpenVLA-OFT research paper
    - [x] Comprehensive review of the RLinf repository and documentation
    - [x] Configured and launched LIBERO-Long training run (150 steps) with OpenVLA-OFT + GRPO
    - [x] Run preempted at step 145 — resumed from last checkpoint, completion imminent
    - [ ] LIBERO-Long results analysis — pending run completion

### Decisions and adjustments

!!! info "Decisions"
    - LIBERO-Long prioritized as the main task: large margin for RL improvement over the 11.9% SFT base rate
    - Learning rate to be adjusted for future runs following instabilities observed on LIBERO-Spatial

### Difficulties encountered

!!! warning "Difficulties"
    - LIBERO-Long run preempted at step 145 — resumed from available checkpoint


## Week 5 (June 1–5, 2026)

### Goals for the period
- Visualize VLA training datasets to analyze endpoint distributions and identify OOD tasks
- Study how training time can be reduced through hyperparameter tuning
- Conceptually think through how RL can be guided for OOD generalization
- Read relevant research papers
- Analyze LIBERO-Long results

### Work completed

!!! abstract "Progress"
    - [x] Discussion with the team on the true objective of the project: demonstrating RL enables OOD generalization beyond what SFT can achieve
    - [x] Dataset visualization and endpoint distribution analysis — Jupyter notebook created ([Colab](https://colab.research.google.com/github/Josh012006/OpenX-Embodiment-Datasets-Visualization/blob/main/colabs/Open_X_Embodiment_Datasets.ipynb), [Repo](https://github.com/Josh012006/OpenX-Embodiment-Datasets-Visualization))
    - [x] LIBERO-Long results analyzed: `success_once` oscillated between 0.06 and 0.19 without an upward trend — exploration bottleneck confirmed (not optimization instability, as `approx_kl` remained stable at ~0.035–0.048)
    - [x] Hyperparameter exploration to reduce training time: tested increasing replay ratio (`rollout_epoch` 4→8 for GRPO, `update_epoch` 2→4 for PPO)
    - [x] Read relevant research papers

### Decisions and adjustments

!!! info "Decisions"
    - Decided to focus on short-horizon, precise tasks for the initial phase rather than long-horizon multi-subgoal tasks

### Difficulties encountered

!!! warning "Difficulties"
    - Increasing the replay ratio shifted the bottleneck from rollout collection to gradient updates — net effect was slower training (~27 min/step vs ~14 min/step). Reverted.


## Week 6 (June 8–12, 2026)

### Goals for the period
- Analyze endpoint distributions and identify OOD tasks
- Continue hyperparameter exploration for training time reduction
- Conceptually think through RL levers for OOD generalization
- Read relevant research papers

### Work completed

!!! abstract "Progress"
    - [x] Discussion with the team on the visualization analysis
    - [x] Identification of OOD tasks based on endpoint distribution analysis
    - [x] Selected ManiSkill as the OOD benchmark
    - [x] Analyzed hyperparameter exploration results — see Decisions below
    - [ ] Finish reading relevant research papers

### Decisions and adjustments

!!! info "Decisions"
    - Increasing replay ratio does not reduce training time on large models — gradient updates are the bottleneck. Reverted to original values.
    - Effective strategy to reduce training time: reduce `max_episode_steps` and `max_epochs` directly
    - Shifted research focus: instead of OpenVLA-OFT, will study RL fine-tuning of base OpenVLA on ManiSkill `StackCube-v1`


## Week 7 (June 15–19, 2026)

### Goals for the period
- Refine the definition of OOD for this research project
- Learn how to add new tasks to the RLinf framework
- Analyze RL performance on the StackCube task

### Work completed

!!! abstract "Progress"
    - [x] Integrated StackCube into the RLinf pipeline
    - [x] Documented the process of adding a new task to RLinf (`ADDING_TASKS.md`)
    - [x] Launched GRPO and PPO runs with OpenVLA base on StackCube-v1
    - [x] Discussion with the team on the run results

### Decisions and adjustments

!!! info "Decisions"
    - OOD defined simply: a hard task on which the model does not perform well
    - On-policy RL (PPO/GRPO) directly from the base model is insufficient without prior SFT — 0% success rate confirmed
    - Pivoted to reproducing the [ExpoFT project](https://pd-perry.github.io/expo-ft/) as the new research direction

### Difficulties encountered

!!! warning "Difficulties"
    - PPO and GRPO runs on `StackCube-v1` produced 0% success — without SFT warmup, on-policy exploration cannot bootstrap


## Week 8 (June 22–26, 2026)

### Goals for the period
- Read the ExpoFT paper and understand the residual policy architecture
- Reproduce ExpoFT results in their original codebase on StackCube
- Integrate the residual policy architecture into the RLinf framework
- Read more on off-policy algorithms

### Work completed

!!! abstract "Progress"
    - [x] Read the ExpoFT paper and understood the residual policy architecture
    - [x] Reproduce ExpoFT results in the original codebase on StackCube
    - [ ] Integrate the residual policy architecture into the RLinf framework
    - [ ] Launched training runs with the residual policy architecture on StackCube
    - [x] Discussion with the team on the run results
    - [x] Began adapting ExpoFT's original real-robot (DROID) codebase into a new standalone repository (`expo-ft`) driven entirely by ManiSkill simulation instead
    - [x] Converted first StackCube-v1 motion-planning demonstrations to the DROID/LeRobot formats expected by the ExpoFT training pipeline
    - [x] Launched a first SFT warmup run on the converted demonstrations

### Decisions and adjustments

!!! info "Decisions"
    - ExpoFT will be reproduced first in a dedicated ManiSkill-based repository rather than inside RLinf, to keep the original algorithm's code as close to untouched as possible while only swapping out the environment layer
    - Full ExpoFT (SFT warmup + demonstrations + off-policy RL) will be attempted first, with components progressively stripped away afterward to identify the minimum supervision needed

### Difficulties encountered

!!! warning "Difficulties"
    - The first SFT warmup run showed a near-0% success rate, with the arm only approaching the cubes without completing the stack — still investigating whether this reflects a genuine task-difficulty ceiling for StackCube or a setup issue elsewhere in the newly-adapted pipeline
    - Given the discouraging early numbers, began considering whether StackCube itself might simply be too difficult a starting task for this approach, and whether an easier ManiSkill task should be tried instead


## Week 9 (June 29 – July 3, 2026)

### Goals for the period
- Determine whether StackCube's poor SFT results reflect genuine task difficulty or a setup/configuration issue
- Get a fully validated, trustworthy SFT baseline before touching the RL stage
- Build the evaluation tooling needed to trust future results
- Explore whether additional ManiSkill tasks can diversify the OOD task suite

### Work completed

!!! abstract "Progress"
    - [x] Traced the near-0% SFT results to a fundamental action-space mismatch: π₀.₅'s DROID pretraining expects joint-position actions in physical units, while the pipeline was using ManiSkill's `pd_ee_delta_pose` cartesian control with normalization — causing the majority of demonstration actions to be clipped
    - [x] Switched the full pipeline to joint-space control end-to-end (`pd_joint_delta_pos`, `pi05_droid_jointpos` checkpoint, `observation/joint_position` state) — replay validation immediately jumped from a small minority of actions in-range to the large majority
    - [x] Re-ran SFT with the corrected action space — results markedly more promising, but flagged several remaining inconsistencies to chase down before trusting the numbers
    - [x] Found and fixed a cluster of further silent bugs during a systematic pre-SFT review ("MAJOR RESET"): an eval script unconditionally overriding the official DROID normalization stats even for baseline evaluation; a leftover cartesian-space action rescale from the abandoned control mode corrupting a meaningful fraction of joint-space actions; a truncation-length mismatch between the environment and the eval loop; and a simulation backend mismatch preventing correct control-mode conversion
    - [x] Built `validate_demos_full_pipeline.py`, a rigorous end-to-end replay tool (real normalization stats, real environment wrapper, exact recorded seeds) — confirmed 100% demo replay success on StackCube after the fixes, giving real confidence in the data pipeline
    - [x] Built proper evaluation tooling: fixed shared episode seeds across checkpoints for apples-to-apples comparison, ±1 standard-error bars, a resumable checkpoint-sweep script that rebuilds its plot after every checkpoint
    - [x] Ran full SFT checkpoint sweeps on 200 held-out fixed seeds — StackCube reaching real, validated generalization (not memorization) at increasing success rates across SFT iterations; PushCube performing even better
    - [x] Added PushCube-v1 as a second task, converted using the same corrected joint-space pipeline
    - [x] Added PickCube-v1 as a third task to further diversify the OOD task suite, and began investigating a persistent 0% success rate on it specifically
    - [x] Traced PickCube's 0% rate to a task-design issue rather than a policy problem: ManiSkill hides the goal marker from the camera the policy actually sees, and the DROID-style observation schema has no other channel exposing the goal position — the policy had no way to know where to place the cube
    - [x] Built a targeted monkeypatch that unhides the goal marker from the sensor camera specifically (rather than adding a privileged state channel the real robot wouldn't have), and confirmed via camera-level inspection that the goal is now visible to the policy
    - [x] Re-validated PickCube end-to-end with the patch applied (demo replay, camera setup, SFT) — successful, so PickCube-v1 is kept as the third task after all
    - [x] Discussed the dataset-size question raised by the team (Jesse/Albert) and built `--num-demos`-style support so SFT can be limited to a subset of an already-converted dataset without reconversion

### Decisions and adjustments

!!! info "Decisions"
    - The early poor SFT results were not a StackCube difficulty ceiling after all — they were fully explained by the action-space mismatch and the further bugs found during the MAJOR RESET; StackCube is retained as a task
    - PushCube-v1 added as a second, easier task for pipeline sanity-checking and comparison
    - PickCube-v1 initially considered for removal given the hidden-goal-marker issue, but kept after a targeted camera-visibility patch resolved it without resorting to a privileged state channel the real robot wouldn't have — all three tasks (StackCube, PushCube, PickCube) remain in the active suite
    - Standard error (`SE = sqrt(p(1-p)/n)`), not a Wilson interval, adopted for all reported error bars, per the team's explicit preference
    - Next priority set by the team: a dataset-size ablation (full ~550 demos down to as few as ~50), followed by an SFT-warmup-necessity ablation (skip SFT entirely, start RL directly from the base model) — three competing predictions on record within the team for the latter

### Difficulties encountered

!!! warning "Difficulties"
    - Diagnosing the action-space mismatch took real effort: the symptoms (near-0% success, arm approaching but not completing the task) looked identical to a genuine task-difficulty problem
    - Several of the MAJOR RESET bugs were silent — they did not crash anything, they simply produced plausible-looking but wrong numbers, which is what made the systematic review necessary in the first place
    - PickCube's failure mode looked at first like a policy/exploration problem and required camera-level diagnostics to actually trace to the environment's own observability design


## Week 10 (July 6–10, 2026)

### Goals for the period
- Review the RL/ExpoFT training stage as rigorously as the SFT stage was reviewed, before trusting any RL result
- Launch the first real RL fine-tuning runs on StackCube, PushCube, and PickCube
- Get the infrastructure (repo layout, disk usage, submodules) into a sustainable state
- Understand whether RL is actually improving on the SFT policy, and if not, why

### Work completed

!!! abstract "Progress"
    - [x] Forked ManiSkill and switched it (alongside `openpi`) from an untracked/`pip`-installed dependency to a proper editable git submodule, enabling direct, trackable modifications instead of runtime monkeypatches
    - [x] Migrated the repository from living entirely on `$SCRATCH` to living on `$HOME`, keeping only large, regenerable artifacts (`logs/`, `demos/`) symlinked to `$SCRATCH`, following storage guidance from Mila
    - [x] Relocated large package caches (`uv`, `openpi`, `huggingface`, `pip`, `jax`) from `$HOME` to `$SCRATCH`, freeing up the `$HOME` quota
    - [x] Completed the planned line-by-line review of the RL training stage, finding and fixing a cluster of real bugs: a norm-stats override bug identical to the one found in SFT eval; ~15 CLI flags being passed to the training script that it silently never read; a hardcoded checkpoint-overwrite flag that crashed every fresh run; a mismatch between two different copies of a key training flag causing a crash whenever it was set to a non-default value
    - [x] Launched the first real RL fine-tuning runs on all three tasks from their validated SFT checkpoints
    - [x] Diagnosed and fixed two distinct causes of repeated out-of-memory crashes during RL training: a JAX recompilation issue triggered by an inconsistent input shape partway through training, and JAX's default GPU memory allocator reserving only 75% of the card by default
    - [x] Discovered that the evaluation script could not actually evaluate an RL checkpoint at all — it silently assumed every checkpoint was SFT-style and would have only ever evaluated the frozen base policy; built proper support for loading and evaluating the full trained RL agent (residual policy and critic included)
    - [x] Found and fixed a second cluster of configuration bugs: several RL hyperparameters defined in the task configs were never actually being read by the training script, silently falling back to different defaults elsewhere in the codebase; one hyperparameter's value was being silently parsed as text instead of a number by the config file reader
    - [x] Ran full evaluation curves (200 held-out fixed seeds per checkpoint) on the first RL runs across all three tasks
    - [x] Ran a battery of comparison experiments varying the update-to-data ratio and the proportion of each training batch drawn from demonstration data, including one run reproducing the original paper's exact hyperparameters within its own validated training length
    - [x] Reproduced the paper's exact training data augmentation recipe from the paper's description and confirmed the codebase already implements it correctly
    - [x] Fixed the training-time TensorBoard logging, which was silently missing most training metrics (only success rate and loop time were visible) due to a type-filtering issue

### Decisions and adjustments

!!! info "Decisions"
    - `XLA_PYTHON_CLIENT_MEM_FRACTION=0.95` adopted as the standing GPU memory setting for all RL job scripts, resolving the recurring out-of-memory crashes
    - All RL hyperparameters are now sourced exclusively from the task YAML configs, with explicit overrides wired into the training script, rather than being split silently between the YAML and a separate model config file
    - Adopted a reduced update-to-data ratio (2, versus the paper's 20) as a working default after observing that the paper's own ratio destabilizes training even faster than the reduced one, given the constraints of this simulated setup
    - Catastrophic runs from before the memory and configuration fixes were deleted from `$SCRATCH` to free disk space, keeping only the validated SFT checkpoints and the post-fix RL runs

### Difficulties encountered

!!! warning "Difficulties"
    - Every RL run tried so far — across all three tasks, across every update-to-data ratio and demonstration-ratio setting tested, including a short run reproducing the paper's own exact hyperparameters within its own validated training window — eventually degrades the SFT-pretrained policy's success rate rather than improving it
    - The degradation is consistently preceded and accompanied by growing instability in the critic's training loss, suggesting the root cause is a progressively destabilizing value function rather than a bug in the pipeline itself (verified: the evaluation now demonstrably uses the trained residual policy and critic, not just the frozen base policy)
    - Reducing the update-to-data ratio delays the onset of this instability but has not, in any configuration tried so far, prevented it — raising the open question of whether this is a fundamental limitation of high-update-ratio off-policy critic training that the paper's own short, real-robot training runs simply do not run long enough to expose
    - This finding was shared with the broader research group for discussion, as it directly bears on the project's central research question



## Week 11 (July 13–17, 2026) — current week

### Goals for the period
- Determine whether the RL degradation observed at the end of Week 10 is a critic-training instability problem, a data-integrity problem, or something structural in the algorithm itself
- Rebuild the critic architecture to match the XQC/XQCfD papers as closely as possible, and test each of their proposed mechanisms in isolation rather than all at once
- Extend the same rigor to the PPO/GRPO baselines that had been added but never actually exercised
- Bring the project's documentation (README, changelog) up to date with everything found

### Work completed

!!! abstract "Progress"
    - [x] Implemented critic pretraining (XQCfD-style BC/TD warm-start on demonstrations before RL starts, toggleable via YAML) and tested it in isolation — did not resolve the degradation on its own
    - [x] Rebuilt the critic architecture from scratch to match XQC/XQCfD: a categorical (bounded-support, C51-style) critic with batch norm and weight norm, replacing the original scalar-regression critic and its ensemble — the original architecture was preserved unmodified in a separate file rather than overwritten, specifically so it stays directly runnable for comparison
    - [x] Added adaptive reward normalization (a running estimate of reward scale) so the critic's fixed value bounds stay meaningful across tasks without hand-tuning them per task — verified this holds even under a sudden, large shift in a task's reward scale
    - [x] Ran controlled comparisons of the new critic architecture against the old one, and separately of a newly-found reward-timing bug (below) fixed vs. unfixed — in both cases, the critic's own value estimates became measurably more stable and well-behaved, but the policy's success rate collapsed in the same way regardless, which was itself an important finding (see Decisions)
    - [x] Found and fixed a real bug in the online data-collection loop: the environment's reward/termination signal was being read one step too early, so every stored transition was paired with the reward from the *previous* action rather than the one that actually produced it — confirmed via a controlled reproduction, not just a code read, and confirmed the scope is limited to online RL collection (supervised pretraining and demonstration generation were never affected)
    - [x] Investigated why the original paper's real-robot experiments don't appear to hit this same degradation despite using an identical mechanism — traced most likely to their live human-in-the-loop correction during training and their much shorter training horizon, rather than anything different in the algorithm itself
    - [x] Implemented KL regularization against the pretrained policy (replacing the generic exploration bonus, per XQCfD) for the trainable residual policy, and ran an initial test — even though a configuration gap meant this first run wasn't a fully isolated test, results were the most encouraging seen in this investigation so far (a consistent improvement in success rate over a comparable run without it); an isolated follow-up test is in progress
    - [x] Extended the same debugging rigor to the PPO and GRPO baselines added but not yet exercised: found and fixed several independent numerical bugs (a network-construction incompatibility, an action-formatting error, an incorrect fallback for a value that has no closed form for our policy type, an unbounded value that could silently reach infinity under policy drift, and a numerical instability at the boundary of the action space) — each confirmed with a targeted, reproducible test rather than by inspection alone, and a defensive safeguard added on top so a future occurrence of any similar issue is caught and skipped rather than corrupting training silently
    - [x] Discovered that PPO/GRPO's trainable policy was never actually connected to the supervised-pretrained checkpoint in the first place — it is a separate, randomly-initialized network, with the pretrained checkpoint used only to pre/post-process its inputs and outputs. This explains why these baselines showed no learning signal from the start, independent of any of the bugs above. Added an equivalent behavior-cloning warm-start option for these baselines to address it
    - [x] Brought the project README fully up to date: current findings, the new critic architecture and its rationale, the timing bug and its scope, and a complete changelog of the fixes made during this phase
    - [x] Made the original critic architecture directly selectable and runnable (not just preserved as a static reference) so it can be A/B-tested against the new one under identical conditions on request

### Decisions and adjustments

!!! info "Decisions"
    - Treating the critic-architecture rebuild and the data-timing fix as two separate, independently-tested changes rather than bundling them, specifically so their effects (or lack thereof) could each be attributed cleanly
    - Given both of those real, confirmed fixes improved the critic's own behavior without changing the policy's outcome, the leading hypothesis shifted to the action-selection mechanism itself (the same critic both proposes and judges its own preferred action, a known bias pattern in this style of algorithm) rather than anything about how the critic is trained
    - Chose to exhaust the paper's own proposed mechanisms (critic pretraining, the new critic architecture, KL regularization, and — still to come — an architectural change intended to make the policy aware of when it's outside familiar territory) before testing an alternative fix from a different line of research, so that what does or doesn't work stays attributable to a single, citable approach
    - Confirmed the tasks in use rely on a shaped (dense), not sparse, reward signal — relevant context for judging how much the timing bug actually mattered in practice, versus in the original paper's sparse-reward, real-robot setting

### Difficulties encountered

!!! warning "Difficulties"
    - Two independent, confirmed fixes (the critic architecture, the reward-timing bug) each produced a real, measurable improvement to the critic's behavior without improving the actual policy performance — a useful result for narrowing down the cause, but a reminder that "the critic looks healthier" and "the policy is getting better" are not the same thing
    - The first test of KL regularization was confounded by a leftover exploration-bonus term that should have been disabled for an isolated test — caught before drawing any conclusion from it, but a reminder to double check every knob a new feature interacts with before reporting results
    - The PPO/GRPO baselines were not doing what they were assumed to be doing (continuing from the pretrained policy) for the entire time they had been in use — a significant, non-obvious gap that only surfaced once their results were looked at critically rather than taken at face value