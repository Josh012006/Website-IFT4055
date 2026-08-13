---
title: Evaluation & Discussion
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Evaluation & Discussion

This page documents how the project was evaluated, what the results showed, and how we
interpret them. It summarizes the evidence behind the report's central claim: ExpoFT
consistently failed to improve on the SFT baseline until a silent reward-configuration bug
was found and fixed — and even then, only one of the three tasks clearly benefited.

## Evaluation Methodology

The project moved through two experimental phases, each evaluated differently.

- **Primary metric throughout:** task success rate.
- **Phase 1 (on-policy RL, RLinf framework)** also tracked `env/return`, `actor/policy_loss`,
  and `actor/approx_kl` to monitor optimization health independently of task performance.
- **Phase 2 (ExpoFT, off-policy)** also tracked critic diagnostics: `training/target_q_max`,
  `training/critic_loss`, and a custom **misrank-rate** diagnostic comparing the critic's chosen
  action against a separately validated, high-performing reference policy.
- **Rigor:** every checkpoint is evaluated on the same **200 fixed initial states**, with
  standard errors reported, so that differences reflect policy behavior rather than
  environment stochasticity.

All three evaluation tasks come from ManiSkill and share a contact-rich, precision-demanding
manipulation setup:

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/pickcube.png" alt="Curve">

            *PickCube-v1*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/pushcube.png" alt="Curve">

            *PushCube-v1*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/stackcube.png" alt="Curve">

            *StackCube-v1*
        </td>
    </tr>
</table>

*The three ManiSkill environments used for evaluation. PickCube-v1 is the most exploration-demanding; PushCube-v1 the easiest; StackCube-v1 the most precision-demanding.*

## Results

### SFT baseline: the bar RL needs to clear

Before any RL fine-tuning, validated SFT success rates on 200 fixed seeds were:

| Task | Best success rate | Evaluation seeds |
|---|---|---|
| StackCube-v1 | 49% | 200 |
| PushCube-v1 | 96% | 200 |
| PickCube-v1 | 27% | 200 |

Reaching these numbers required fixing several silent pipeline bugs first (an action-space
mismatch, a normalization-stats override, a leftover action-rescaling artifact), each confirmed
via a dedicated tool that replays demonstrations end-to-end and checks for 100% replay fidelity.
The full SFT trajectory across checkpoints looked like this:

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/curve-sft-pickcube.png" alt="Curve">

            *SFT checkpoint sweep: success rate vs. RL training step for PickCube*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/curve-sft-pushcube.png" alt="Curve">

            *SFT checkpoint sweep: success rate vs. RL training step for PushCube*
        </td>
    </tr>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/curve-sft-stackcube.png" alt="Curve">

            *SFT checkpoint sweep: success rate vs. RL training step for StackCube*
        </td>
    </tr>
</table>

*Note the different y-axis ranges per task — PushCube climbs toward 90%+, StackCube toward ~45%, PickCube toward ~25%.*

### The central finding: ExpoFT degraded every configuration

With the SFT baseline validated, ExpoFT was run from these checkpoints across several
update-to-data (UTD) ratios. Every configuration **degraded** success rate rather than
improving it:

| Configuration | UTD ratio | Steps | Initial SR | Final SR |
|---|---|---|---|---|
| Original (paper) configuration | 20 | 20k | 62% | 1% |
| Reduced updates | 2 | 120k | 62% | 0% |

A higher UTD ratio collapsed faster; a lower ratio only delayed the collapse — it never
prevented it. Total training time was not the driver; the UTD ratio was.

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/config2-eval-success-pushcube.png" alt="Curve">

            *Success rate collapse under UTD ratio 2 (120K steps)*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/config3-eval-success-pushcube.png" alt="Curve">

            *Success rate collapse under UTD ratio 20 (20K steps, paper configuration)*
        </td>
    </tr>
</table>

*Both configurations collapse from a ~60–100% starting success rate down to single digits — the higher UTD ratio (right) collapses faster and more severely.*

### Diagnosing the failure mechanism

Degradation was consistently accompanied by rising critic-loss instability, which pointed
first at critic-training instability as the likely cause. Five controlled diagnostic
experiments were run to test this and narrow down the real cause:

| Experiment | Observation | Interpretation |
|---|---|---|
| Frozen critic encoder | Similar Q-value drift | Not caused by representation drift |
| Near-zero residual budget | Similar degradation | Not caused by the residual policy learning badly |
| Candidate sweep (N=2 vs. N=8) | Nearly identical target-Q drift | Not driven by candidate-pool size |
| Categorical (bounded) critic rebuild | Better-behaved values, same collapse | Not only a critic-architecture problem |
| **Misrank diagnostic** | **0.94–1.00 from the very first checkpoint** | **Directly confirms an action-selection bias** |

The misrank diagnostic was decisive. For real training states, we compared the critic's score
for its own selected action against its score for the action a separately validated,
96%-success reference policy would have taken instead:

![misrank_rate by RL checkpoint, saturated between 0.94 and 1.00 from the first checkpoint onward]({{ config.site_url }}/images/fig_misrank_rate.png)
*The critic preferred its own, worse choice 94–100% of the time, saturated from the very first RL checkpoint evaluated — a textbook maximization / self-reference bias: the same critic that scores the candidates also bootstraps its own training target from whichever candidate gets picked.*

### Correction attempts

Guided by this diagnosis, six stabilization mechanisms from the literature were tested one at
a time on top of a categorical critic:

| Mechanism | Targeted issue | Outcome |
|---|---|---|
| Critic pretraining (warm-start on demos) | Improve value initialization | Partial improvement |
| KL regularization toward base policy | Limit residual drift | Partial improvement |
| Double-Q selection decoupling | Break critic self-reference | No significant improvement |
| Clipped double-Q (TD3-style) | Reduce Q overestimation | No significant improvement |
| HetStat conservative policy | Revert to conservative actions OOD | No improvement |
| Larger residual budget / demo ratio | Improve exploration & data quality | No improvement |

!!! warning "None of these resolved the collapse on its own"
    This is an important negative result in itself — it points toward something more
    fundamental than any single stabilization trick.


![eval/success_rate over all the categorical critics corrections attempts]({{ config.site_url }}/images/categorical_critic_corrections.png)
*Success rate across all the categorical critic correction attempts.*

### The turning point: a silent reward-configuration bug

A sanity check on the critic's reported values (routinely 13–19, implausible under a sparse
binary reward) uncovered two compounding, independent bugs:

1. The environment had **never been explicitly set to sparse reward** — it was silently using
   ManiSkill's dense, shaped default (confirmed by replaying a demonstration and finding 0%
   exactly-zero-reward steps).
2. The categoricalcritic's reward normalization divided by a running estimate of its own scale — as
   success fell, that estimate shrank, inflating the effective reward for the rare remaining
   successes (a self-reinforcing loop).

Both were corrected. This does not overturn the misrank-rate evidence (a same-critic, relative
comparison) — but it substantially weakens the earlier framing of raw value magnitudes as
self-evidently absurd.

### Final evaluation after correction

Repeating the UTD ratio comparison under the corrected sparse reward **reversed** the earlier
result: UTD = 20 became the most stable, best-performing configuration, while UTD = 1 degraded
steadily.

![eval/success_rate under UTD ratio 1 vs. UTD ratio 20, sparse reward, corrected configuration]({{ config.site_url }}/images/utd_reversal_after_fix.png)
*Reading: the UTD ratio amplifies whatever training signal is present — a self-reinforcing error before the fix, sound learning after it.*

Testing the corrected configuration (UTD = 20) on all three tasks gave mixed but genuine
progress:

| Configuration | StackCube-v1 | PushCube-v1 | PickCube-v1 |
|---|---|---|---|
| SFT baseline | 41% | 62% | 22% |
| ExpoFT (MSE critic, corrected reward) | 32% | **90%** | 16% |

<table>
    <tr>
        <td>
            <img src="{{ config.site_url }}/images/final-curve-pickcube.png" alt="Curve">

            *Final training curve after the reward fix for PickCube-v1*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/final-curve-pushcube.png" alt="Curve">

            *Final training curve after the reward fix for PushCube-v1*
        </td>
        <td>
            <img src="{{ config.site_url }}/images/final-curve-stackcube.png" alt="Curve">

            *Final training curve after the reward fix for StackCube-v1*
        </td>
    </tr>
</table>

*PushCube-v1 clearly beats its SFT baseline. StackCube-v1 and PickCube-v1 no longer collapse to near-zero, but oscillate around their starting performance rather than steadily improving — note the different y-axis scales across the three panels.*

## Analysis and Interpretation

- **On-policy RL is limited primarily by exploration.** When the base policy has near-zero
  success on a task, it generates no successful trajectories to learn from — optimization
  itself stayed numerically healthy (stable `approx_kl`), but success never trended upward.
- **Off-policy RL (ExpoFT) solves that cold-start problem via demonstrations**, but this
  shifts the bottleneck rather than removing it: training becomes highly sensitive to reward
  formulation, critic architecture, and the update-to-data ratio.
- **Reward calibration is not a peripheral detail — it changed the entire training dynamic.**
  The same UTD ratio that was most destabilizing under the miscalibrated reward became the
  most stabilizing one after correction.
- Overall, the experiments do not give a definitive answer on whether RL can consistently
  learn genuinely new behavior beyond the SFT distribution, but they do surface concrete,
  practical limitations that must be addressed before RL fine-tuning can reliably extend VLA
  capabilities.

## Project Limitations

- **Narrow scope:** experiments are exclusive to ManiSkill simulation across three
  manipulation tasks — a narrow slice of possible robotic environments. Results obtained here
  may not directly transfer to physical robots.
- **Run-to-run variability:** RL training showed non-negligible variance between identical
  reruns — on the order of a ~35% relative difference in final success rate for the same
  configuration. Every single-run comparison in this project therefore carries real
  uncertainty; a more extensive multi-seed evaluation would be needed to confirm which partial
  improvements are robust.
- **Inconsistent evaluation protocols across phases:** Phase 1 benefited from parallelized
  on-policy environment collection, while Phase 2 (ExpoFT) relied on sequential online
  interaction — complicating any direct efficiency comparison between the two approaches.
- **Time constraints** left some promising leads unexplored, most notably recalibrating the
  categorical critic's value support for the sparse-reward regime.
- **Recurring pattern — silent bugs, not crashes:** action-space mismatches, reward-
  configuration differences, and evaluation tools making architecture-specific assumptions all
  produced plausible-looking but incorrect numbers, never a crash. The fix was independent
  validation: a dedicated tool replayed demonstrations end-to-end through the real
  normalization statistics, environment, and seeds, confirming 100% replay fidelity before any
  downstream result was trusted. The broader lesson: reproducibility in RL fine-tuning
  requires verifying every interaction between environment, reward, data pipeline, and model
  architecture — not just reporting hyperparameters.