# Experiment plan

*Status: draft, 2026-10-15. Every compute and cost figure below is an **assumption** for planning, not a measurement. Each phase's first task is to replace its assumptions with measured numbers.*

## Hypotheses

| ID | Hypothesis | How we would falsify it |
| --- | --- | --- |
| H1 | RL with rewards from engineering tools improves a small open model's success on tool-graded aerospace design tasks beyond what prompting achieves. | No significant gain over the base model with the best prompt and the same sampling budget on held-out tasks. |
| H2 | The gains transfer to aircraft, flight conditions, and task variants never seen in training. | Gains vanish on the held-out split (new aircraft classes or flight phases). |
| H3 | Continued pretraining on an aerospace corpus makes later RL more sample-efficient. | CPT+RL does not reach a given reward sooner, or reaches a lower plateau, than RL alone. |
| H4 | Domain training can be done without degrading general capability. | Regression beyond a preset tolerance on a fixed general suite (math, code, knowledge, instruction following). |
| H5 | A model trained this way can produce outputs whose acceptance depends only on deterministic, qualifiable checkers. | Any eval task where a correct grade requires trusting the model rather than a tool. |

## Fixed design principles

- **Procedurally generated tasks.** Tasks are generated from parameterized aircraft models and requirement templates, so we can make unlimited fresh instances and keep held-out splits clean. Splits are by aircraft class and flight phase, not by random instance.
- **Rewards from tools only.** No learned reward model. Unparseable output, tool non-convergence, or a timeout scores zero.
- **Reproducibility.** Every run records: commit, config, seeds, base-model weight hash, tool versions and binary hashes, task-generator version, and the data manifest hash.
- **Statistics.** Report means with bootstrap 95% confidence intervals over tasks, and at least 3 training seeds for any headline result. Report pass@1 and pass@k at a matched sampling budget.
- **Reward-hacking audits.** For every RL run, sample high-reward outputs and check them by hand and against a second, independent tool where one exists.
- **Base model choice.** Prefer an open-weights base with documented pretraining data where possible, since data provenance is part of the evidence trail (see [certification.md](certification.md)). Pick candidates in Phase 0 from the current open-weights field; do not commit to a family in advance.

## Phase 0: Foundations

**Goal:** a harness that turns "task + model output" into a reproducible score, and baseline numbers for existing models.

Deliverables:

1. Verifier harness: containerized wrappers for python-control, AVL, XFOIL, and JSBSim with pinned versions, timeouts, input validation, and result caching. See [verifiers.md](verifiers.md).
2. Handling-qualities module: computes short-period, phugoid, Dutch roll, roll, spiral, and CAP metrics from a linear model, and returns the Level per criterion.
3. Task generators for at least three task families: (a) longitudinal SAS/autopilot gain design against HQ and margin specs, (b) trim and static-margin design with AVL, (c) airfoil selection or modification with XFOIL.
4. Eval suite v0: ~500 tasks per family across train-like and held-out splits, plus a small hand-checked "gold" set.
5. Data provenance pipeline v0: manifest schema (source URL, retrieval date, license, distribution statement, SHA-256), and an ingest of a first public-domain slice. See [data-sources.md](data-sources.md).
6. Baselines: two or more open-weights models at roughly 1–8B parameters, plus one strong prompted model as a ceiling reference.

Compute assumptions:

| Item | Assumption | Estimate |
| --- | --- | --- |
| Baseline eval, per model | 1,500 tasks × 8 samples × ~2k tokens ≈ 24M generated tokens | A few GPU-hours on one H100-class GPU with a batched inference server |
| Verifier load, per model | 12,000 checks; python-control ~0.01–0.1 s, AVL/XFOIL ~0.5–5 s each | ~1–10 CPU-hours |
| Total Phase 0 | 3 models, repeated runs while debugging | ~20–50 GPU-hours, ~100 CPU-hours |

**Exit:** baselines for at least two open models reproducible from one command, with confidence intervals.

## Phase 1: RL with tool rewards (no continued pretraining)

**Goal:** test H1, H2, H4 as cheaply as possible.

Design:

- Algorithm: a GRPO-style group-relative policy-gradient method (no value network), with a KL or reference-policy constraint swept as a hyperparameter.
- Model sizes: a pilot at ~1.5B parameters, then ~7–8B.
- Reward: a weighted sum of criteria met, with an all-criteria bonus; zero for format failures. Shaping terms are logged separately so their effect can be removed later.
- Ablations: reward shaping vs. binary pass/fail; group size; with and without a short supervised warm-up on tool-call format.

Compute assumptions (order of magnitude):

| Run | Assumption | Estimate |
| --- | --- | --- |
| 1.5B pilot | 300 steps × 512 rollouts × ~1.5k tokens | ~20–60 GPU-hours |
| 7–8B main run | 300 steps × 2,048 rollouts × ~1.5k tokens; generation dominates | ~150–400 GPU-hours per seed |
| Verifier load | 2,048 checks per step; python-control tasks are milliseconds, AVL/XFOIL tasks ~1–5 s | 64–128 CPU cores keep pace if checks run alongside generation |
| Phase 1 total | 3 seeds of the main run + pilot + ablations | ~1,000–2,000 GPU-hours |

At an assumed US$2–4 per H100-hour, Phase 1 is roughly US$2k–8k of GPU time.

**Exit:** a statistically significant improvement on held-out tool-graded tasks, and general-suite regression within tolerance.

## Phase 2: Continued pretraining

**Goal:** test H3 and H4.

Design:

- Corpus: the license-clean slice from Phase 0's pipeline, extended. Assumption: 5–20B tokens of domain text after filtering, mixed with 20–50% general "replay" data to limit forgetting.
- Decontamination against every eval split (n-gram and near-duplicate matching).
- Compare: base → RL vs. base → CPT → RL, with the same RL budget.

Compute assumptions, using training FLOPs ≈ 6 × parameters × tokens and ~40% utilization of an H100-class GPU (~400 TFLOP/s effective):

| Model | Tokens | FLOPs | Estimate |
| --- | --- | --- | --- |
| 8B | 10B | ≈ 4.8 × 10²⁰ | ≈ 330 GPU-hours |
| 8B | 20B | ≈ 9.6 × 10²⁰ | ≈ 670 GPU-hours |
| 32B | 20B | ≈ 3.8 × 10²¹ | ≈ 2,700 GPU-hours |

**Exit:** domain perplexity and downstream gains without general regression beyond tolerance.

## Phase 3: CPT + RL at scale

**Goal:** a model worth using.

- Add verifiers: closed-loop JSBSim (nonlinear time responses, gust and failure cases), aero database assembly (cross-checked AVL vs. VSPAERO), and requirements-traceability tasks checked by a deterministic trace checker.
- Multi-turn tool use: the model calls tools itself during the episode, and the final answer is graded by the same tools in a separate, clean sandbox.
- Scale: 8B and possibly ~32B. Assumption: 2,000–10,000 GPU-hours depending on model size and verifier mix.

**Exit:** beats the strongest prompted baseline on the full eval suite, with reward-hacking audits clean.

## Phase 4: Qualification-oriented hardening

**Goal:** show what a DO-330 evidence package would look like for the verifier tools.

- Draft Tool Operational Requirements and a Tool Qualification Plan for the HQ module and one aero verifier.
- Automate the evidence trail: every eval result linked to model, data, tool, and task-generator versions.
- Dry-run the package against DO-330 objectives with an experienced reviewer.

Compute: negligible beyond re-running evals.

## Highest-leverage first experiment

**HQ-graded flight control design with RL, on a small model, with millisecond-scale verifiers.**

1. Task: given a linearized aircraft model (procedurally varied stability derivatives and flight conditions), propose pitch-axis augmentation gains so the closed loop meets Level 1 short-period damping and CAP limits plus gain and phase margin targets. Start with the MIL-F-8785C limits, which [verifiers.md](verifiers.md) checked against the source tables, and add the MIL-STD-1797A forms once their limits are confirmed.
2. Verifier: python-control plus the HQ module. Each check takes milliseconds, so the reward signal is nearly free and fully deterministic. A second, independent implementation, such as the organization's Rust [flight tools](https://aerospace-superintelligence.com/tools/), cross-checks a sample of grades.
3. Train a ~1.5B model, then a ~7–8B model, with GRPO-style RL; no continued pretraining.
4. Measure held-out success on unseen aircraft classes and flight phases against the base model, a prompted frontier model, and a classical design baseline.
5. It tests the core hypothesis (H1, H2) for about a hundred GPU-hours, and the harness it needs is the foundation for every later phase.

Closest prior work (details in [landscape.md](landscape.md)):

- [BeamPERL](https://arxiv.org/abs/2603.04124) ran RL with solver-checked rewards on a 1.5B model for beam mechanics. It found the model learned templates that broke when the problem setup changed, which is why our splits hold out whole aircraft classes and why checkpoints are selected on held-out robustness.
- [Reinforcement Learning with Verifiable Physics](https://arxiv.org/abs/2607.10474) combines hard validity checks with continuous physics-error terms in one reward. That is the template for mixing pass/fail HQ Levels with continuous margin terms.
- [ControlAgent](https://arxiv.org/abs/2410.19811) provides a control-design task set whose designs are checked by computation against specifications. It is a candidate source of tasks and a baseline agent.
- A study of [rollout-verified fine-tuning for building HVAC control](https://arxiv.org/abs/2607.27914) found no sustained gain from simulator-scored fine-tuning. It is a reminder to budget for a negative result.

## Risks

| Risk | Mitigation |
| --- | --- |
| Reward hacking (gains that satisfy linear criteria but are unrealistic) | Bound gains physically; add actuator-rate and noise-sensitivity checks; audit samples; cross-check with nonlinear simulation in Phase 3 |
| Linear-model tasks too easy, so RL learns a formula | Vary plant structure and add requirements that conflict; include tasks needing iteration with the tool |
| Verifier bugs become model "knowledge" | Unit-test verifiers against published worked examples; cross-check two tools; version and hash everything |
| Data licensing errors | Manifest-first ingest; default-deny anything without a confirmed license |
| Compute estimates wrong | Phase 0 measures real throughput before any Phase 1 budget is committed |
