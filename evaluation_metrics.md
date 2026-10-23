# Evaluation Metrics Specification — Group 42

**Project:** When Everyone Takes the Shortcut: Coordinating Personalized Navigation under Delayed Feedback  
**Status:** Draft v0.1 — for team agreement before implementing evaluation  
**Scope:** Metric definitions only (Task 1).  

> **Provenance convention:** **[Proposal]** means explicitly described in Sections 3.1, 3.3, or 4.4 of the project plan. **[Proposed operational definition]** is a reasonable implementation choice that the group must confirm. Mathematical expressions in the uploaded plain-text version of the proposal are missing in several places; the reconstructed expressions below therefore must be checked against the original formatted proposal before being treated as exact transcriptions.

## 1. Shared indexing and data

- `e ∈ E_mon`: monitored road links (not necessarily every link in the network).
- `u`: simulation time step; `t = 0, ..., T-1`: coordinator decision epoch.
- `W_t`: simulation steps from decision epoch `t` up to (but not including) the next decision epoch. **[Proposal]**
- `n_e(u)`: simulator-ground-truth occupancy of link `e` at simulation step `u` (vehicles on the link).
- `C_e`: fixed, predefined overload threshold for link `e`; **not** necessarily the road's physical maximum capacity. **[Proposal]**
- `i ∈ {1,...,N}`: vehicle/driver ID consistent across compared runs.
- `R_i,t`: feasible full origin-to-destination path recommendations for driver `i` at epoch `t`; paths inconsistent with the driven prefix have probability zero. **[Proposal]**
- `p_i,t(r)`: original fixed navigation policy's marginal probability of recommending full path `r` at epoch `t`.
- `q_t(r_1,...,r_N)`: coordinator's joint distribution on paths, conditional on **this epoch's** actual navigation inputs.

The *evaluation* reads the simulator's realized traffic state. The *coordinator* must not be given future or unobserved ground-truth traffic when making decisions. **[Proposal]**

## 2. Primary objective: overload score J₀.₉₀

### 2.1 Instantaneous overload

**[Proposal, expression operationalized]**

\[
O_u = \sum_{e\in E_{\mathrm{mon}}} \max\{0,\ n_e(u)-C_e\}.
\]

`O_u` is the *sum of positive occupancy excess* over monitored links at simulation step `u`. Each term is in vehicles. If no link exceeds its threshold, `O_u = 0`.

### 2.2 Window overload and discounted episode score

**[Proposed operational definition consistent with the proposal's window wording]**

\[
L_t = \sum_{u\in W_t} O_u, \qquad
J_\gamma = \sum_{t=0}^{T-1} \gamma^t L_t.
\]

The project uses `J_0.90` as its **primary** score, minimized over complete dispersal episodes. A *lower* `J_0.90` means less discount-weighted overload. `L_t` is accumulated over simulator steps, rather than treating only one snapshot per decision epoch. With occupancy measured in vehicles and one-second steps, its natural discrete unit is vehicle-steps (equivalently, vehicle-seconds when multiplied by `Δu = 1 s`).

**Decision required:** The available plaintext of the proposal describes a window score but omits its typeset equation. Confirm whether `L_t` is the raw **sum**, the time-step-size-weighted **integral** `Σ_u O_u Δu`, or a **window average**. Also confirm whether discounting resets per episode (recommended) and whether scores are comparable across different decision frequencies, sim-step sizes, and episode durations. All algorithms must use the same definition.

### 2.3 Numerical check

For three one-step windows with overloads `[8, 12, 5]`:

- `J_0.90 = 8 + 0.9·12 + 0.9²·5 = 22.85`.
- `J_0.99 = 8 + 0.99·12 + 0.99²·5 = 24.7805`.

This is a **unit test**, not experimental evidence. If a window contains multiple simulation steps, first aggregate those steps into `L_t`.

### 2.4 Minimum data required

| Field | Meaning | Suggested source |
|---|---|---|
| `episode_id`, `method`, `seed` | run identifiers | experiment controller |
| `decision_epoch_t`, `sim_step_u` | index and time | scheduler / SUMO |
| `link_id`, `occupancy_n` | true instantaneous link occupancy | SUMO / TraCI simulator ground truth |
| `threshold_C` | per-link overload threshold | scenario configuration |
| `decision_window_id` | map sim steps to decision windows | experiment controller |

**Do not** use coordinator-observed delayed occupancy in this metric. Compute scores from the simulator's true traffic state after the run.

## 3. Secondary objective: J₀.₉₉

**[Proposal]** Report `J_0.99` using *the same* windows, thresholds, traffic trajectories and score implementation as `J_0.90`, only changing `γ` from `0.90` to `0.99`:

\[
J_{0.99} = \sum_{t=0}^{T-1} (0.99)^t L_t.
\]

It puts relatively more weight on later windows than `J_0.90`. **Do not train or choose parameters using J₀.₉₉ unless the experiment protocol explicitly permits it**; the stated primary optimization is J₀.₉₀.

## 4. Per-driver extra travel time

**[Proposal]** Extra travel time is defined *relative to independent recommendations under matched simulation conditions*; the complete distribution and its upper tail must be reported (Sections 3.3 and 4.4).

**[Proposed operational definition]** For driver `i` present in matched runs:

\[
TT_i^{(m,s)} = t_{\mathrm{arrival},i}^{(m,s)} - t_{\mathrm{departure},i}^{(m,s)},
\]

\[
\Delta TT_i^{(m,s)} = TT_i^{(m,s)}- TT_i^{(\mathrm{ind},s)}.
\]

Here `m` is the evaluated method and `s` is the matched experiment seed/scenario. Positive `ΔTT_i` means that driver took longer under coordination, negative means the driver benefited. This is **realized travel time**, not an estimate based on the recommended route. The independent run's driver with the **same vehicle ID** should have the same origin, destination, departure schedule and underlying demand assumptions.

**[Proposed reporting]** Across drivers and paired replicates, report:

- Mean and median `ΔTT` (seconds).
- 90th/95th percentile and maximum positive `ΔTT` (tail burden; report the chosen percentile consistently).
- Proportion of drivers with `ΔTT > 0`, plus histogram / ECDF of driver-level `ΔTT`.

**Decision required:** Ensure stable IDs and consistent driver cohorts when compliance, route changes, and congestion affect arrivals. If some trips do not finish by the end of a run, predefine a completion/censoring rule instead of silently dropping them. Use paired scenarios and random seeds for fair comparison, but do not assume stochastic driver choices will be identical across policies.

**Data required:** `episode_id`, `method`, `seed`, `vehicle_id`, `depart_time`, `arrival_time`, `completed`, `origin`, `destination` (and if available, route, accepted recommendation).

## 5. Marginal preservation: hard constraint and numerical verification

### 5.1 Exact constraint

**[Proposal]** At every epoch `t`, conditional on that epoch's inputs, the coordinator may change *dependence across drivers* but must retain each driver's full-path distribution:

\[
\forall i,\ \forall r\in R_{i,t}:\quad
\sum_{\mathbf r_{-i}} q_t(r,\mathbf r_{-i}) = p_{i,t}(r).
\]

This is a **feasibility requirement**, **not** a congestion objective and **not** a constraint only on aggregate route counts. It must hold for every driver, every eligible full path, and each given set of decision inputs; preserving marginal probabilities only on average over different traffic states would not establish the required property.

### 5.2 Verification if analytical marginals are available

**[Proposed operational definition]** Denote the marginal induced by the joint sampler as `q_i,t(r)` and compute

\[
E_{\mathrm{TV},t} = \max_i\; \frac12 \sum_{r\in R_{i,t}} |q_{i,t}(r)-p_{i,t}(r)|.
\]

For an exactly feasible sampler, this is zero up to floating-point tolerance. Report the maximum over evaluated epochs and **explicitly specify the tolerance** (e.g., `1e-8` when numerically evaluating probabilities, subject to implementation precision). If exact `q_i,t` is intractable, provide a sampler-level argument/proof of marginal preservation as well as an empirical check.

### 5.3 Empirical conditional sampling check

**[Proposed diagnostic, not a proof]** At a **fixed** `t`, freeze `p_i,t`, coordinator parameter `θ_t`, and all input observations. Generate `K` repeated joint recommendation draws from the *same conditional sampler*. Define

\[
\hat p_{i,t}(r)=\frac{1}{K}\sum_{k=1}^{K}\mathbb{1}[r_i^{(k)}=r],
\quad
\widehat E_{\mathrm{TV},t}=\frac1N\sum_{i=1}^{N}\frac12 \sum_{r\in R_{i,t}}|\hat p_{i,t}(r)-p_{i,t}(r)|.
\]

The empirical error generally **will not equal zero**, even if the method exactly preserves marginals, due to finite sampling. Do **not** estimate this by pooling recommendation samples across different epochs, different observations or different changing `p_i,t` distributions. Record `K` and ideally an uncertainty interval / comparison to an independent sampler with the same `p_i,t`.

**Data required:** frozen `episode_id`, `epoch`, `vehicle_id`, `path_id`, source marginal probability `p`, repeated `sample_id` and sampled `recommended_path`, plus optional analytical sampler marginal `q_i`.

## 6. Reporting and interpretation

| Output | Role | Direction | Comparisons |
|---|---|---|---|
| `J_0.90` | **Primary traffic outcome** | Lower better | Independent, fixed, random, learned |
| `J_0.99` | Supporting traffic outcome | Lower better | Same runs/methods |
| `ΔTT_i` distribution and upper tail | Individual burden / fairness | Lower burden generally better; inspect both signs | Relative to matched independent baseline |
| Marginal-preservation constraint / diagnostic error | **Validity check** | Exact conditional marginals required; empirical error near sampling noise | Each coordinator and each epoch |

Suggested secondary presentation measure (**not defined by proposal**):

\[
\mathrm{Improvement}(m,\%) = 100\times \frac{\overline{J}_{0.90}^{\mathrm{ind}}-\overline{J}_{0.90}^{m}}{\overline{J}_{0.90}^{\mathrm{ind}}}.
\]

Compute means across matched replicates under one clearly identified scenario and compliance/delay setting, then report spread or paired confidence intervals. This ratio is undefined if the baseline mean is zero.

## 7. Questions to resolve with the team before freezing this spec

1. **Exact window metric:** Does the original formatted proposal use `Σ O_u`, `Σ O_u Δu`, or average `O_u` within `W_t`? What is `Δu`, decision interval, and episode termination criterion?
2. **Thresholds:** Which road links are monitored, how is each `C_e` determined and frozen across compared methods?
3. **Baseline matching:** Are vehicle IDs, O/D pairs, departure demand and compliance assumptions kept consistent across methods? How are incomplete trips handled?
4. **Marginal verification:** Can the algorithm implementation expose analytical conditional marginals? If not, can it support repeat sampling at frozen inputs to test individual full-path probabilities?
5. **Reporting:** Which tail percentile (e.g. 95th) and which replicates/uncertainty convention will be used? 

**Source:** Group 42 project proposal, §§3.1, 3.3, 4.4. This document is an evaluation implementation draft, not a claim that all formulas above appeared verbatim in the proposal.
