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

    **Supervisor(s)**: [Glen Berseth](https://mila.quebec/fr/annuaire/glen-berseth)

    **Collaborator(s)**: [Florence Cloutier](https://mila.quebec/fr/annuaire/florence-cloutier)

## **Project description**

This project is part of the emerging field of reinforcement learning (RL)-based fine-tuning applied to vision-language-action (VLA) models. VLAs are multimodal foundation models that combine visual understanding, linguistic reasoning, and motor action generation for robotics applications.  
<br />

While these models achieve strong performance on simple tasks through supervised imitation learning (SFT), their generalization and reasoning capabilities on complex tasks remain limited. Post-SFT reinforcement learning is a promising direction to overcome these limitations, but its application to VLAs is still underexplored and raises fundamental challenges in optimization, training stability, and reward design.

### Context

Imitation learning has led to significant advances in robotics, but it requires large amounts of expert demonstrations, which are costly and difficult to scale. RL offers an alternative by enabling agents to learn directly through interaction with their environment, guided by a reward signal.  
When applied to VLAs, RL could improve reasoning and planning capabilities on tasks requiring precision or long-horizon decision-making. However, RL optimization on large-scale models remains an open challenge, particularly in terms of algorithmic stability, computational efficiency, and reward signal design.

### Problem statement

- **Under what conditions does RL-based fine-tuning improve VLA capabilities beyond what SFT can achieve?**  
- **Which algorithms, architectures, and task types are most suitable for such improvements?**  
- **How can tasks and reward signals be designed to encourage genuine reasoning rather than memorized behavior?**

### Preliminary state of the art

Reinforcement learning fine-tuning of large language models (LLMs) has achieved notable success with algorithms such as PPO and GRPO, particularly in works like DeepSeek-R1 and InstructGPT.  
Extending these approaches to vision-language-action (VLA) models is more recent and introduces additional challenges due to their multimodal and action-oriented nature.  
<br />

Recent frameworks such as [RLinf](https://github.com/RLinf/RLinf) provide unified infrastructure for applying RL to VLA models, supporting multiple architectures (OpenVLA, π0, GR00T) and environments (LIBERO, ManiSkill, RoboCasa).  
<br />

Early results indicate that RL can significantly improve performance over supervised fine-tuning (SFT), especially on long-horizon and multi-step tasks. However, the optimal conditions for this improvement remain poorly understood. Recent works ([LIBERO-OOD, ICLR 2026](https://openreview.net/forum?id=jtrhwfgseW); [LIBERO-Plus, arXiv 2510.13626](https://arxiv.org/abs/2510.13626)) also show that SFT-trained models suffer from out-of-distribution generalization issues, often memorizing training configurations rather than developing robust task understanding. This suggests that current fine-tuning and evaluation methods do not always promote the learning of robust and generalizable representations.

### Research objectives

1. **Explore** the conditions under which RL improves VLA performance compared to SFT, in terms of algorithms, architectures, and task types.  
2. **Identify** task categories where SFT is insufficient and RL provides clear benefits—particularly tasks requiring precision, sequential planning, or reasoning.  
3. **Study** the impact of reward signal design on RL fine-tuning effectiveness.  
4. **Contribute** to understanding the optimization challenges of RL on VLAs, documenting configurations and practices that work under limited computational resources.

### Proposed methodology

The project follows a comparative experimental approach. RL fine-tuning experiments are conducted across multiple VLA architectures and robotic manipulation benchmarks in simulation, using the RLinf framework. Reinforcement learning algorithms (PPO, GRPO, etc.) are compared, along with different model architectures (OpenVLA-OFT, π0).

Special attention is given to task selection—prioritizing tasks where the base SFT model performs poorly, in order to better isolate the contribution of RL. Reward signal design and its impact on learning quality is also an important axis of investigation.

### Evaluation of results

Results are evaluated using episodic success rate (`success_once`), comparing the base SFT model with the RL fine-tuned model. Learning curves are used to assess training stability and efficiency.  
A qualitative analysis of learned behaviors (via episode videos) complements the quantitative evaluation. Observed limitations—such as algorithmic instability, computational constraints, and generalization gaps—are integrated into the analysis to better understand open challenges in the field.

## Team

**Josué Mongan** is a student in the **Bachelor of Computer Science at the University of Montreal**. He is the sole student member of the team and is responsible for experiment design and execution, literature review, and project documentation.

## Timeline

!!! info
    Full tracking is available on the [Project Tracking](suivi.md) page.

| Activities                      | Start   | End     | Deliverable                          | Status       |
|--------------------------------|---------|---------|--------------------------------------|--------------|
| Topic definition               | May 4   | May 15  | Project proposal                     | Completed    |
| Preliminary literature review  | May 4   | May 29  | Annotated bibliography / synthesis   | In progress  |
| Methodology and analysis plan  | May 18  | Jun 5   | Methodological plan                  | In progress  |
| Experimentation or analysis    | Jun 1   | Jul 24  | Preliminary results                  | Upcoming     |
| Synthesis and interpretation   | Jul 20  | Aug 7   | Discussion of results and limitations| Upcoming     |
| Presentation + final report    | Aug 7   | Aug 14  | Presentation + research report       | Upcoming     |