# Verifiers for aero-foundation: tool-computed rewards for aerospace RL

**Status:** research note, checked 2026-10-15
**Scope:** engineering tools that can act as reward oracles for continued pretraining plus RL. The model proposes a design or an answer, and a tool computes a checkable result.
**Target uses:** flight control design, aero database generation, handling-qualities (HQ) analysis, requirements traceability.

> **Ground rules for this document**
> - License, version, and maintenance facts were confirmed against each project's official site, LICENSE file, GitHub/GitLab API metadata, or PyPI metadata on 2026-10-15. Each one links to its source. Anything not confirmed is marked **Unverified**.
> - Every **throughput number is an assumption** (a planning estimate, not a benchmark), unless it is quoted from a source and attributed to it.
> - License comments are engineering notes, **not legal advice**. Have counsel review before you distribute anything.
> - Handling-qualities numbers appear only where we read them directly from a fetched copy of the defining document. Everywhere else the text says "limits per the standard (verify)".

---

## 1. Summary matrix

| Tool | Domain | License (as confirmed) | What it verifies | Reward type | Est. cost/eval (assumption) |
|---|---|---|---|---|---|
| **JSBSim** | 6-DoF nonlinear flight dynamics | LGPL-2.1 (COPYING); source headers say "version 2 … or any later version" | Trim feasibility, linearized A/B/C/D, closed-loop time responses, envelope checks | Pass/fail on spec plus normalized error vs. a hidden reference | ~0.1–1 s for trim+linearize; ~1–10 s for a 30–60 s time sim (1 core) |
| **AVL** | Vortex-lattice aero + flight-dynamics eigenmodes | GPL (the official page links the GPL **v2** text) | CL, CDi, Cm, span efficiency, stability/body-axis derivatives, trim, eigenvalues, A/B matrices | Normalized error vs. AVL-computed reference; constraint satisfaction | ~0.05–0.5 s per run case (1 core) |
| **XFOIL** | 2-D viscous/inviscid airfoil analysis | GPL (the official page links the GPL **v2** text) | Cl, Cd, Cm, transition, Cp at given Re/M/α | Constraint satisfaction plus normalized error; non-convergence gives reward 0 | ~0.05–0.5 s per α point; ~2–10 s per polar (1 core) |
| **OpenVSP (+VSPAERO)** | Parametric geometry + VLM/panel aero | NOSA 1.3 | Geometry validity, areas/volumes, VLM/panel forces and moments, sweeps | Constraint satisfaction; cross-check vs. AVL | ~5–120 s per VSPAERO case (1 core) |
| **python-control** | Linear control analysis/design | BSD-3-Clause | Poles, margins, bandwidth, step metrics, LQR/placement, linearization, HQ metrics | Pass/fail per spec; level-based HQ reward | ~1–50 ms per check (1 core) |
| Slycot | SLICOT wrappers (Riccati, H∞, etc.) | GPL-2.0 (LICENSE.txt) | Back-end for advanced python-control routines | n/a (dependency) | n/a |
| SciPy | Numerics (ODE, linalg, signal, optimize) | BSD-3-Clause | Eigenvalues, ODE time responses, root finding | Building block | ms |
| **NeuralFoil** | Learned airfoil surrogate (trained on XFOIL runs) | MIT | Cl, Cd, Cm, transition + `analysis_confidence` | Fast shaping reward; audited against XFOIL | ~1–6 ms per run (README table) |
| AeroSandbox | Aircraft design optimization, VLM, AVL/XFOIL interfaces | MIT | Multidisciplinary design feasibility, mass, aero buildup | Constraint satisfaction | ~0.01–10 s (assumption, depends on model) |
| OptVL | AVL fork with direct Python API + gradients | GPL-3.0 (LICENSE) | Same as AVL, in-process | Same as AVL | ~0.01–0.3 s per case (assumption) |
| OpenAeroStruct / OpenMDAO | VLM + beam aerostructural MDO | Apache-2.0 (LICENSE files) | CL/CD, structural failure index, stability derivatives | Constraint satisfaction with penalties | ~1–60 s per analysis (assumption) |
| CasADi | NLP/OCP with automatic differentiation | LGPL-3.0 (LICENSE.txt; PyPI says LGPLv3+) | Trim/trajectory feasibility, KKT residuals | Feasibility + optimality gap | ~0.01–10 s (assumption) |
| SU2 | RANS/Euler CFD + adjoint | LGPL-2.1 (LICENSE.md, site) | High-fidelity coefficients | Audit only (too expensive for the RL inner loop) | CPU-hours (assumption) |
| ArduPilot SITL | Autopilot software-in-the-loop | GPL-3.0 | Parameter sets / mission behavior in closed loop | Episode pass/fail | ~real-time to a few× (assumption) |
| PX4 SITL (SIH) | Autopilot software-in-the-loop | BSD-3-Clause | Same as ArduPilot; headless SIH | Episode pass/fail | ~real-time to a few× (assumption) |
| FlightGear | Full flight simulator (visual, uses FDMs) | GPL-2.0-or-later (GitLab LICENSES/) | Visualization / demo, not a verifier | Not recommended as a reward source | Real-time (assumption) |
| OpenModelica | Acausal multi-physics modeling (Modelica) | Dual: OSMC-PL 1.8 or AGPL-3.0 | Actuator/hydraulic/electrical subsystem models, FMU export | Pass/fail on simulated spec | ~1–30 s compile+sim (assumption) |
| Z3 | SMT solver | MIT | Consistency of formalized requirements; spec entailment | Satisfiable/unsatisfiable + unsat-core match | ~1 ms–1 s (assumption) |
| Doorstop / StrictDoc / sphinx-needs | Requirements-as-code + trace graphs | LGPL-3.0 / Apache-2.0 / MIT | Link integrity, trace coverage | Set-overlap F1 vs. hidden gold trace | ~0.1–2 s (assumption) |

---

## 2. Tool-by-tool

Each entry has a common layout: **Confirmed facts**, **What it can check**, **Reward formulation**, **Throughput**, **License implications**, and **Sample task**.

### 2.1 JSBSim (required)

**Confirmed facts**
- **What it is:** an open-source C++ flight dynamics model (FDM). It provides nonlinear 6-DoF dynamics and XML-configured flight control, aerodynamics, propulsion, and gear, plus WGS84 and the 1976 standard atmosphere ([GitHub](https://github.com/JSBSim-Team/jsbsim)).
- **License:** the `COPYING` file is the GNU LGPL v2.1. Source headers (e.g. `src/FGFDMExec.h`) grant LGPL "version 2 of the License, or (at your option) any later version", and the PyPI classifier is LGPLv2+ ([COPYING](https://github.com/JSBSim-Team/jsbsim/blob/master/COPYING), [PyPI](https://pypi.org/project/jsbsim/)). The README notes that the MATLAB interface is BSD and the Unreal reference app is MIT.
- **Interfaces:**
  - Standalone console/batch executable
  - Python module (`pip install jsbsim`; wheels for Python 3.10–3.14 per README)
  - C++ `FGFDMExec` API
  - MATLAB S-function
  - Unreal plugin
- **Trim and linearization:** the source tree has `FGTrim`, `FGSimplexTrim`, and `FGLinearization` under [`src/initialization/`](https://github.com/JSBSim-Team/jsbsim/tree/master/src/initialization). The Python bindings expose `do_trim()` and an `FGLinearization` class with a `state_space()` method ([`python/jsbsim.pyx.in`](https://github.com/JSBSim-Team/jsbsim/blob/master/python/jsbsim.pyx.in)).
- **Maintenance:** latest release v1.3.2 (2026-10-08). Last push 2026-10-15 (GitHub API).

**What it can check**
- Whether a proposed trim exists. It reports trim success or failure and residual accelerations at a given altitude, speed, flight-path angle, and configuration.
- Linear models at trim: the A, B, C, D matrices, from which short-period, phugoid, Dutch roll, roll, and spiral modes follow (feeds §3).
- Closed-loop time responses of a proposed control law (written as JSBSim `<flight_control>` XML or as gains on an existing system). Examples are overshoot, settling time, and limit exceedance in α, nz, or bank.
- Envelope protection behavior: does α or nz stay within limits during a scripted maneuver?

**Reward formulation**
- **What the model emits:** either a parameter set (e.g. pitch-rate and integrator gains for an existing FCS template, written as JSON) or a JSBSim XML fragment for a `<flight_control>` channel.
- **Scalar reward:**
  - `r = 1[trim converged] · Π_i 1[spec_i met]` for hard specs, or
  - a smooth version `r = Σ w_i · clip(1 − |m_i − target_i| / tol_i, 0, 1)` over metrics m_i (overshoot, rise time, etc.).
  - For aero database tasks, compare the model's numeric answers against JSBSim-computed references using normalized error `e = |y − y_ref| / max(|y_ref|, ε)` and `r = exp(−e/τ)`.
- **Anti-hacking:**
  - Validate XML against an allow-list schema: no `<script>` blocks, no file includes, no property writes outside the FCS subtree.
  - Clamp gains to physically plausible ranges.
  - Run in a container with no network and a wall-clock limit.
  - Evaluate on **hidden** flight conditions and perturbed mass/CG cases, not just the conditions named in the prompt.
  - Treat NaN/Inf states, or JSBSim's own crash detection, as reward 0.
  - Never trust a trim the model reports; re-trim inside the verifier.

**Throughput.** Assumption: ~0.1–1 s per trim+linearize and ~1–10 s for a 30–60 s scripted simulation on one CPU core. JSBSim runs much faster than real time in batch mode. Not benchmarked here.

**License implications.** LGPL allows dynamic linking or import from proprietary code, with conditions that apply when you distribute. Running JSBSim internally as a subprocess or a Python import inside a training cluster is generally considered low-risk. If you ship a modified JSBSim, you must provide those modifications under the LGPL. *Not legal advice.*

**Sample task**
> *Prompt:* "For the bundled `c172x` model at 4,000 ft, 100 KTAS, wings level, propose a pitch-attitude-hold PI gain set `{Kp, Ki}` (elevator per rad of θ error) so that a 5° θ step gives overshoot < 10% and 2% settling < 6 s. Answer as JSON."
>
> *Grading:*
> 1. Trim at a hidden altitude/speed perturbation (±10%).
> 2. Insert the gains into a fixed FCS template.
> 3. Simulate a 5° step.
> 4. Score `r = 0.5·1[overshoot<10%] + 0.5·1[t_s<6 s]`, multiplied by `1[no limit exceedance]`.

---

### 2.2 AVL — Athena Vortex Lattice (required)

**Confirmed facts**
- **What it is:** aerodynamic and flight-dynamic analysis of rigid aircraft of arbitrary configuration. It uses an extended vortex-lattice model for lifting surfaces and a slender-body model for fuselages and nacelles ([official page](https://web.mit.edu/drela/Public/web/avl/)).
- **License:** the page says AVL is released under the GNU General Public License and links to [`gpl.txt`](https://web.mit.edu/drela/Public/web/gpl.txt). That file is **GPL Version 2, June 1991**. The page does not say whether the grant is "v2 only" or "v2 or later".
- **Interfaces:** Fortran CLI, menu-driven. Started as `avl xxx`, which reads `xxx.avl`, `xxx.run`, and `xxx.mass`. The user guide refers to running "in batch mode using a command file" and advises turning off graphics for that ([avl_doc.txt](https://web.mit.edu/drela/Public/web/avl/avl_doc.txt)). No official Python interface; see OptVL (§2.11) and AeroSandbox (§2.9).
- **Maintenance:** AVL 3.52 is dated 09/02/25 in [version_notes.txt](https://web.mit.edu/drela/Public/web/avl/version_notes.txt). It fixed DERMATS moment derivatives being reported in body axes rather than stability axes, so **pin ≥ 3.52**.

**What it can check** (all per the user guide)
- Total forces and moments: CL, CD, Cl, Cm, Cn in stability axes.
- Trefftz-plane CDi and span efficiency e.
- Stability-axis (`ST`) and body-axis (`SB`) derivatives with respect to angles, rates, and controls.
- Hinge moments.
- Trim: operating variables (α, β, p, q, r, control deflections) constrained directly or via CL or moment targets. `C1` sets up level/banked flight and `C2` steady pitch-rate flight.
- Eigenvalues and the A/B state-space matrices from the `MODE` menu.
- Neutral point: the user guide we fetched does not document a direct Xnp output, so the verifier should **derive** it as `x_np = x_ref − c̄·Cm_α/CL_α` from `ST` derivatives. That is exact for a linear model about the reference point.

**Reward formulation**
- **What the model emits:**
  - (a) a parameter set (tail volume, CG location, incidence), which the verifier renders into a templated `.avl` file, or
  - (b) a complete `.avl` geometry file (harder, more hackable), or
  - (c) numeric answers to "what is Cm_α / static margin / trim elevator for this geometry".
- **Scalar reward:**
  - Answer tasks: `r = exp(−|y−y_ref|/(τ·|y_ref|))` per quantity, averaged.
  - Design tasks: `r = Π 1[constraint met]`, e.g. static margin in [5%, 15%] MAC, trim δe within ±10°, Dutch roll and spiral eigenvalues stable. Use soft penalties in early curriculum.
- **Anti-hacking:**
  - Prefer (a) over (b).
  - If accepting raw `.avl` files, parse and re-emit them through a strict whitelist parser. Bound panel counts (prevents runaway runtime), surface spans, chords, and dihedral. Reject degenerate geometry (zero area, crossing surfaces). Recompute reference quantities (Sref, cref, bref) from geometry instead of trusting the file header, because a model could otherwise scale coefficients by editing reference values.
  - Score against hidden flight conditions.

**Throughput.** Assumption: ~0.05–0.5 s per run case on one core for ≤ ~1,000 vortices, including process start. Batch dozens of cases per process to amortize startup.

**License implications.** GPL. Invoke the unmodified AVL binary as a **separate process** that communicates via files or pipes. The usual view is that this does not make the training harness a derivative work. If you modify AVL and distribute the binary (e.g. in a public verifier container), you must provide source under the GPL. Model weights trained on AVL *outputs* are not AVL code. *Not legal advice.*

**Sample task**
> *Prompt:* "Given this conventional-tail template (wing fixed; free variables: horizontal tail area `S_h` ∈ [0.5, 2.0] m², tail arm `l_h` ∈ [2, 5] m, CG `x_cg` ∈ [0.2, 0.5] m), choose values giving a static margin of 10 ± 2% MAC and CL_trim = 0.5 with |δe| ≤ 5°."
>
> *Grading:*
> 1. Render the template.
> 2. Run AVL `ST` at α from trim and derive Xnp.
> 3. Compute SM.
> 4. Run the trim case.
> 5. Score `r = 1[|SM−0.10| ≤ 0.02] · 1[|δe| ≤ 5°] · 1[converged]`.

---

### 2.3 XFOIL (required)

**Confirmed facts**
- **What it is:** an interactive, menu-driven program for design and analysis of subsonic isolated airfoils ([official page](https://web.mit.edu/drela/Public/web/xfoil/)). It offers:
  - viscous or inviscid analysis with free or forced transition, separation bubbles, and limited trailing-edge separation
  - lift and drag just beyond CLmax
  - Karman–Tsien compressibility correction
  - full and mixed inverse design
  - geometric redesign
  - polar save files
- **License:** GPL per the official page, which links to the same [`gpl.txt`](https://web.mit.edu/drela/Public/web/gpl.txt) (GPL v2). "Only" vs. "or later" is not stated.
- **Interfaces:** Fortran CLI. The official page does not document batch mode. It does provide sample session inputs (`sessions.txt`). Piping a keystroke script to stdin is common community practice, but it is **not documented on the official page (verify per version)**. A third-party Python binding, [DARcorporation/xfoil-python](https://github.com/DARcorporation/xfoil-python), is GPL-3.0 and was last pushed in 2021 (latest release 0.0.16, 2019), so treat it as unmaintained.
- **Maintenance:** Xfoil 6.996 is listed in the page revisions dated 2026-01-01.

**What it can check**
- Cl, Cd, Cm, Cp distribution, top/bottom transition x/c, and boundary-layer quantities, at given Re, M, Ncrit, and α (or target Cl).
- Polars and Cl_max, near stall only approximately.
- Geometry operations: thickness, camber, LE radius, flap deflection.

**Reward formulation**
- **What the model emits:** airfoil parameters (e.g. 18-parameter CST/Kulfan, matching NeuralFoil) or coordinates. Coordinates are riskier; re-panel them with XFOIL's `PANE`.
- **Scalar reward:**
  - Design tasks: `r = 1[t/c ≥ t_min] · 1[Cm ≥ Cm_min] · f(Cd)`, with f decreasing in Cd relative to a reference airfoil.
  - Answer tasks: normalized error against XFOIL's own result.
  - **Non-convergence at any required α → reward 0** (not an exception), logged separately.
- **Anti-hacking:**
  - Bound CST coefficients, and reject self-intersecting or non-closed shapes and extreme trailing-edge thickness.
  - Re-panel every input.
  - Require convergence at **neighboring** α values too. XFOIL can "converge" to spurious solutions at isolated points, and an RL policy will find them.
  - Use hidden Re/Ncrit/α combinations.
  - Kill the process at a timeout.

**Throughput.** Assumption: ~0.05–0.5 s per α point and ~2–10 s for a 20-point polar on one core. For comparison, the NeuralFoil README lists XFOIL at ~73 ms per single analysis on its benchmark.

**License implications.** Same as AVL: run it as a separate process; obligations attach if you distribute modified binaries. *Not legal advice.*

**Sample task**
> *Prompt:* "Propose CST parameters for an airfoil with t/c ≥ 12%, Cm_c/4 ≥ −0.05 at Cl = 0.6, Re = 1×10⁶, M = 0.1, Ncrit = 9, minimizing Cd at that Cl."
>
> *Grading:*
> 1. Check geometry validity.
> 2. Run XFOIL in Cl mode at 0.6 plus neighbors 0.5 and 0.7.
> 3. Score `r = 1[valid]·1[converged at all 3]·1[t/c≥0.12]·1[Cm≥−0.05]·clip((Cd_ref − Cd)/Cd_ref + 0.5, 0, 1)`, where Cd_ref is a hidden baseline airfoil.

---

### 2.4 OpenVSP + VSPAERO (required)

**Confirmed facts**
- **What it is:** a parametric aircraft geometry tool ([openvsp.org](https://openvsp.org/)). VSPAERO is its potential-flow solver, developed by David Kinney at NASA Ames. It runs thin-surface (VLM-style), thick-surface (panel), and hybrid cases and offers stability analyses under its Advanced settings ([NASA VSPAERO Basics](https://www.nasa.gov/reference/openvsp-vspaero-basics)). That NASA page warns that OpenVSP v3.45 introduced major VSPAERO changes (version 7), and its tutorials predate them.
- **License:** NASA Open Source Agreement **version 1.3** ([license page](https://openvsp.org/license.shtml); [LICENSE](https://github.com/OpenVSP/OpenVSP/blob/main/LICENSE)). GitHub's API reports `NOASSERTION`, so it is not auto-detected.
- **Interfaces:**
  - GUI
  - C++ API with Python bindings (the `python` directory in the distribution; the Python version must match the build)
  - AngelScript `.vspscript` files, runnable from the GUI or the `vspscript` command line ([API docs 3.55.0](https://openvsp.org/api_docs/latest/))
- **Maintenance:** OpenVSP 3.55.0 was announced 2026-10-12 on openvsp.org. The repo was last pushed 2026-10-12.

**What it can check**
- Geometry validity and derived quantities: wetted area, volumes, and the degenerate-geometry representation.
- VSPAERO forces and moments (CMx/CMy/CMz normalized by bref and cref as described by NASA), sweeps such as drag polars, and spanwise loads.
- Stability analyses exist, but **we did not confirm the exact derivative outputs of VSPAERO 7 from a fetched source (Unverified)**.

**Reward formulation**
- **What the model emits:** a parameter set applied to a baseline `.vsp3` via the Python API (preferred), or an AngelScript.
- **Scalar reward:** constraint satisfaction (e.g. wing area within 1%, CL at cruise α within tolerance) plus normalized error versus VSPAERO references. Use it as a **cross-check** verifier against AVL (§4).
- **Anti-hacking:**
  - **Never execute model-written AngelScript or Python** without a sandbox. Prefer a parameter-set interface where the verifier calls `SetParmVal` on an allow-listed set of parameter IDs.
  - Bound parameters and the tessellation/wake settings, since mesh density drives runtime.
  - Apply a timeout.

**Throughput.** Assumption: ~5–120 s per VSPAERO VLM case on one core, depending on tessellation and wake iterations. Panel mode costs more. Geometry-only checks run in ~0.1–1 s.

**License implications.** NOSA 1.3 has its own distribution terms (e.g. the LICENSE requests user registration and defines "Larger Work"). Read the full agreement before redistributing OpenVSP in a public container. Calling it as a separate tool for internal training is the low-risk pattern. *Not legal advice.*

**Sample task**
> *Prompt:* "Modify the baseline transport's wing (span, root chord, taper, sweep) so S_ref = 125 ± 1 m² and aspect ratio = 9.5 ± 0.1, and so that VSPAERO CL at α = 2°, M = 0.3 is within ±0.03 of 0.45."
>
> *Grading:*
> 1. Apply the allow-listed parameters.
> 2. Run CompGeom (areas).
> 3. Run VSPAERO VLM.
> 4. Score `r = 1[S]·1[AR]·exp(−|CL−0.45|/0.03)`.

---

### 2.5 python-control (required)

**Confirmed facts**
- **What it is:** a Python package for analysis and design of feedback control systems ([docs](https://python-control.readthedocs.io/en/latest/)):
  - state-space and frequency-domain I/O systems
  - Bode, Nyquist, and Nichols analysis
  - stability margins
  - root locus
  - eigenvalue placement, LQR, H2/H∞, and MPC
  - nonlinear I/O systems
  - time responses
- The [function reference](https://python-control.readthedocs.io/en/latest/functions.html) lists `find_operating_point`, `linearize`, `step_info`, `stability_margins`, `margin`, `lqr`, `place`, `bandwidth`, `damp`, `input_output_response`, and `forced_response`.
- **License:** BSD-3-Clause ([LICENSE](https://github.com/python-control/python-control/blob/main/LICENSE); PyPI `license_expression: BSD-3-Clause`).
- **Interfaces:** Python library (`pip install control`).
- **Maintenance:** latest PyPI release 0.10.2 (2025-07-05). Repo last pushed 2026-09-29.
- **Optional dependency Slycot:** GPL-2.0. Its README says it is primarily a python-control dependency and that it stays GPLv2 even though SLICOT ≥ 5.7 moved to BSD-3-Clause ([Slycot](https://github.com/python-control/Slycot)). Slycot 0.7.0 was released 2026-06-16.

**What it can check**
- Pole locations, damping, and natural frequency (`damp`).
- Gain and phase margins (`stability_margins`).
- Step metrics: overshoot, rise time, settling (`step_info`).
- Closed-loop bandwidth.
- LQR optimality: compare the model's K to `lqr` output.
- Controllability and observability.
- All of the linear HQ metrics in §3: CAP, bandwidth/phase delay, Neal–Smith, and dropback via `step_response`.

**Reward formulation**
- **What the model emits:** gains, or a controller as numeric matrices or transfer-function coefficients (JSON). Avoid arbitrary Python unless the sandbox is strong.
- **Scalar reward:**
  - Hard specs as a product of indicators, or margin-normalized soft scores, e.g. `min(1, PM/45°)·min(1, GM_dB/6)·1[closed-loop stable]`.
  - For HQ, use level-based rewards (§3.11).
- **Anti-hacking:**
  - Check stability of the **closed loop on hidden plant variants**: mass/CG/speed perturbations and actuator lag and delay. This stops overfitting to the nominal plant.
  - Reject non-finite or absurdly large gains, since actuator saturation is ignored in linear models.
  - Re-derive all metrics; never trust model-reported margins.

**Throughput.** Assumption: ~1–50 ms per check on one core for ≤ 20-state models. This makes it the cheapest high-signal verifier here.

**License implications.** BSD-3-Clause is permissive. If you install Slycot, you are importing a GPL-2.0 extension into the verifier process. That matters if you **distribute** the verifier, not for internal use. You can keep the verifier pure-BSD by not installing Slycot and avoiding the routines that require it. *Not legal advice.*

**Sample task**
> *Prompt:* "Given the short-period state-space model A, B (α, q; elevator), design pitch-rate and α feedback gains so the closed-loop short-period ζ ∈ [0.5, 0.8] and ω_n ∈ [2.5, 4] rad/s, with ≥ 45° phase margin at the elevator input including a 50 ms delay."
>
> *Grading:*
> 1. Close the loop with `ss` and `feedback`.
> 2. Run `damp` to check ζ and ω_n.
> 3. Compute `stability_margins` with a Padé delay.
> 4. Repeat on 5 hidden plant perturbations.
> 5. Score `r = mean over plants of Π indicators`.

---

### 2.6 Slycot and SciPy (numerical back-ends)

- **Slycot:** GPL-2.0 per [LICENSE.txt](https://github.com/python-control/Slycot/blob/main/LICENSE.txt), which says "version 2" with no "or later" in the header we read. PyPI's license expression also lists bundled BSD and GCC-exception components. v0.7.0. It provides Riccati, Lyapunov, and Sylvester solvers, H2/H∞ synthesis, and model reduction via SLICOT ([README](https://github.com/python-control/Slycot)).
- **SciPy:** BSD-3-Clause ([repo](https://github.com/scipy/scipy)). v1.18.1 (2026-08-21). Used for eigenvalues, `solve_ivp` time responses, `solve_continuous_are`, and optimization.
- **Role:** these are building blocks, not standalone verifiers. The HQ module (§3) should depend only on NumPy/SciPy and python-control so the core verifier stays permissively licensed.

---

### 2.7 NeuralFoil

**Confirmed facts**
- **What it is:** a physics-informed ML airfoil analysis tool trained on "nearly 8 million XFoil runs", in pure Python/NumPy ([GitHub](https://github.com/peterdsharpe/NeuralFoil)).
- **Outputs:** CL, CD, CM, top/bottom transition, `analysis_confidence`, and (for experts) boundary-layer and pressure information.
- **Speed and accuracy (as stated in the README, not reproduced here):** ~1.2–6.1 ms per run vs. ~73 ms for XFoil, and "typically accurate to within a few percent of XFoil".
- **Inputs:** coordinates, `.dat` files, or 18-parameter CST (Kulfan).
- **License:** MIT ([LICENSE.txt](https://github.com/peterdsharpe/NeuralFoil/blob/main/LICENSE.txt)).
- **Maintenance:** PyPI 0.3.3 (2026-07-11). Repo pushed 2026-10-07.

**What it can check.** Fast approximations of XFOIL-like coefficients. **It is a surrogate, not ground truth.**

**Reward formulation**
- **Shaping reward:** use NeuralFoil as a dense, cheap reward for airfoil tasks.
- **Low-confidence gating:** multiply the reward by `1[analysis_confidence ≥ c_min]`. This stops the policy from exploiting out-of-distribution regions where the surrogate is wrong but flattering.
- **Audit:** re-evaluate a random 2–5% of rollouts (and **all** top-k candidates) with XFOIL. If NeuralFoil-vs-XFOIL disagreement exceeds a threshold, replace the NeuralFoil reward with the XFOIL result and log the case as a blind-spot sample (§4.7).

**Throughput.** ~1–6 ms per run (README figure, model-size dependent). Batched evaluation is faster per case.

**License implications.** MIT; no copyleft concerns.

**Sample task.** Same as the XFOIL task in §2.3, but graded by NeuralFoil with confidence gating, plus an XFOIL audit of the final answer.

---

### 2.8 SU2 (high-fidelity audit only)

- **What it is:** open-source C++/Python PDE analysis and PDE-constrained optimization on unstructured meshes, with discrete adjoints, compressible and incompressible flow, and HPC support ([su2code.github.io](https://su2code.github.io/)).
- **License:** LGPL-2.1 per the site and [LICENSE.md](https://github.com/su2code/SU2).
- **Maintenance:** latest release v8.5.0 (2026-04-27). Repo pushed 2026-10-15.
- **Use:** too expensive for an RL inner loop. Assumption: CPU-hours per RANS case. Use it to (a) audit top candidates offline and (b) build fixed high-fidelity reference sets for "aero database" tasks, where the model must reproduce or interpolate stored values.
- **Reward:** normalized error vs. a precomputed SU2 table; no live SU2 calls during RL.
- **License note:** LGPL as a separate process is low-risk. *Not legal advice.*

### 2.9 AeroSandbox

- **What it is:** an MIT-licensed Python package for aircraft design optimization ([GitHub](https://github.com/peterdsharpe/AeroSandbox), [LICENSE.txt](https://github.com/peterdsharpe/AeroSandbox/blob/master/LICENSE.txt)). It includes:
  - an automatic-differentiation optimization stack
  - a VLM and airfoil tools, plus interfaces to AVL and XFoil
  - trajectory and stability analysis
  - propulsion and mass-estimation modules
- **Maintenance:** PyPI 4.2.10 (2026-07-05). Repo pushed 2026-10-07.
- **What it can check:** whole-aircraft feasibility (mass closure, lift = weight, stability margins) with fast in-process models.
- **Reward:** constraint satisfaction with penalties. It is a good "environment" for conceptual-design tasks, cross-checked against AVL.
- **Anti-hacking:** run model-proposed designs only through the verifier's own fixed problem definition; never execute model-written optimization code without a sandbox.
- **Throughput (assumption):** ~0.01–10 s per evaluation.
- **License:** MIT.

### 2.10 OpenAeroStruct and OpenMDAO

- **OpenAeroStruct:** lightweight aerostructural analysis and optimization on OpenMDAO. It couples a VLM with a 6-DoF-per-node beam model and supports multipoint runs, wingbox and composite structures, ground effect, and stability derivatives ([docs](https://mdolab-openaerostruct.readthedocs-hosted.com/en/latest/)).
  - **License:** Apache-2.0 per [LICENSE.md](https://github.com/mdolab/OpenAeroStruct). **Note:** the PyPI metadata says "BSD-3", which disagrees with the repo; treat the repo LICENSE as authoritative and flag this upstream.
  - **Version:** v2.12.0 (2025-10-06). Last push 2025-10-06, so activity is slower.
- **OpenMDAO:** Apache-2.0 ([LICENSE.txt](https://github.com/OpenMDAO/OpenMDAO)). v3.45.1 (2026-09-10).
- **What it can check:** wing CL/CD, structural failure indices, mass, and fuel burn proxies for a given planform and structural sizing.
- **Reward:** `Π 1[failure ≤ 0] · 1[CL = CL_req ± tol] · g(fuel burn or CD)`.
- **Throughput (assumption):** ~1–60 s per analysis.
- **License:** permissive.

### 2.11 AVL variants: OptVL

- **OptVL:** "A modified version of Mark Drela and Harold Youngren's famous AVL code with a direct Python interface and gradient based optimization support" ([PyPI](https://pypi.org/project/optvl/), [GitHub](https://github.com/joanibal/OptVL)).
  - **License:** the LICENSE file contains GPL v3 text plus a bundled-LAPACK BSD-3 notice, so treat it as **GPL-3.0**.
  - **Version:** PyPI 2.5.0 (2026-04-17). Repo pushed 2026-10-02.
- **Why use it:** in-process calls avoid subprocess and file overhead, and gradients enable verifier-side sanity checks such as finite-difference consistency.
- **Caveat:** results should be cross-checked against upstream AVL 3.52, since it is a fork.
- **License:** importing GPL-3.0 code into the verifier process makes a distributed verifier a GPL combined work. For internal training that is usually not a concern, but isolate it in its own worker process if you plan to publish the verifier. *Not legal advice.*
- **Do not confuse with** the unrelated PyPI package `pyavl`, which is public domain with a different upstream.

### 2.12 CasADi

- **What it is:** nonlinear optimization and algorithmic differentiation. It provides ODE/DAE integration (incl. SUNDIALS), NLP solvers via IPOPT and others, QP solvers, and optimal control, with Python, MATLAB, and C++ interfaces ([web.casadi.org](https://web.casadi.org/)).
- **License:** LGPL-3.0 per [LICENSE.txt](https://github.com/casadi/casadi); PyPI says "LGPLv3 or later".
- **Version:** 3.8.1 (2026-09-16).
- **What it can check:** feasibility of a proposed trim or trajectory by evaluating dynamics residuals and constraint violations. For optimality tasks, it measures the gap to a verifier-solved optimum.
- **Reward:** `r = 1[‖residual‖∞ ≤ tol] · exp(−gap/τ)`.
- **Throughput (assumption):** ~0.01–10 s.
- **License:** LGPL, import OK internally. *Not legal advice.*

### 2.13 ArduPilot SITL and PX4 SITL

- **ArduPilot SITL:** builds the autopilot as a native executable with built-in physics per vehicle type. It can instead use JSBSim or a JSON simulator interface, with FlightGear for visualization. Access is via MAVLink/MAVProxy, and multiple instances use `-I n` port offsets ([SITL docs](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html)).
  - **License:** GPL-3.0 ([repo](https://github.com/ArduPilot/ardupilot)).
  - **Version:** latest release Plane-4.7.2 (2026-10-14).
- **PX4 SITL:** Gazebo is recommended. **SIH** runs physics inside PX4, is headless by default, and runs in lockstep. `PX4_SIM_SPEED_FACTOR` sets faster-than-real-time for Gazebo, Gazebo Classic, and SIH ([PX4 simulation docs](https://docs.px4.io/main/en/simulation/)).
  - **License:** BSD-3-Clause ([repo](https://github.com/PX4/PX4-Autopilot)).
  - **Version:** v1.17.0 (2026-05-13).
- **What they check:** whether a proposed **autopilot parameter set** (e.g. TECS/L1 or rate-loop gains) flies a scripted mission within tolerances: track error, altitude hold, no failsafes.
- **Reward:** episode-level pass/fail plus normalized tracking-error score.
- **Anti-hacking:** clamp to parameter ranges and log any parameters outside the allow-list. Randomize wind and sensor noise with hidden seeds. Watchdog timeouts give reward 0.
- **Throughput (assumption):** roughly real time up to a few times faster (PX4 lockstep plus speed factor), i.e. **tens of seconds to minutes per episode**. Too slow for Phase 0.
- **License:** ArduPilot is GPL-3.0; run it as a separate process and don't distribute modified builds without source. PX4 is permissive. *Not legal advice.*

### 2.14 FlightGear

- **What it is:** a free, open-source flight simulator ([flightgear.org](https://www.flightgear.org/)).
- **License:** the site's [license page](https://www.flightgear.org/about/license/) says GNU GPL without a version. The official GitLab repository's `LICENSE.md` is GPL v2, and its `LICENSES/` directory includes `GPL-2.0-or-later` ([GitLab](https://gitlab.com/flightgear/flightgear)). Treat it as **GPL-2.0-or-later**.
- **Version:** 2024.1.7, dated 14 Aug 2026 on the site.
- **Role:** visualization and demos. It adds no verification beyond its FDMs (JSBSim is usable directly), so it is **not recommended as a reward source**.

### 2.15 OpenModelica

- **What it is:** an open-source Modelica environment: the compiler (`omc`), OMEdit, and OMSimulator for FMI/SSP co-simulation ([GitHub](https://github.com/OpenModelica/OpenModelica)). The official site was unreachable at fetch time.
- **License:** `OSMC-License.txt` states the program is provided under **AGPL version 3 or the OSMC Public License (OSMC-PL) version 1.8, at the recipient's choice**.
- **Versions:** latest release v1.27.1 (2026-09-08). The Python API `OMPython` is 4.1.0 on PyPI.
- **What it can check:** actuator, hydraulic, and electrical subsystem responses. Examples: does a proposed servo-loop design meet bandwidth and rate limits on a physical actuator model? It can also export FMUs for co-simulation with a JSBSim-based closed loop.
- **Reward:** pass/fail on simulated specs.
- **Throughput (assumption):** ~1–30 s including compilation. Cache compiled models and vary only parameters.
- **License note:** AGPL's network clause matters if you expose a modified OpenModelica **as a network service to third parties** (e.g. a hosted public verifier API). The OSMC-PL alternative has its own terms; read it. *Not legal advice.*

### 2.16 Z3 (SMT) for requirements logic

- **What it is:** a theorem prover from Microsoft Research. MIT license ([repo](https://github.com/Z3Prover/z3), LICENSE.txt). Python via `pip install z3-solver`.
- **Version:** latest release z3-5.3.1 (2026-10-15).
- **What it can check:** once requirements are formalized as linear-arithmetic and Boolean constraints over named variables (e.g. `0.35 ≤ zeta_sp ≤ 1.30`, `mode = CatA → tau_R ≤ 1.0`):
  - **consistency** (satisfiable?)
  - **conflict localization** (unsat core)
  - **entailment** (does derived requirement D follow from parents P1..Pn? i.e. `P ∧ ¬D` is unsat)
  - **coverage of a verification matrix**
- **Reward formulation:**
  - The model emits a formalization (in a restricted DSL that the verifier compiles to Z3, never raw Python) or answers like "requirements 7 and 12 conflict".
  - Reward is exact-match on sat/unsat, Jaccard overlap with the verifier's minimal unsat core, or entailment correctness.
  - Hidden gold formalizations prevent the policy from writing vacuous formulas such as `True`. Score formalization tasks by **semantic equivalence** to the gold formula (`F_model ↔ F_gold` valid), not by sat status alone.
- **Throughput (assumption):** ~1 ms–1 s for typical requirement sets. Set a solver timeout (`timeout` param); timeout = reward 0.
- **License:** MIT.

### 2.17 Requirements-trace checker (Doorstop / StrictDoc / sphinx-needs + custom checks)

- **Doorstop:** stores requirements as YAML files in version control, organized as a parent/child document tree. Running `doorstop` with no arguments runs integrity checks on the tree; it can also publish ([docs](https://doorstop.readthedocs.io/en/latest/)).
  - **License:** LGPL-3.0 ([LICENSE.md](https://github.com/doorstop-dev/doorstop)).
  - **Version:** v3.2 (2026-07-10).
- **StrictDoc:** documentation and requirements management in `.sdoc` files. Its docs include a technical note titled "DO-178C requirements tool requirements" ([docs](https://strictdoc.readthedocs.io/en/stable/)).
  - **License:** Apache-2.0 ([LICENSE](https://github.com/strictdoc-project/strictdoc)).
  - **Version:** 0.30.2 (2026-10-09).
- **sphinx-needs:** MIT ([repo](https://github.com/useblocks/sphinx-needs)). PyPI 9.0.0 (2026-10-14).
- **Custom checker (to build):** load the trace graph (system requirement → HQ requirement → design artifact → verification case) and compute:
  1. dangling or invalid links
  2. orphans (child without parent)
  3. coverage (every leaf requirement has ≥ 1 verification case)
  4. link-set **precision/recall vs. a hidden gold trace**
  5. optional Z3 consistency of the numeric constraints attached to nodes
- **Reward:** `r = F1(links_model, links_gold) · 1[no dangling] · 1[schema valid]`.
- **Anti-hacking:** linking everything to everything is punished by precision. Cap link count per item, and validate YAML/SDoc against a strict schema.
- **Throughput (assumption):** ~0.1–2 s per check.
- **License:** permissive or LGPL; all low-risk internally.

---

## 3. Computable handling-qualities criteria (MIL-STD-1797A / MIL-F-8785C lineage)

### 3.0 Source documents and status (as fetched)

- **MIL-F-8785C**, *Military Specification: Flying Qualities of Piloted Airplanes*, 5 Nov 1980, superseding MIL-F-8785B ([EverySpec page](https://everyspec.com/MIL-SPECS/MIL-SPECS-MIL-F/MIL-F-8785C_5295/); [PDF](https://everyspec.com/MIL-SPECS/MIL-SPECS-MIL-F/download.php?spec=MIL-F-8785C.028392.pdf)).
  - Numeric limits quoted below were read directly from the scanned tables in this PDF (pp. 11–28).
  - Classes run from I (small, light) to IV (high maneuverability). Flight Phase Categories A, B, and C and Levels 1–3 are defined in §1.
- **MIL-STD-1797A**, *Flying Qualities of Piloted Aircraft* (base document dated 30 Jan 1990). **Distribution status is disputed; treat it as restricted.**
  - The official DoD repository, [ASSIST QuickSearch](https://quicksearch.dla.mil/qsDocDetails.aspx?ident_number=70037), lists Revision A (1990) and Revision A Change Notice 1 (28 Jun 1995) as **Distribution D** (DoD and DoD contractors only), checked 2026-10-15. Only the one-page cancellation (Notice 2, 1997) and reinstatement (Notice 3, 2004) notices are Distribution A. Revision B (2006) is also Distribution D.
  - A third-party mirror copy of Notice 1 states that it changed the distribution statement to A. We do not rely on that, because it conflicts with ASSIST.
  - Policy until DLA or the preparing activity resolves it: do not ingest MIL-STD-1797A, do not link mirror copies, and take verifier limits from **MIL-F-8785C** (Distribution A). Section and figure numbers for MIL-STD-1797A below come from its contents pages and are given for orientation only, marked "(verify)". See [data-sources.md](data-sources.md).
- **NASA-CR-179654**, Peahl, Kolkailah & Sandlin, *A Survey of Handling Qualities Criteria and Their Applications to High Performance Aircraft*, Cal Poly / NASA grant NCC2-338 (1986) ([NTRS PDF](https://ntrs.nasa.gov/api/citations/19860021266/downloads/19860021266.pdf)). It explains the Neal–Smith, bandwidth, and equivalent-systems procedures with an F-104 example.
- **Hendarko**, *Development of a Handling Qualities Evaluation Toolbox on the Basis of Gibson Criteria*, ICAS 2002 paper 533 ([PDF](https://icas.org/ICAS_ARCHIVE/ICAS2002/PAPERS/533.PDF)). It explains Gibson's dropback, flight-path time delay, and phase-rate criteria.

**Model sources for all criteria.** The verifier obtains a linear model by trimming and linearizing. Options:
- JSBSim `FGLinearization.state_space()` (§2.1)
- AVL `MODE`→`S` A/B matrices (§2.2)
- python-control `find_operating_point` + `linearize` on a nonlinear I/O model (§2.5)

For augmented (high-order) aircraft, MIL-F-8785C and MIL-STD-1797A apply many modal limits to **equivalent** low-order parameters. That requires a low-order equivalent system (LOES) fit; NASA-CR-179654 has a section on equivalent systems. The LOES fit is itself a verifier step: mismatch above a threshold means "criterion not applicable" rather than a pass.

### 3.1 Short-period frequency, damping, and CAP vs. n/α

- **What is computed:**
  - From the longitudinal linear model (or LOES fit of θ/Fs and nz/Fs): ω_sp, ζ_sp, and n/α. The last is the steady normal-acceleration change per unit α, approximately (V/g)/T_θ2 at the trim speed.
  - **CAP** = ω_sp² / (n/α).
- **Limits (MIL-F-8785C, verified from the PDF):**
  - ω_sp vs. n/α must lie within Figures 1 (Cat A), 2 (Cat B), and 3 (Cat C) (§3.2.2.1.1).
  - Figure 1 (Cat A) draws diagonal boundaries labeled by ω_sp²/(n/α):
    - **Level 1: 0.28 to 3.6**
    - **Levels 2/3 lower: 0.16; Level 2 upper: 10.0**
    - plus lower floors on ω_sp at low n/α (read the floor values from the figure; verify)
  - Figures 2 and 3 were not transcribed here (verify).
  - Damping ζ_sp, Table IV (§3.2.2.1.2):

    | Level | Cat A & C min | Cat A & C max | Cat B min | Cat B max |
    |---|---|---|---|---|
    | 1 | 0.35 | 1.30 | 0.30 | 2.00 |
    | 2 | 0.25 | 2.00 | 0.20 | 2.00 |
    | 3 | 0.15* | — | 0.15* | — |

    \*May be reduced above 20,000 ft if approved by the procuring activity.
- **MIL-STD-1797A:** §4.2.1.2 "Short-term pitch response" (Appendix A p. 171). Appendix A includes Fig. 13 "Short-period dynamic requirements", Fig. 20 "Definition of CAP from frequency response asymptotes", and Fig. 24 (ω_sp·T_θ2 vs. ζ_sp). Limits per MIL-STD-1797A (verify).
- **Reward:** `level = worst(level_from_CAP_or_ω, level_from_ζ)`, then map to a scalar (§3.11).

### 3.2 Phugoid damping

- **What is computed:** the long-period (phugoid) mode ζ_p, or the time to double amplitude T₂ = ln2/|σ| if unstable, from the longitudinal linear model with controls fixed and free.
- **Limits (MIL-F-8785C §3.2.1.2, verified):**
  - **Level 1:** ζ_p ≥ 0.04
  - **Level 2:** ζ_p ≥ 0
  - **Level 3:** T₂ ≥ 55 s
- **MIL-STD-1797A:** §4.2.1.1 "Long-term pitch response" (Appendix A p. 167). Limits per the standard (verify).
- **Reward:** level classification. The mode must be identified robustly; see §4.4 for mode labeling.

### 3.3 Dutch roll frequency, damping, and ζ_d·ω_nd

- **What is computed:** ω_nd, ζ_d, and ζ_d·ω_nd from the lateral-directional model (β, p, r, φ), plus |φ/β|_d from the Dutch roll eigenvector. That ratio is needed for the correction below.
- **Limits (MIL-F-8785C Table VI, §3.3.1.1, verified).** Minimums:

  | Level | Category | Class | min ζ_d | min ζ_d·ω_nd (rad/s) | min ω_nd (rad/s) |
  |---|---|---|---|---|---|
  | 1 | A (CO and GA) | IV | 0.4 | — | 1.0 |
  | 1 | A | I, IV | 0.19 | 0.35 | 1.0 |
  | 1 | A | II, III | 0.19 | 0.35 | 0.4** |
  | 1 | B | All | 0.08 | 0.15 | 0.4** |
  | 1 | C | I, II-C, IV | 0.08 | 0.15 | 1.0 |
  | 1 | C | II-L, III | 0.08 | 0.10 | 0.4** |
  | 2 | All | All | 0.02 | 0.05 | 0.4** |
  | 3 | All | All | 0 | — | 0.4** |

  - The governing damping requirement is whichever yields the larger ζ_d; ζ_d = 0.7 is the maximum required for Class III.
  - \*\*Class III may be excepted from the ω_nd minimum, subject to approval.
  - When ω_nd²·|φ/β|_d > 20 (rad/s)², the minimum ζ_d·ω_nd increases by 0.014, 0.009, or 0.005 × (ω_nd²|φ/β|_d − 20) for Levels 1, 2, and 3 respectively.
- **MIL-STD-1797A:** §4.6.1.1 "Dynamic lateral-directional response"; Appendix A Table XL "Recommended minimum dutch roll frequency and damping" (p. 506). Limits per MIL-STD-1797A (verify).
- **Reward:** level classification across all three minimums, including the |φ/β| correction.

### 3.4 Roll-mode time constant

- **What is computed:** τ_R = −1/λ_roll from the lateral-directional model, or from a LOES fit of p/F_as.
- **Limits (MIL-F-8785C Table VII, §3.3.1.2, verified).** Maximum τ_R in seconds:

  | Category | Class | L1 | L2 | L3 |
  |---|---|---|---|---|
  | A | I, IV | 1.0 | 1.4 | 10 |
  | A | II, III | 1.4 | 3.0 | 10 |
  | B | All | 1.4 | 3.0 | 10 |
  | C | I, II-C, IV | 1.0 | 1.4 | 10 |
  | C | II-L, III | 1.4 | 3.0 | 10 |

- **MIL-STD-1797A:** §4.5.1.1 "Roll mode"; Appendix A Table XXIV "Recommended maximum roll-mode time constant" (p. 377). Verify.

### 3.5 Spiral stability (time to double)

- **What is computed:** the spiral eigenvalue λ_s. If λ_s > 0, T₂ = ln2/λ_s; a stable spiral passes. Optionally, confirm with a nonlinear time response: release from a ≤ 20° bank disturbance with controls free and measure the bank doubling time.
- **Limits (MIL-F-8785C Table VIII, §3.3.1.3, verified).** Minimum time to double:
  - **Cat A & C:** 12 s (L1), 8 s (L2), 4 s (L3)
  - **Cat B:** 20 s (L1), 8 s (L2), 4 s (L3)
- **Coupled roll-spiral (MIL-F-8785C §3.3.1.4, verified):** if a coupled roll-spiral oscillation exists (allowed only for Cat B/C), ζ_RS·ω_nRS must exceed 0.5 (L1), 0.3 (L2), or 0.15 (L3) rad/s.
- **MIL-STD-1797A:** §4.5.1.2 (Table XXV, p. 392) and §4.5.1.3 (Table XXVI). Verify.

### 3.6 Roll performance (time to bank)

- **What is computed:** time t to reach a bank-angle change after a full roll-control step. Simulate the nonlinear model (JSBSim) or the linear lateral model with control limits and rate limits.
- **Limits (MIL-F-8785C §3.3.4, verified):**
  - Table IXa covers Class I/II. Example: Class I Cat A Level 1 is 60° in 1.3 s.
  - Table IXb covers Class IV by speed range (VL/L/M/H). Example values for **Level 1 Cat A**: 30° in 1.1 s (VL and L), 90° in 1.3 s (M), and 50° in 1.1 s (H). **Cat B** is 90° in 1.7–2.0 s, and **Cat C** is 30° in 1.1 s.
  - Tables IXc–IXf cover Phase CO, GA, and Class III. See §3.3.4.1–3.3.4.2 for speed-range definitions and caveats.
- **MIL-STD-1797A:** §4.5.8.1 "Roll axis response to roll control inputs"; Appendix A Tables XXVIII–XXXIII. Verify.
- **Reward:** level classification, with **hidden** speed ranges and loadings.

### 3.7 Pitch attitude bandwidth and phase delay

- **What is computed** (procedure as explained in NASA-CR-179654 pp. 8–9, following Hoh et al.):
  - From the open-loop Bode plot of θ/F_es (or θ/δ_es), bandwidth ω_BW is the **lower** of two frequencies:
    - the 45° phase-margin frequency (phase = −135°)
    - the 6 dB gain-margin frequency (gain 6 dB above the gain at the −180° frequency ω_180)
  - Phase delay τ_p is estimated from the phase at 2·ω_180: τ_p = −(Φ(2ω_180) + 180°) / (57.3 · 2ω_180).
  - The survey notes difficulty with "shelf-like" amplitude plots, so flag those cases.
- **Limits:** level boundaries in the (ω_BW, τ_p) plane per **MIL-STD-1797A Appendix A Figure 57 "Bandwidth requirements"** (p. 225), with the definition in Figure 59. Related: Figure 41, "Time delay bands associated with flying qualities boundaries vs bandwidth". **Limits per MIL-STD-1797A Figure 57 (verify)**; not transcribed here.
- **Implementation:** python-control `frequency_response` on a dense log grid, then interpolate crossings. Pure delays use exact `exp(−sτ)` in the frequency domain, not Padé, for this metric.
- **Reward:** a level from the region the (ω_BW, τ_p) point falls in. For a smooth variant, use the signed distance to the Level 1 boundary.

### 3.8 Gibson dropback

- **What is computed** (definition as explained by Hendarko, ICAS 2002):
  1. Apply a pitch-controller step held for a period, then released.
  2. From the pitch-rate time response, take the **pitch-rate overshoot ratio** q_max/q_ss.
  3. Take **dropback** DB = the difference between pitch attitude when the stick is released and the steady attitude afterward. Positive = dropback, negative = overshoot.
  4. Normalize as DB/q_ss.
  - Negative dropback indicates sluggishness; large positive dropback indicates abrupt or bobbling tendencies.
  - Hendarko notes that Gibson later redefined the criterion so that only zero dropback is acceptable, and gives an acceptable q_max/q_ss range of 1.0–3.0. These are paraphrased from that paper.
- **Limits:** satisfactory/acceptable region boundaries per Gibson's dropback chart (reproduced as Hendarko Fig. 2; the original is Gibson's work, e.g. AGARD-CP-333, which MIL-STD-1797A Notice 1 cites as the source of boundaries in Appendix A Fig. 29). **Verify boundaries against Gibson's original.**
- **Model:** time simulation (linear or nonlinear) of the attitude response. Use `forced_response` with a step-then-release input.
- **Reward:** region classification (satisfactory / acceptable / unacceptable). Gibson's criterion is not defined in Levels, so map regions to scores explicitly and document the mapping.

### 3.9 Neal–Smith

- **What is computed** (procedure as explained in NASA-CR-179654 pp. 3–7, citing Neal & Smith AFFDL-TR-70-74):
  1. Close a pitch-attitude tracking loop with a pilot model: gain K_p, time delay τ, and first-order lead-lag compensation.
  2. Choose a minimum bandwidth BW_min, defined where closed-loop phase = −90°. The survey reports typical fighter values of 3.0–3.5 rad/s depending on task and n/α, and 3.0 rad/s suggested for landing.
  3. Set a maximum low-frequency droop of −3 dB.
  4. Set the pilot delay: Neal–Smith used 0.3 s up-and-away; a landing study used 0.2 s.
  5. Solve for the pilot compensation (lead/lag phase angle) that meets the bandwidth and droop requirements.
  6. Read off the closed-loop resonance peak |θ/θ_c|_max.
  7. Plot (pilot compensation angle, resonance peak) on the Neal–Smith parameter plane. Its rating boundaries are empirical, from flight test.
- **Limits:** Neal–Smith parameter-plane boundaries per AFFDL-TR-70-74 and MIL-STD-1797A Appendix A. Notice 1 lists Fig. 68 "Design criteria for pitch dynamics with the pilot in the loop" and Neal–Smith data comparisons in Figs. 22, 43, 44, 60, and 64. **Limits per the standard (verify).**
- **Implementation:** a small nonlinear solve over (K_p, T_lead, T_lag) using python-control closed-loop frequency responses. The survey shows the result depends strongly on the chosen BW_min, so **fix BW_min and τ per task in the verifier** and never let the model choose them.
- **Reward:** level from the parameter-plane region. If the solve fails, the reward is 0 and the case is logged.

### 3.10 Other computable criteria worth adding later

These are listed in MIL-STD-1797A Notice 1's contents; limits per the standard (verify):
- equivalent time delay (Appendix A Figs. 52–56)
- roll time delay (§4.5.1.5, Table XXVII)
- roll oscillations (§4.5.1.4)
- PIO tendency (§4.2.2)
- heading bandwidth for the wings-level turn mode (Tables XLI–XLII)
- flight-path stability (MIL-F-8785C §3.2.1.3 gives Level 1/2/3 limits of 0.06/0.15/0.24 deg/knot local slope at V₀min, read from the scanned PDF)

### 3.11 From criteria to a scalar reward

Steps:
1. Classify each applicable criterion c into a Level ℓ_c ∈ {1, 2, 3, 4}, where 4 means worse than Level 3.
2. Overall level: `ℓ = max_c ℓ_c`. This is weakest-link, matching how Levels are assigned per characteristic in the specs.
3. Scalar: `r = {1: 1.0, 2: 0.4, 3: 0.1, 4: 0.0}[ℓ]`. The weights are a design choice, not from the standard.
4. Optional shaping for early curriculum: `r += 0.1 · tanh(margin_to_L1_boundary / scale)`. Remove it later to avoid boundary-hugging.

Rules:
- **Applicability flags:** Class, Category, and Flight Phase are verifier inputs from the task, never model outputs.
- **LOES mismatch** above threshold means the equivalent-parameter criteria are not applicable. Fall back to bandwidth and Neal–Smith, which work directly on high-order responses.

---

## 4. Verifier engineering requirements

1. **Determinism and seeding**
   - Every verifier call takes `(tool_version_hash, input_canonical_json, seed)` and must return bit-identical outputs on rerun on the same container image.
   - Fix BLAS threads (`OMP_NUM_THREADS=1`) to avoid nondeterministic reductions.
   - Seed all disturbance/noise generators (SITL wind, sensor noise) from a verifier-side hidden seed derived from the task ID.
   - Record float environment flags.
2. **Version pinning and hashing**
   - Pin exact versions and record SHA-256 of every tool binary, wheel, and model file (e.g. AVL ≥ 3.52 because of the DERMATS axis fix; XFOIL 6.996; JSBSim 1.3.2; python-control 0.10.2; OpenVSP 3.55.0; NeuralFoil 0.3.3).
   - Store hashes with every reward record so any reward can be reproduced.
   - The same manifest (tool identity, version, hash, configuration, test results) is the kind of configuration evidence a tool-qualification effort under RTCA DO-330 would need. We did not review DO-330 text for this note, so confirm specific objectives against the document.
3. **Containerization**
   - One OCI image per verifier family, read-only root FS, no network, non-root user, and seccomp default.
   - CPU, memory, and process-count cgroups.
   - Model-supplied text enters only as data files; nothing model-written is executed except through a constrained DSL.
   - GPL tools live in their own images and run as separate processes.
4. **Timeouts**
   - Hard wall-clock limit per call, set to ~5–10× the expected runtime and enforced by the orchestrator (kill the process group).
   - Timeout → reward 0, plus a "timeout" status that is logged and monitored. A policy that learns to cause timeouts must not be rewarded.
5. **Numerical-failure handling**
   - These all give **reward 0, not a crash**, with a structured status code (`NONCONVERGED`, `NAN`, `INVALID_INPUT`, `TIMEOUT`, `TOOL_ERROR`):
     - XFOIL non-convergence
     - AVL trim failure
     - JSBSim trim failure
     - NaN/Inf in states or outputs
     - Z3 `unknown`
     - LOES fit failure
   - Track the failure-rate per status. A spike in `TOOL_ERROR` indicates a verifier bug, not a policy problem; quarantine those samples from training.
6. **Caching**
   - Content-addressed cache keyed by `hash(tool_hash, canonical_input, seed)`.
   - Canonicalize inputs (sorted keys, rounded floats to declared precision) so trivially different encodings hit the cache.
   - Never cache across tool-version changes.
   - Cache hit rate is also a useful diversity metric for policy exploration.
7. **Cross-checking tools to find blind spots**
   - **AVL vs. OpenVSP VSPAERO (VLM):** on a sample of model-proposed geometries, compare CL_α, Cm_α, and CDi. Large disagreement flags geometry the policy may be exploiting in one solver's discretization (e.g. extreme dihedral or tiny chords).
   - **XFOIL vs. NeuralFoil:** audit a random fraction plus all top-k. Disagreement plus low `analysis_confidence` indicates surrogate exploitation.
   - **JSBSim linearization vs. python-control `linearize`** on the same nonlinear model, or vs. AVL-derived A/B for the same geometry: catches sign and axis-convention errors.
   - **SU2 offline** on a handful of final designs: catches potential-flow blind spots (separation, transonic effects).
   - Treat cross-check disagreements as their own logged dataset, to be used for hard-case tasks or verifier fixes.
8. **Hidden references and anti-memorization**
   - Keep evaluation flight conditions, plant perturbations, and gold traces in a store that the policy's environment cannot read.
   - Rotate task instances.
9. **Input validation as a first-class stage**
   - Schema-validate, range-clamp (or reject), and geometry-sanity-check before any tool runs.
   - Recompute reference quantities (Sref, cref, bref, n/α) from the verified geometry or model rather than trusting supplied values.
10. **Observability**
    - Log per-call runtime, status, and the reward components (not only the final scalar) so reward-hacking patterns are visible, e.g. rewards concentrating on one constraint boundary.

---

## 5. Recommended Phase 0 verifier set

For a first RL experiment, the goal is the most reward signal per CPU-second with low hackability:

1. **python-control + an in-house HQ-criteria module (§3).** Millisecond checks (assumption) on linear models. It covers flight control design and HQ analysis directly, is BSD-3-Clause without Slycot, and is deterministic. Phase 0 criteria: the MIL-F-8785C limits transcribed above (short-period ζ, CAP lines, phugoid, Dutch roll, roll mode, spiral).
2. **AVL 3.52, as a subprocess.** About 0.05–0.5 s per case (assumption). It provides forces, derivatives, trim, eigenmodes, and A/B matrices for stability and control and aero-database tasks, and it feeds linear models into (1). GPL is acceptable as an unmodified separate process.
3. **NeuralFoil, with XFOIL audit sampling.** About 1–6 ms per case (README), MIT. Gate rewards by `analysis_confidence` and re-score top-k and a random slice with XFOIL to prevent surrogate exploitation.
4. **JSBSim 1.3.2 (Python module).** About 0.1–1 s per trim+linearize (assumption). It adds nonlinear trim feasibility and closed-loop time responses (roll performance, dropback) that linear checks miss. It is LGPL and importable.

**Cheap add-on, if requirements traceability is in Phase 0 scope:** Z3 plus a Doorstop-style trace checker. Both run in milliseconds to seconds (assumption) and are permissive or LGPL.

**Deferred:** OpenVSP/VSPAERO (cross-check only), OpenAeroStruct, CasADi, SU2 (offline audit), ArduPilot/PX4 SITL (too slow per episode), OpenModelica, and FlightGear (not a verifier).

---

## 6. Sources (all fetched 2026-10-15)

**Tools**
- JSBSim: <https://github.com/JSBSim-Team/jsbsim>, [COPYING](https://github.com/JSBSim-Team/jsbsim/blob/master/COPYING), [python/jsbsim.pyx.in](https://github.com/JSBSim-Team/jsbsim/blob/master/python/jsbsim.pyx.in), [src/initialization](https://github.com/JSBSim-Team/jsbsim/tree/master/src/initialization), [PyPI](https://pypi.org/project/jsbsim/)
- AVL: <https://web.mit.edu/drela/Public/web/avl/>, [avl_doc.txt](https://web.mit.edu/drela/Public/web/avl/avl_doc.txt), [version_notes.txt](https://web.mit.edu/drela/Public/web/avl/version_notes.txt), [gpl.txt](https://web.mit.edu/drela/Public/web/gpl.txt)
- XFOIL: <https://web.mit.edu/drela/Public/web/xfoil/>; xfoil-python: <https://github.com/DARcorporation/xfoil-python>
- OpenVSP: <https://openvsp.org/>, [license](https://openvsp.org/license.shtml), [LICENSE](https://github.com/OpenVSP/OpenVSP/blob/main/LICENSE), [API docs](https://openvsp.org/api_docs/latest/), [NASA VSPAERO Basics](https://www.nasa.gov/reference/openvsp-vspaero-basics)
- python-control: [docs](https://python-control.readthedocs.io/en/latest/), [function reference](https://python-control.readthedocs.io/en/latest/functions.html), [LICENSE](https://github.com/python-control/python-control/blob/main/LICENSE), [PyPI](https://pypi.org/project/control/)
- Slycot: <https://github.com/python-control/Slycot>; SciPy: <https://github.com/scipy/scipy>
- NeuralFoil: <https://github.com/peterdsharpe/NeuralFoil>; AeroSandbox: <https://github.com/peterdsharpe/AeroSandbox>
- SU2: <https://su2code.github.io/>, <https://github.com/su2code/SU2>
- OpenAeroStruct: [docs](https://mdolab-openaerostruct.readthedocs-hosted.com/en/latest/), <https://github.com/mdolab/OpenAeroStruct>; OpenMDAO: <https://github.com/OpenMDAO/OpenMDAO>
- OptVL: <https://github.com/joanibal/OptVL>, [PyPI](https://pypi.org/project/optvl/)
- CasADi: <https://web.casadi.org/>, <https://github.com/casadi/casadi>
- ArduPilot SITL: [docs](https://ardupilot.org/dev/docs/sitl-simulator-software-in-the-loop.html), <https://github.com/ArduPilot/ardupilot>
- PX4: [simulation docs](https://docs.px4.io/main/en/simulation/), <https://github.com/PX4/PX4-Autopilot>
- FlightGear: <https://www.flightgear.org/>, [license page](https://www.flightgear.org/about/license/), <https://gitlab.com/flightgear/flightgear>
- OpenModelica: <https://github.com/OpenModelica/OpenModelica> (OSMC-License.txt); [OMPython on PyPI](https://pypi.org/project/OMPython/)
- Z3: <https://github.com/Z3Prover/z3>
- Doorstop: [docs](https://doorstop.readthedocs.io/en/latest/), <https://github.com/doorstop-dev/doorstop>; StrictDoc: [docs](https://strictdoc.readthedocs.io/en/stable/), <https://github.com/strictdoc-project/strictdoc>; sphinx-needs: <https://github.com/useblocks/sphinx-needs>

**Handling qualities**
- MIL-F-8785C (5 Nov 1980): [EverySpec](https://everyspec.com/MIL-SPECS/MIL-SPECS-MIL-F/MIL-F-8785C_5295/), [PDF](https://everyspec.com/MIL-SPECS/MIL-SPECS-MIL-F/download.php?spec=MIL-F-8785C.028392.pdf)
- MIL-STD-1797 revision history and distribution codes: [ASSIST QuickSearch](https://quicksearch.dla.mil/qsDocDetails.aspx?ident_number=70037) (Rev A and Change Notice 1 are Distribution D; mirror copies are not linked)
- Peahl, Kolkailah, Sandlin, NASA-CR-179654 (1986): <https://ntrs.nasa.gov/api/citations/19860021266/downloads/19860021266.pdf>
- Hendarko, ICAS 2002 paper 533: <https://icas.org/ICAS_ARCHIVE/ICAS2002/PAPERS/533.PDF>

**Not fetched, so not relied on for numbers:** the MIL-STD-1797A 1990 base text and its Appendix A figures; MIL-HDBK-1797 (export-controlled per search results); Neal & Smith AFFDL-TR-70-74 itself; Gibson's original dropback publications; RTCA DO-330.
