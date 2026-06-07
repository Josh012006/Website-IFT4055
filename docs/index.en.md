---
title: Project Overview
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Project Overview

!!! info "General information"
    **Session**: Summer 2026  

    **Author(s)**: [Josué Saint-martin Junior Mongan](https://mila.quebec/en/directory/josue-mongan) (Student ID 20290870)

    **Theme(s)**: Research internship in - **Reinforcement Learning for fine-tuning vision-language models (fine-tuning, RL, VLAs)**

    **Supervisor(s)**: [Glen Berseth](https://mila.quebec/en/annuaire/glen-berseth)

    **Collaborator(s)**: [Florence Cloutier](https://mila.quebec/en/annuaire/florence-cloutier), [Albert Zhan](https://mila.quebec/en/annuaire/albert-zhan), [Jesse Silverberg](https://mila.quebec/en/annuaire/jesse-silverberg)

## **Project description**

This project is part of the emerging field of reinforcement learning (RL)-based fine-tuning applied to vision-language-action (VLA) models. VLAs are multimodal foundation models that combine visual understanding, linguistic reasoning, and motor action generation for robotics applications.  
<br />

While these models achieve strong performance on simple tasks through supervised imitation learning (SFT), their generalization and reasoning capabilities on complex tasks remain limited. Post-SFT reinforcement learning is a promising direction to overcome these limitations, but its application to VLAs is still underexplored and raises fundamental challenges in optimization, training stability, and reward design.

### Context

Imitation learning has enabled significant advances in robotics, but requires large quantities of expert 
demonstrations, which is costly and difficult to scale. RL offers an alternative by allowing the robot to 
learn directly through environment interaction, guided by a reward signal. Applied to VLAs, RL could enable 
these models to learn beyond their initial training distribution, developing behaviors that imitation alone 
cannot capture. However, RL optimization on large-scale models remains an open challenge, particularly in 
terms of algorithmic stability, computational efficiency, and the ability to generalize.

### Problem statement

- **Can RL fine-tuning enable VLAs to learn beyond their SFT training data distribution?**
- **Is exploration or optimization the primary bottleneck in RL fine-tuning of VLAs?**
- **Which algorithms, architectures, and task types are most conducive to this kind of improvement?**
- **How can RL improve the model's ability to develop genuine reasoning capabilities rather than simply memorizing behaviors?**

### Preliminary state of the art

Reinforcement learning fine-tuning of large language models (LLMs) has achieved notable success with algorithms such as PPO and GRPO, particularly in works like DeepSeek-R1 and InstructGPT.  
Extending these approaches to vision-language-action (VLA) models is more recent and introduces additional challenges due to their multimodal and action-oriented nature.  
<br />

Recent frameworks such as [RLinf](https://github.com/RLinf/RLinf) provide unified infrastructure for applying RL to VLA models, supporting multiple architectures (OpenVLA, π0, GR00T) and environments (LIBERO, ManiSkill, RoboCasa).  
<br />

Early results indicate that RL can significantly improve performance over supervised fine-tuning (SFT), especially on long-horizon and multi-step tasks. However, the optimal conditions for this improvement remain poorly understood. Recent works ([LIBERO-OOD, ICLR 2026](https://openreview.net/forum?id=jtrhwfgseW); [LIBERO-Plus, arXiv 2510.13626](https://arxiv.org/abs/2510.13626)) also show that SFT-trained models suffer from out-of-distribution generalization issues, often memorizing training configurations rather than developing robust task understanding. This suggests that current fine-tuning and evaluation methods do not always promote the learning of robust and generalizable representations.

### Research objectives

1. **Disprove** the hypothesis that RL only redistributes probability mass from the pretrained model, by testing on tasks maximally different from the SFT training data.
2. **Identify** whether exploration or optimization is the primary bottleneck in RL fine-tuning of VLAs.
3. **Compare** different architectures (OpenVLA-OFT, π0,...) and algorithms (PPO, GRPO,...) in terms of efficiency and stability on a single A100 GPU.
4. **Contribute** to understanding the minimum conditions required for RL to genuinely expand a VLA's capabilities beyond its initial supervised training.

### Proposed methodology

The project adopts a comparative experimental approach. RL fine-tuning experiments are conducted across 
multiple VLA architectures and robotic manipulation task suites in simulation, using frameworks such as RLinf. 
Reinforcement learning algorithms are compared, as well as different model architectures.

Particular attention is given to task selection — prioritizing tasks where the SFT base model performs poorly 
and where the model's training data is precisely documented, in order to rigorously evaluate whether RL enables 
out-of-distribution generalization.

### Evaluation of results

Results are evaluated using episodic success rate (`success_once`), comparing the base SFT model with the RL fine-tuned model. Learning curves are used to assess training stability and efficiency.  
A qualitative analysis of learned behaviors (via episode videos) complements the quantitative evaluation. Observed limitations—such as algorithmic instability, computational constraints, and generalization gaps—are integrated into the analysis to better understand open challenges in the field.

## Team

**Josué Mongan** is a student in the **Bachelor of Computer Science at the University of Montreal**. He is the sole student member of the team and is responsible for experiment design and execution, literature review, and project documentation.

## Timeline

!!! info
    Full tracking is available on the [Project Tracking](suivi.md) page.

| Activities                     | Start   | End     | Deliverable                          | Status          |
|--------------------------------|---------|---------|--------------------------------------|-----------------|
| Topic definition               | May 4   | May 15  | Project proposal                     | ✅ Completed    |
| Preliminary literature review  | May 4   | May 29  | Annotated bibliography / synthesis   | 🔄 In progress  |
| Methodology and analysis plan  | May 18  | Jun 5   | Methodological plan                  | 🔄 In progress  |
| Experimentation or analysis    | Jun 1   | Jul 24  | Preliminary results                  | ⏳ Upcoming     |
| Synthesis and interpretation   | Jul 20  | Aug 7   | Discussion of results and limitations| ⏳ Upcoming     |
| Presentation + final report    | Aug 7   | Aug 14  | Presentation + research report       | ⏳ Upcoming     |