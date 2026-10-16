Canonical: https://foundation.aerospace-superintelligence.com/

Aero Foundation — aerospace AI research and tool-verified learning                [Skip to content](https://foundation.aerospace-superintelligence.com/#main) 

AERO-FOUNDATIONAEROSPACE SUPERINTELLIGENCE · RESEARCH PROGRAM   

 Domain model · Tool-verified rewards 

# An aerospace model that learns from the tools engineers trust.

aero-foundation specializes an open-weights language model for flight control, aerodynamics, and handling qualities. It is trained with continued pretraining, then reinforcement learning in which every reward comes from an engineering tool that computes a checkable answer.

 [VIEW ON GITHUB →](https://github.com/aerospace-superintelligence/aero-foundation) [HOW IT WORKS ↓](https://foundation.aerospace-superintelligence.com/#approach)  

 Phase 0 · founding Apache-2.0 Ground tool · not airborne DO-330 qualification path   

Mission

## Engineering judgment, shaped by engineering checks.

Aerospace already has rigorous ways to tell a right answer from a wrong one. General-purpose models rarely use them, so they produce designs that read well and fail the first check. We train on the checks themselves.

01

### Flight control design

Gains and architectures that meet stability margins, bandwidth, and time-response requirements.

02

### Aerodynamic databases

Set up and run analyses, then assemble coefficient tables and derivatives that agree with the tools.

03

### Handling qualities

Compute modal characteristics and grade them against MIL-F-8785C and MIL-STD-1797A flying-qualities criteria.

04

### Requirements traceability

Link requirements to design artifacts and verification cases, in the style ARP4754B and DO-178C expect.

Approach

## Propose. Check. Reward. Update.

Continued pretraining on a license-clean aerospace corpus gives the model the vocabulary. Reinforcement learning gives it judgment: the model proposes, a sandboxed tool checks, and the check is the reward. No learned reward model sits between the physics and the gradient.
     Fig. 1 · Training loop. Tool versions are pinned and hashed so any reward can be reproduced.   

Verifiers & tools

## Rewards computed, not judged.

Each verifier runs in a sandbox with timeouts and input validation. A tool that fails to converge returns zero reward, never a crash or a default pass. Where two tools overlap, we cross-check them to expose blind spots.

ToolChecksExample reward

[JSBSim](https://github.com/JSBSim-Team/jsbsim)Nonlinear 6-DOF flight: trim, time responses, closed-loop behaviorStep response meets rise-time and overshoot specs

[AVL](https://web.mit.edu/drela/Public/web/avl/)Vortex-lattice aero: lift, induced drag, moments, stability derivatives, neutral pointGeometry trims at the target CL with static margin in band

[XFOIL](https://web.mit.edu/drela/Public/web/xfoil/)2-D airfoil polars, transition, pressure distributionAirfoil meets a cl/cd target at the given Reynolds number

[OpenVSP](https://openvsp.org/)Parametric geometry, VSPAERO analysisValid geometry matching area and span constraints

[python-control](https://python-control.readthedocs.io/)Poles, gain and phase margins, bandwidth, step metricsLoop meets margins with the specified crossover

HQ criteriaMIL-STD-1797A / MIL-F-8785C modal criteria: short period, CAP, Dutch roll, roll mode, spiral, bandwidthLevel 1 in every required flight phase

Certification philosophy

## The model proposes. Qualified tools decide.

aero-foundation is a ground-based engineering tool, not airborne software. The long-term path is tool qualification under DO-330, within programs that follow ARP4754B, DO-178C, and DO-331.

### The model is a proposer

Non-deterministic and hard to specify, so its outputs are treated as unverified until checked. We do not plan to claim certification credit for the model itself.

### The checkers are qualifiable

Deterministic, version-pinned tools whose behavior can be specified, tested, and placed under configuration management. Credit, if ever claimed, comes from independent verification of outputs.
    
- ARP4754B

- DO-178C

- DO-330

- DO-331

- MIL-STD-1797A

Traceability is built in from day one: data provenance manifests, training configurations, weight hashes, verifier hashes, and per-task evidence logs. Qualification is granted within a specific certification project by the relevant authority; nothing here is qualified today. This is engineering research, not regulatory advice.

Roadmap

## Evidence before scale.

Each phase ends with a checkable exit criterion. Compute estimates in the experiment plan are stated as assumptions.

Phase 0 · now

### Foundations

Verifier harness, task bank, eval suite, provenance pipeline, base-model baselines.

ExitReproducible baselines for two or more open models from one command. 

Phase 1

### RL with tool rewards

RLVR on a small base model with Phase 0 verifiers; no continued pretraining.

ExitSignificant gain on held-out tool-graded tasks, no general regression. 

Phase 2

### Continued pretraining

Aerospace corpus with license tagging, dedup, and decontamination.

ExitDomain gains without catastrophic forgetting. 

Phase 3

### CPT + RL at scale

Combined stages; closed-loop JSBSim, aero databases, traceability verifiers.

ExitBeats the strongest prompted baseline on the full suite. 

Phase 4

### Qualification hardening

TOR and TQP drafts for the verifier tools; automated evidence.

ExitEvidence-package dry run against DO-330 objectives.   

Research

## Start with what exists.

Every source in these documents was opened and checked. Nothing is invented; everything is paraphrased.
   
- [landscape.mdDomain LLMs for science and engineering, LLMs driving engineering tools, RL with verifiable rewards, and LLM work in aerospace and controls.](https://github.com/aerospace-superintelligence/aero-foundation/blob/main/research/landscape.md)
 
- [data-sources.mdCandidate corpora with licenses, and what needs a paid license.](https://github.com/aerospace-superintelligence/aero-foundation/blob/main/research/data-sources.md)
 
- [verifiers.mdTools for the training loop, reward design, and computable handling-qualities criteria.](https://github.com/aerospace-superintelligence/aero-foundation/blob/main/research/verifiers.md)
 
- [certification.mdQualifying a model-based ground tool under DO-330, and the evidence it takes.](https://github.com/aerospace-superintelligence/aero-foundation/blob/main/research/certification.md)
 
- [experiment-plan.mdPhased experiment plan with compute assumptions.](https://github.com/aerospace-superintelligence/aero-foundation/blob/main/research/experiment-plan.md)

 Status 

## Founding phase. No model yet.

The repository holds research and planning documents. There is no trained model and no capability claim. Work is tracked in GitHub issues under the Phase 0 milestone, on four tracks: data, verifiers, eval, and certification.

 [OPEN ISSUES →](https://github.com/aerospace-superintelligence/aero-foundation/issues) [CONTRIBUTE →](https://github.com/aerospace-superintelligence/aero-foundation/blob/main/CONTRIBUTING.md)   

phase0 · foundations 

modelnone trained 

claimsnone 

licenseApache-2.0 

scopeground tool   

 Citation 

## Cite this work
 @software{aero_foundation_2026, title = {aero-foundation: an aerospace engineering model trained with tool-verified rewards}, author = {{Aerospace Superintelligence contributors}}, year = {2026}, url = {https://github.com/aerospace-superintelligence/aero-foundation}, license = {Apache-2.0} }
