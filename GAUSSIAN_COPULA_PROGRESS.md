# Gaussian Copula Sampling — Progress Report

**Project:** When Everyone Takes the Shortcut: Coordinating Personalized Navigation under Delayed Feedback  
**Member:** Li Wenbo — Coordination Method and Stability  
**Date:** 2026-10-23  
**Status:** The core sampler prototype and initial correctness checks are complete, providing a foundation for subsequent CEM development.

## 1. Introduction and Implemented Features

**[Specified in the Plan]** The coordinator preserves each driver's recommendation distribution over complete origin–destination paths for the given navigation inputs, while adjusting only the dependence between drivers' recommendations. Gaussian Copula sampling implements this joint recommendation mechanism.

The sampling procedure is: correlated standard normal variables → standard normal CDF → uniform variables → path selection using each driver's cumulative probabilities. In theory, a valid correlation matrix with unit diagonal ensures that each transformed variable remains Uniform(0,1), preserving the corresponding marginal distribution. Finite samples exhibit random variation, and numerical calculations are subject to floating-point precision.

**[Proposed design: currently implemented]**

- Multiple drivers with different numbers of candidate paths, different probabilities, zero-probability paths, and deterministic choices.
- Independent recommendations, positive/negative dependence, and disjoint coordination groups; ungrouped drivers remain independent.
- An explicit NumPy random generator and validation of probabilities, correlation matrices, and group definitions.
- Marginals and a correlation matrix as inputs, with one path index per driver as output; the sampler neither accesses internal traffic state nor modifies the input probabilities.

Each group uses a common correlation parameter rho. For a group of m drivers, `-1/(m-1) < rho < 1` is required. This initial implementation accepts only strictly positive definite matrices and excludes singular endpoints. Here, rho describes the correlation of latent normal variables, not the correlation of discrete path indices. These parameterisation choices are implementation decisions rather than requirements fixed by the Plan.

## 2. Demonstrations and Correctness Checks

The following results come from actual local runs. Probability experiments use **100,000 draws per configuration**, with inputs held fixed within each batch; these draws do not represent successive traffic epochs. Path labels identify each driver's own complete paths and do not imply shared roads.

### Demo A: Two Drivers, Two Paths

Both drivers choose A/B with probabilities 50%/50%; seed=1110.

| rho | D1 chooses A | D2 chooses A | Both choose the same path label |
|---|---:|---:|---:|
| -0.9 | 49.87% | 49.99% | 14.20% |
| 0 | 49.87% | 49.94% | 49.73% |
| +0.9 | 49.87% | 50.08% | 85.44% |

Individual frequencies remain close to their targets while joint behaviour changes substantially. Automated tests also check the joint distribution against the analytical expression `P(AA) = 1/4 + arcsin(rho)/(2*pi)`.

### Demo B: Four Drivers with Heterogeneous Path Distributions

The input distributions are `[0.2,0.5,0.3]`, `[0.7,0.3]`, `[0.1,0.2,0.3,0.4]`, and `[0,0.6,0.4]`. The demo compares rho=0 with rho=-0.3; seed=2026.

- Each path's empirical frequency remains close to its target; the full demo output displays these comparisons individually.
- The frequency of jointly recommending D1-A and D2-A changes from **13.81%** to **10.66%**. Under independence, the theoretical probability is 14%.
- D4's zero-probability path is never recommended under either configuration.

### Demo C: Multiple Coordination Groups and an Independent Driver

All five drivers use 50%/50% distributions. D1/D2 have rho=-0.8, D3/D4 have rho=+0.8, and D5 remains independent; seed=3030.

| Joint event | Theoretical probability | Empirical frequency |
|---|---:|---:|
| D1-A and D2-A | 10.24% | 10.03% |
| D3-A and D4-A | 39.76% | 39.78% |
| D1-A and D3-A (across groups) | 25.00% | 24.90% |
| D1-A and D5-A (ungrouped driver) | 25.00% | 24.97% |

These results are consistent with the intended dependence within groups and independence across groups.

### Demos D/E: Changing Inputs, Boundary Cases, and Reproducibility

- **Changing inputs (seed=4040):** Three input snapshots are supplied manually while retaining rho=-0.6 for the two drivers. For example, when D1's distribution changes from `[0.2,0.5,0.3]` to `[0,0.4,0.6]`, the new empirical frequencies are `[0,0.4022,0.5978]`. The final deterministic inputs `[0,0,1]` and `[1,0]` produce exactly the corresponding choices. This checks that the sampler accepts updated marginals; it does not simulate navigation feedback.
- **Reproducibility (seed=5050):** Two separately initialised generators with identical inputs and seeds each produce 1,000 joint recommendations. Their outputs are identical element by element.

**Automated test result: 34 passed.** Tests cover analytical joint probabilities, heterogeneous marginals, zero-probability and single-path cases, unchanged inputs, reproducibility, and invalid inputs. Marginal frequency checks use a tolerance of six standard errors plus one-sample resolution. These statistical checks support implementation correctness but do not replace theoretical reasoning or a comprehensive stability analysis.
