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

---
> [!success] Key Definition (Page 1)
> Intention-Aligned Imitation Learning (IAIL) framework, a behavior adaptation approach that extends the conventional scope of IL by enabling robots to reproduce motions demonstrated by heterogeneous peers, even in previously unseen situations.%% begin 6KAP2CXS %%%% end 6KAP2CXS %%

---
> [!note] Key Passage (Page 1)
> Inspired by human cultural learning, IAIL aligns and adapts robot motions on the basis of high-level intentions annotated in natural language rather than by directly copying motor movements.%% begin 89HDDXKM %%%% end 89HDDXKM %%

---
> [!abstract] Literature Review (Page 1)
> For differences in physical embodiments, such as robots with different degrees of freedom, body structures, or actuator types, motion correspondence can be established on the basis of invariant body components (7, 8) or transitions in environmental states (9, 10).%% begin LY4MCTAD %%%% end LY4MCTAD %%

---
> [!abstract] Literature Review (Page 1)
> For environmental variations, such as different lighting conditions, varied camera views, or interactable objects with changing colors or textures, motion correspondence can be established by finding a common feature space with domain confusion approaches%% begin PVP8P6T3 %%%% end PVP8P6T3 %%

---
> [!abstract] Literature Review (Page 1)
> they become inadequate in cases involving substantial differences%% begin BC89U7E5 %%%% end BC89U7E5 %%

---
> [!info] Methods (Page 2)
> We define an intention as the goal or outcome of a motion and represent it using a human-annotated language description that abstracts away control and embodiment details. We then construct a shared intention space by aligning motion embeddings with their corresponding annotations, enabling intention similarity to be measured across heterogeneous robots.%% begin 4MSDIM8D %%%% end 4MSDIM8D %%

---
> [!info] Methods (Page 2)
> At inference time, a learner retrieves the most demonstration-relevant behavior from its pretrained repertoire by matching intentions.%% begin H4RS4MWZ %%%% end H4RS4MWZ %%

---

![[source-notes/chenCrossrobotBehaviorAdaptation2026/image-2-x36-y425.png]]%% begin ZLMMKQYN %%%% end ZLMMKQYN %%

---
> [!abstract] Literature Review (Page 2)
> This aligns with the cognitive science literature on rational imitation, which suggests that learners, even infants, prioritize reproducing a demonstrator’s inferred goals over their exact movement patterns%% begin 88JLIBLB %%%% end 88JLIBLB %%

---
> [!note] Key Passage (Page 3)
> The process consists of three key stages: context-aware motion generation, motion intention extraction, and motion association based on intention similarity.%% begin MRRVAA4P %%%% end MRRVAA4P %%

---
> [!info] Methods (Page 3)
> we generate feasible actions for the learner robot i on the basis of its current state, defined as the information perceived by its onboard sensors, which captures its operational context in the environment. Each action is a single command or a sequence of commands that the learner can execute to achieve a specific goal in its environment.
- **My Note:** Context-aware motion generation%% begin V6DI8XSY %%%% end V6DI8XSY %%

---
> [!info] Methods (Page 3)
> both the learner’s generated actions and the demonstrator’s executed actions are projected into a shared motion-intention embedding space. The projection is performed using a motion encoder fψi for the learner and fψj for the demonstrator, each trained jointly with a shared annotation encoder fξ.
- **My Note:** Motion intention extraction%% begin UVS2YP57 %%%% end UVS2YP57 %%

---
> [!info] Methods (Page 3)
> we compare the embedded representation of the demonstrator’s motion with the embeddings of the learner’s candidate actions generated in the first stage. The system selects the candidate whose embedding is closest to the demonstrator’s embedding in intention space, ensuring that the chosen action is executable for the learner and aligned with the demonstrated objective.
- **My Note:** Motion association based on intention similarity%% begin UDEJZ89C %%%% end UDEJZ89C %%

---
> [!info] Methods (Page 3)
> We evaluated the IAIL framework in a complex real-world environment involving seven heterogeneous robots performing multistep tasks.%% begin ZP2JH6G6 %%%% end ZP2JH6G6 %%

---
> [!info] Methods (Page 3)
> The tasks included the following: monitoring user activities at one of four locations (M1 to M4), fetching items at one of three areas (I1 to I3), and delivering items to one of two destinations (D1 and D2).%% begin I5KLJUUJ %%%% end I5KLJUUJ %%

---

![[source-notes/chenCrossrobotBehaviorAdaptation2026/image-3-x63-y130.png]]%% begin TRIWYKBA %%%% end TRIWYKBA %%

---
> [!info] Methods (Page 4)
> ecause of embodiment and workspace constraints, each robot had access to a subset of areas and supported distinct capabilities. Tello was a drone that could monitor all four locations M1 to M4. Dual-Arm had two manipulators and could retrieve items from the drawer at I3, whereas Single-Arm could retrieve items placed on the table at I2. Spark was a mobile transporter that could deliver items to D1 and D2. Cuboat operated in the water pool and could monitor locations M1 and M2. Diablo was a height-adjustable mobile robot that could monitor all four locations M1 to M4 and deliver items to D1 and D2. Pepper was a mobile manipulator that could access items on the table at I1 and also deliver items to D1 and D2.%% begin W29MFUVS %%%% end W29MFUVS %%

---

![[source-notes/chenCrossrobotBehaviorAdaptation2026/image-4-x82-y383.png]]%% begin QBGEMCWS %%%% end QBGEMCWS %%

---
> [!success] Key Definition (Page 5)
> A task was considered successful when the exact demonstrated item, or an item in the same class, was delivered to the correct location.%% begin 98DU76GR %%%% end 98DU76GR %%

---
> [!note] Key Passage (Page 5)
> the robots successfully completed an average of 22 of 24 scenarios, achieving an overall success rate of 0.92.%% begin GBV5VRDI %%%% end GBV5VRDI %%

---
> [!success] Key Definition (Page 5)
> This metric measured the percentage of instances in which the robots achieved the best possible outcome under three conditions.
- **My Note:** Adaptation accuracy%% begin HSEG4MX3 %%%% end HSEG4MX3 %%

---

![[source-notes/chenCrossrobotBehaviorAdaptation2026/image-5-x37-y52.png]]%% begin TPBP2CKF %%%% end TPBP2CKF %%

---

![[source-notes/chenCrossrobotBehaviorAdaptation2026/image-6-x36-y52.png]]%% begin HDMYI8EI %%%% end HDMYI8EI %%

---
> [!note] Key Passage (Page 9)
> We evaluated the imitation performance between all possible demonstrator-learner robot pairs. On the basis of the alignment results after execution, we assigned a score of 1 if the learner robot monitored the correct target or picked the same item as in the demonstration, a score of 0.5 if the learner robot picked an item in the same class, a score of 0 for skipping the action, and a score of −1 for performing an irrelevant action.%% begin 876D3FXV %%%% end 876D3FXV %%

---

![[source-notes/chenCrossrobotBehaviorAdaptation2026/image-9-x81-y116.png]]%% begin JNGSKRCQ %%%% end JNGSKRCQ %%

%% Import Date: 2026-06-30T00:14:31.380-04:00 %%
