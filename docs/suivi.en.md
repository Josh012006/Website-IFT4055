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

## Week 1 (May 5–9, 2026)

### Goals for the period
- Get in touch with the team and set up the work environment
- Launch a first RL training run on the Mila cluster
- Identify and resolve infrastructure issues

### Work completed

!!! abstract "Progress"
    - [x] Connected with the team, joined the lab's Slack
    - [x] Read the RLinf documentation (LIBERO)
    - [x] Connected to the Mila cluster and resolved access issues with IT support
    - [x] Cloned the repository and installed dependencies on an interactive compute node (`salloc --no-root`)
    - [x] Wrote the `job.sh` script and submitted it via `sbatch`
    - [x] Progressively resolved errors: wrong normalization key, insufficient CPU RAM, insufficient GPU VRAM, multi-GPU NCCL timeout
    - [x] Switched to A100L GPU (80GB) with `enable_offload: True` and `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`
    - [x] First successful training run reaching 132 steps
    - [x] Migrated project to `$SCRATCH`, created symbolic link from `$HOME`, added automatic checkpoint cleanup script
    - [x] Full 1000-step training completed (31h40)
    - [x] Versioned job script and config on GitHub
    - [ ] Batch size increase — deferred to next week

### Decisions and adjustments

!!! info "Decisions"
    - Dropped L40S GPU (44GB insufficient) in favor of A100L (80GB)
    - Switched to single-GPU setup with `enable_offload: True` instead of multi-GPU (NCCL issues)
    - Reduced scope: moved away from LIBERO-90 (90 simultaneous tasks) toward a more focused configuration
    - Added checkpoint cleanup script to avoid filling up storage

### Difficulties encountered

!!! warning "Difficulties"
    - The model is loaded twice into GPU memory (RolloutGroup + ActorGroup), doubling the memory footprint and causing OOM errors
    - Resolved by enabling `enable_offload: True` and setting `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`
    - Corrupted checkpoint at step 50: the `$HOME` directory (96GB) was too small to store checkpoints (~50GB each)
    - Resolved by moving the project to `$SCRATCH` and adding the cleanup script
    - Ray conflict on the assigned node: a residual Ray cluster from another user prevented initialization
    - Resolved by excluding the contaminated node via `#SBATCH --exclude`
    - Training results at 1000 steps: occasional reward spikes (~0.13) with no stable learning trend — batch size too small to extract a robust learning signal
