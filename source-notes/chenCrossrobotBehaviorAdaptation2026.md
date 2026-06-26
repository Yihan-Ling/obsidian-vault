# [Cross-robot behavior adaptation through intention alignment](zotero://select/library/items/I9SIRAXR)

 **Authors:** Xi Chen, Yuan Gao, Hangxin Liu, Fangkai Yang, Ali Ghadirzadeh, Jun Yang, Bin Liang, Chongjie Zhang, Tin Lun Lam, Song-Chun Zhu
 **Published:** 2026
 **Journal/Conference:** Science Robotics
**Citekey:** `chenCrossrobotBehaviorAdaptation2026`

---
## Abstract
> [!abstract]
>  ```
>  Imitation learning (IL) has succeeded in enabling robots to perform new tasks by learning from demonstrations. However, its success is often constrained by the need for direct skill mappings between a learner and a demonstrator under identical conditions, limiting its adaptability to diverse environments and generalization across robots with different physical embodiments. To address these challenges, we introduce the Intention-Aligned Imitation Learning (IAIL) framework, a behavior adaptation approach that extends the conventional scope of IL by enabling robots to reproduce motions demonstrated by heterogeneous peers, even in previously unseen situations. Inspired by human cultural learning, IAIL aligns and adapts robot motions on the basis of high-level intentions annotated in natural language rather than by directly copying motor movements. This alignment is achieved by constructing a shared intention space that connects robot-generated motions with linguistic annotations, enabling inference-time behavior adaptation across diverse embodiments and environmental contexts. The framework further supports scalable task allocation in heterogeneous robot teams by leveraging differences in capabilities and constraints. We validated IAIL through real-world experiments involving seven distinct robots performing multistep collaboration tasks across 30 scenarios. Our results demonstrate that IAIL enables robust intention-aligned behavior adaptation across variations in embodiment, motion modality, and task configuration. These capabilities enable flexible behavior transfer across heterogeneous robots and support resilient, autonomous multirobot systems for reliable real-world collaboration.
          , 
            Intention-level alignment enables robust behavior adaptation across robots and scales to heterogeneous multirobot teams.
>  ```

---

%% begin notes %%
## Summary
## Introduction

Imitation learning (IL) has largely required that a learner and demonstrator operate under identical conditions with aligned action spaces, making it impossible to transfer skills across robots with fundamentally different bodies or environments. When a drone must learn from a robotic arm, or a ground vehicle from an underwater vehicle, there is no sensible motor-level correspondence to exploit. This work asks whether robots can instead imitate at the level of *intention* — what a motion is trying to achieve — rather than at the level of motor commands, and whether that framing naturally extends from single-agent to heterogeneous team-to-team imitation with capability-aware task allocation.

## Related Work

**Embodiment-correspondence methods (agent-to-agent)**

| Approach | Key idea | Limitation addressed by IAIL |
|---|---|---|
| Body-component correspondence (Ghadirzadeh et al. 2021 IROS; Hejna et al. 2020 ICML) | Align invariant body parts across morphologies | Fails when motion modalities are fundamentally different (e.g., ground vs. aerial) |
| Domain-confusion / adversarial feature transfer (Chen et al. 2020 ICRA; Lee et al. 2024 *Sci. Robot.*) | Find shared visual/feature space under env. variation | Limited to moderate domain gaps; breaks under large embodiment mismatch |
| Task/outcome-paired correspondence (Finn et al. 2017; Yu et al. 2018; Jang et al. 2022; Beck et al. 2025) | Learn from manually labeled or paired trajectory datasets | Requires per-demonstrator-learner pair labeling; not scalable |
| Unsupervised skill correspondence (Shankar et al. 2022 ICML — density-based baseline) | Align skill distributions via cycle-consistency loss | Degrades severely when demonstrator and learner have different task distributions |

**Language-conditioned imitation**

Methods such as LUMOS (Nematollahi et al. 2025 ICRA), RT-2 (Zitkovich et al. 2023 CoRL), OpenVLA (Kim et al. 2025 CoRL), and language-conditioned IL (Zhou et al. 2024; Yao et al. 2025) use natural language as an intermediate representation for zero-shot cross-embodiment transfer. These form the **description-based baseline**: they lack an explicit mechanism to assess whether the decoded action is *feasible* for the specific learner robot, causing failures when capability constraints are violated.

**Large-scale cross-embodiment pretraining**

Open X-Embodiment (O'Neill et al. 2024 ICRA), Octo (Ghosh et al. 2024 RSS), OpenVLA, and HPT (Wang et al. 2024 NeurIPS) focus on universal policy learning from heterogeneous datasets but do not explicitly address the problem of transferring task-solving behaviors between specific robot pairs.

**Multirobot IL and task allocation**

Allen et al. (2012), Song et al. (2018 NeurIPS), Feng et al. (2025), and Gao et al. (2023 *IEEE Trans. Robot.*) address multirobot IL or task allocation but assume homogeneous teams or predefined role mappings, leaving heterogeneous team-to-team imitation unexplored.

**Cognitive science motivation**

The paper draws on rational imitation (Gergely et al. 2002 *Nature*; Meltzoff 1995) and mirror neuron literature (Bonini et al. 2022; Oztop et al. 2006) to argue that intention-level, rather than motor-level, imitation is also the mechanism humans use.

## Methods & Architecture

IAIL trains three model types per robot team and operates in three inference stages.

**Training: Motion Generator $p_{\theta_i}$**

A VAE trained independently per robot $i$ on an expert dataset $D_i$ of $(s, a)$ trajectories. Given state $s$ and latent code $z$, it produces executable action sequences. Objective:

$$\arg\max_{\vartheta_i,\theta_i} \mathbb{E}_{(s,a)\sim D_i}\!\left[\mathbb{E}_{q_{\vartheta_i}(z|s,a)} \log p_{\theta_i}(a|s,z) - \beta_{\mathrm{KL}}\,\mathrm{KL}\!\left(q_{\vartheta_i}(z|s,a)\,\big\|\,p(z)\right)\right]$$

The generator's role is to enumerate *feasible* candidate actions given the robot's current context.

**Training: Motion Encoder $f_{\psi_i}$ and Annotation Encoder $f_\xi$**

Each robot has a robot-specific motion encoder; all robots share a single annotation encoder. They are trained jointly with a contrastive (InfoNCE-style) loss over motion–language pairs:

$$\mathcal{L}_{\psi,\xi} = \mathcal{L}^{\tau\to l}_{\psi,\xi} + \mathcal{L}^{l\to\tau}_{\psi,\xi}$$

where the score function is $\varepsilon(\tau_i, l) = \exp\!\bigl(\langle f_{\psi_i}(\tau_i),\, f_\xi(l)\rangle\bigr)$ with $\langle\cdot,\cdot\rangle$ denoting cosine similarity, and $N_b - 1$ negatives are sampled from the combined dataset $D^l = \bigcup_i D_i$. Each trajectory receives 3–5 human-written language annotations at varying abstraction levels (e.g., "pick up a white paper cup" down to "pick up a cup").

**OOD handling**

Actions sampled from the generator with random $z$ (out-of-distribution, OOD) are annotated as `"unknown"` and added to training. This pushes their embeddings near the `"unknown"` label in intention space, enabling reliable detection at inference.

**Inference Stage 1 — Context-aware motion generation**

Sample a batch of candidate trajectories $\{\tau_i^{(k)}\}$ from $p_{\theta_i}(a|s_i, z)$ with $z\sim p(z)$.

**Inference Stage 2 — Motion intention extraction**

Project candidates through $f_{\psi_i}$ and project the demonstrator trajectory $\tau_j$ through $f_{\psi_j}$ into the shared intention space.

**Inference Stage 3 — Motion association by intention similarity**

Compute for each candidate:

$$V_{\mathrm{valid}}(\tau_i^{(k)}) = -\varepsilon\!\left(\tau_i^{(k)},\,\texttt{"unknown"}\right), \quad V_{\mathrm{aligned}}(\tau_i^{(k)},\tau_j) = \varepsilon\!\left(\tau_i^{(k)},\,\tau_j\right)$$

The feasible set is $B = \{\tau_i^{(k)} \mid V_{\mathrm{aligned}} > \epsilon_{\mathrm{aligned}},\; V_{\mathrm{valid}} > \epsilon_{\mathrm{valid}}\}$. If $B = \emptyset$ the robot stays inactive; otherwise, $\pi(\tau|\tau_j) = \arg\max_{\tau_i^{(k)}\in B} (V_{\mathrm{aligned}} + V_{\mathrm{valid}})$.

**Team-to-team extension**

Candidates are pooled across all learner robots: $B_{\mathrm{team}} = \bigcup_{i\in R_{\mathrm{team}}} \{\tau_i^{(k)}\}$. The highest-scoring candidate is selected and dispatched to the robot that generated it, achieving capability-aware role assignment without any predefined allocation.

## Experiments & Results

**Real-world setup**

Seven heterogeneous robots across two teams:

- *Demonstrators:* Tello (drone), Dual-Arm (bimanual manipulator), Spark (mobile transporter)
- *Learners:* Cuboat (USV), Single-Arm, Pepper (mobile manipulator), Diablo (height-adjustable mobile)

Task: 5-step monitoring-fetch-deliver pipeline across 30 scenarios with randomized user location, item type, and delivery destination. One-third of scenarios randomly drop a learner robot. 6 of 30 scenarios are completion-infeasible. Evaluations repeated 3 times with different random seeds.

**Task success and adaptation accuracy (real-world)**

| Condition | N scenarios | Mean achieved | Rate |
|---|---|---|---|
| Task success | 24 | 22 ± 1.00 | 92% |
| Best adaptation (overall) | 30 | 26.33 ± 1.15 | 88% |
| Same item available | 11 | 9.33 ± 0.58 | 85% |
| Same-class item available | 13 | 11.33 ± 1.15 | 87% |
| Completion infeasible (correct idle) | 6 | 5.67 ± 0.58 | 94% |

Task assignment feasibility was 100% across all learner robots (Table 2): all assigned tasks were within each robot's physical capability scope.

**Latent intention space analysis**

120 held-out trajectories (3 per robot per task, unseen during training):

| Metric | Value |
|---|---|
| Global interclass cosine distance | 0.997 ± 0.003 |
| Semantic separation ratio (interclass / intraclass) | 3.764 |
| Embodiment alignment ratio | 3.046 |

Intraclass distances ranged from 0.023 (delivery tasks — nearly identical across robots) to 0.499 (cup-class picking — reflecting within-class appearance variation). Same-item intraclass distances were consistently much lower than same-class distances (e.g., 0.119 vs. 0.499 for cups), confirming fine-grained object-level specificity.

**Simulation comparison against baselines**

*Monitoring task (4 robots × 500 runs × 3 seeds = 1500 scores per pair):*

- vs. density-based baseline on 8 distributional-mismatch pairs: $\Delta = 1.40$, 95% CI (1.01, 1.79), all $P < 0.001$
- vs. description-based baseline on 4 capability-mismatch pairs: $\Delta = 0.94$, 95% CI (0.84, 1.04), all $P < 0.001$

*Item picking task (3 UR5 robots, 18 items, 5 classes; 9 robot pairs):*

- vs. density-based: $\Delta = 1.11$, 95% CI (1.08, 1.14), all $P < 0.001$
- vs. description-based: $\Delta = 0.63$, 95% CI (0.55, 0.70), all $P < 0.001$

The description-based method produced a mean score of $-1$ for the Pepper–Carter pair (consistent execution of incorrect actions). Neither baseline could detect infeasible tasks.

## Conclusion

IAIL shows that grounding cross-robot imitation in shared natural-language intentions — rather than motor correspondences or distributional alignment — yields robust behavior transfer across robots with fundamentally different embodiments and motion modalities. A single shared intention space, jointly learned by all robot-specific motion encoders and one annotation encoder, supports both agent-to-agent and team-to-team imitation without requiring paired demonstrations or predefined role assignments. Across 30 real-world scenarios with 7 heterogeneous robots, IAIL achieved 92% task success and 88% best-adaptation accuracy, with 100% feasibility in task assignment. Simulation experiments show statistically significant, large-effect-size improvements over density-based and description-based baselines under both distributional and capability mismatch.

## Impact

IAIL resolves a concrete scalability barrier: prior cross-embodiment IL either requires manually curated paired datasets for every demonstrator-learner combination or falls apart when robots have different task distributions or capability constraints. By using language annotations as the universal glue between motion spaces, the framework enables any robot with a pretrained repertoire to participate in an imitation chain with zero additional pairing effort. The conservative idle behavior — staying inactive when no feasible intention match exists — is a practically important safety property that the baselines cannot replicate. The acknowledged limitation of a single fixed threshold $\epsilon_{\mathrm{aligned}}$ shared across robots leaves room for per-robot calibration to push performance further. The preliminary integration with LLMs (replacing trajectory demonstrations with language instructions, shown in supplementary movie S3) suggests a clear path toward connecting foundation models with physical heterogeneous fleets, which is likely to be the most impactful downstream direction from this work.
%% end notes %%

## Notes & Highlights%% begin obsidian-freeform-notes %%

### Freeform Notes%% end obsidian-freeform-notes %%

### From Zotero

%% Import Date: 2026-06-26T11:53:50.297-04:00 %%
