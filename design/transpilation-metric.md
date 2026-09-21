# Optimization objective for transpilation

This records which quantity to optimize when evolving [the transpilation flow](transpilation-flow.md),
and the acceptance rule around it. It builds on [the benchmark and regression plan](transpilation-benchmarks.md)
("the plan") and uses its names: workloads B1–B3, metrics `D`/`D2`/`N2`, correctness
gates C1–C5, and the reproducibility settings of its section 5. Source baseline: Qiskit
`2.6.0.dev0`, commit `0131cbbcc`; line numbers below refer to that commit.

**Status:** proposed objective and policy. Of the measurements it needs, only B3's `D2`
trackers and the ASV timings are **existing**. `N2` companions, the multi-seed scorer,
held-out confirmation, and at-scale correctness checks are **proposed** (section 7).
The general-improvement tier, expanded fixtures and uncertainty across circuit instances
are also **proposed**. The measurements in section 4 cover only the focused panel;
they do not establish general improvement or the expanded panel's statistical power.
Thresholds are starting points to tune. Section 4 reports output-quality values measured
at the baseline commit; it does not establish a controlled timing baseline.

## 1. The objective

**Minimize native two-qubit depth `D2`. Hold two-qubit count `N2`, compile time, and
correctness as constraints.**

This is an acceptance policy for a `D2` optimization campaign. It does not require
unrelated correctness fixes or compiler-speed improvements to reduce `D2`; those use
the broader benchmark plan's review policy.

There are two qualification tiers. The **focused tier** supports a claim about the
three-circuit panel in section 5.1. A **general-improvement claim** requires section 5.2
and the plan's section 1.1: diverse circuit families, sizes, connectivity graphs, native
bases and optimization levels, with improvement on independent validation inputs.
For that tier, peak memory is an additional constraint. Here “general” describes the
declared static, gate-based population and the `D2` metric; it does not imply that every
input improves or that compilation time, fidelity or physical execution time improves.

It is one objective with hard constraints, not a weighted sum. That avoids inventing an
exchange rate between compile seconds and circuit quality. The intent is a shallower
suite with bounded regressions in gate count and compile time. The noise allowances,
per-case caps, and suite averaging below do **not** guarantee a Pareto improvement:
some cases can worsen, and a small aggregate regression can pass a guard.

These four rules are the minimum acceptance conditions. The general tier adds the
breadth, effect-size, validation and memory requirements in section 5.2:

| Signal | Rule | Compared against |
| --- | --- | --- |
| Correctness: C1–C5 plus the at-scale checks of section 6 | All pass. A failure blocks the change whatever it gains | — |
| `D2` | Primary suite score below 1 by more than seed noise: `ln(score) < -2 * SE` (section 4). Apply section 5's guards and, for general qualification, its stronger improvement tests | Current accepted best for improvement; frozen campaign baseline for guards and cumulative general-tier improvement |
| `N2` | Suite score not above 1 by more than seed noise: `ln(score) <= 2 * SE`. No single case above +5% | Frozen campaign baseline |
| Compile time | Suite ratio at most 1 plus the measured timing noise floor. No single case above +10% | Frozen campaign baseline |

The +5% and +10% caps adopt the investigation thresholds of the plan's section 5 as
automatic-acceptance limits. An intended trade outside these rules requires explicit
review (section 3). Per-case quality caps apply to `case_ratio`, not individual seeds;
also report worst-seed results. Freeze case weights and timing noise floors in advance.

The guards compare against the **frozen campaign baseline**, not the previously accepted
change. Otherwise every acceptance spends the noise allowance again: ten accepted changes
at "+1%, within noise" compound into a 10% regression that no single comparison flagged.

For a metric `m`, a case `c` (one circuit, target and optimization configuration),
fixed positive case weights `w_c` summing to one, and a seed panel `S`:

```text
case_ratio(c) = gmean over s in S of m_candidate(c, s)  /  gmean over s in S of m_baseline(c, s)
score         = exp(sum_c w_c * ln(case_ratio(c)))
```

The focused panel gives each of its three cases weight `1/3`, reducing this to the
ordinary geometric mean. The general panel uses the family-balanced weights in section
5.2 so a family with many variants cannot dominate. A score of 0.98 is a 2% reduction
in this weighted geometric summary, not a 2% reduction on every input.

Here `baseline` means the reference revision for the rule being evaluated. Compare
seed distributions; matching seed numbers do not guarantee matched search trajectories
after an algorithm changes. Pairing observations by seed is still valid for estimating
the mean difference, provided the uncertainty uses the observed differences rather
than assuming a beneficial correlation. Report every `case_ratio` before the aggregate.

Ratios and logarithms require positive metrics. Declare zero-baseline cases as separate
absolute-delta guards and require explicit review of any increase from zero. If a scored
candidate reaches zero, report that improvement separately and review it; do not silently
change the scored panel or take `ln(0)`. Never drop failures or missing results from the
denominator to improve a score.

## 2. Why `D2`, and why not the alternatives

| Candidate | Role | Reason |
| --- | --- | --- |
| Total depth `D` = `output.depth()` | Not an objective | It is what B2's trackers return ([`transpiler_levels.py`](../test/benchmarks/transpiler_levels.py), line 220). It counts a layer of virtual `rz` — zero duration in the plan's duration table — the same as a layer of two-qubit gates, so it can fall with no physical benefit. Over seeds 0–9 of section 4's setup, QFT's median `D` was 5,376.5 on the `cz` target and 8,037 on `ecr` while `D2` and `N2` were identical: this difference reflects single-qubit dressing |
| Two-qubit depth `D2` | **Objective** | A duration proxy when two-qubit gates dominate and their durations are comparable. B3's trackers already return it ([`utility_scale.py`](../test/benchmarks/utility_scale.py), line 96). It omits single-qubit, measurement, delay, and alignment costs; a reduction need not reduce makespan or decoherence |
| Two-qubit count `N2` | Constraint | Two-qubit errors often dominate gate error on the intended hardware. SABRE selects for SWAP count, a proxy for added `N2`; final translation and optimization can change that relationship |
| Scheduled makespan `T` | Later | Duration under a declared timing/resource model, not a fidelity measure. It needs frozen durations and scheduling constraints, and `GenericBackendV2` durations are synthetic. Adopt it when the change under test concerns scheduling |
| Fidelity estimate from target error rates | Trade arbiter only (section 3) | Rewards fitting synthetic calibration values, and is harder to interpret than a gate or layer count |

**SABRE does not select trials by final native two-qubit depth.** At the baseline commit:

- `SabreLayout` and `SabreSwap` keep the trial with the fewest swaps, ties broken by
  trial index: `min_by_key(|(index, result)| (result.swap_count(), *index))` in
  [`layout.rs`](../crates/transpiler/src/passes/sabre/layout.rs) (lines 203 and 309) and
  [`route.rs`](../crates/transpiler/src/passes/sabre/route.rs) (line 1053). Depth is never compared.
- The swap-scoring heuristic has `basic`, `lookahead`, and `decay` components and no depth
  term ([`heuristic.rs`](../crates/transpiler/src/passes/sabre/heuristic.rs), line 189).
  `decay` multiplies a swap's score by a factor that grows each time a qubit is swapped,
  which discourages serial swaps on the same qubits. That is the only pressure toward
  parallelism in this heuristic, and it is indirect. Another depth consideration is in
  `force_enable_closest_node`, the "release valve" that ignores the heuristics and forces
  progress along a shortest path (`route.rs`, line 865).
- Swaps scoring within `best_epsilon` of the best are chosen among at random (`route.rs`, line 962).
- The preset's standalone `SabreSwap` uses `basic` at level 0 and `decay` at levels 1–3,
  with 5 trials at levels 0–1 and 20 at levels 2–3
  ([`builtin_plugins.py`](../qiskit/transpiler/preset_passmanagers/builtin_plugins.py), from line 387).
  `SabreLayout` can perform the routing instead; its default level-2 configuration
  uses 20 layout trials, 20 swap trials, and two layout iterations (from line 780).
- The optimization stage does inspect **total** depth: levels 1–2 use depth/size fixed
  points, and level 3 uses a depth/size minimum-point check (`builtin_plugins.py`,
  from line 480). Thus depth is not absent from the pipeline, but these checks do not
  select SABRE trials by final `D2`.

Section 4 shows the consequence: compilations with nearly the same gate count differ
widely in depth. This motivates testing a `D2`-aware selection rule; it does not prove
that such a rule will improve the complete pipeline.

## 3. Why the `N2` guard, and how to judge a trade

`D2` alone can be bought with gates. Routing that spreads more SWAPs in parallel lowers
depth while adding two-qubit gates, each of which costs fidelity. The guard makes that
purchase visible.

Do not automatically accept a candidate outside the guards that lowers `D2` and raises
`N2`. Retain it for the explicit tradeoff review in the plan's section 5. One optional
review ranking is:

```text
trade_score = sqrt(N2_score * D2_score)      # below 1 is better
```

For this ranking, compute both scores against the same frozen campaign baseline on
the same cases, target, and seeds. Do not combine the two different references used
by the automatic acceptance rules.

Model negative log fidelity as `-ln F ≈ a*N2 + b*D2`, where `a` is the error per two-qubit gate
and `b` is the idle error accumulated per two-qubit layer across the active qubits. To
first order its relative change is `w*ΔN2/N2 + (1-w)*ΔD2/D2`, where `w` is the share of
`-ln F` due to gates. Choosing `w = 1/2` assumes equal contributions; it is a policy
choice, not a hardware-independent prior. To first order this gives equal weighting
of the log changes in `trade_score`; it does not derive that geometric formula as an
exact fidelity model for finite changes. `a`, `b`, and `w` depend on hardware and width.
Inspect per-case trades and plausible weights; the ranking is not the loop's objective.

## 4. Seed noise, measured at the baseline

SABRE is randomized, so `D2` depends on `seed_transpiler`, and any change that draws
random numbers differently reshuffles the result at a fixed seed. B2 and B3 each use one
seed (`0`, and `1234567845` at `utility_scale.py` line 41). A loop scored on one seed
harvests this noise.

Measured at `0131cbbcc` with B3's setup (level-2 preset, `GenericBackendV2` on
`CouplingMap.from_heavy_hex(9)`, backend seed `12345678942`), `cz` target, varying only
`seed_transpiler` over 0–49. These are output-quality values, deterministic for a given
checkout, configuration, and seed. The three numeric-circuit rows were reproduced in
the plan's serial environment during review. Re-measure on the benchmark runner and
retain raw per-seed observations before relying on the derived thresholds.

| Workload | `D2` min / median / max | `N2` min / median / max | sd of `ln D2` | sd of `ln N2` | Correlation of the log metrics | Linear-fit residual sd of `ln D2` |
| --- | --- | --- | --- | --- | --- | --- |
| `qft_N100` | 1,610 / 1,843 / 2,144 | 8,632 / 9,522.5 / 10,088 | 6.8% | 3.0% | 0.78 | 4.2% |
| `square_heisenberg_N100` | 345 / 400.5 / 498 | 1,671 / 1,822.5 / 1,974 | 8.7% | 3.9% | 0.55 | 7.3% |
| `qaoa_barabasi_albert_N100_3reps` | 1,392 / 1,528 / 1,710 | 8,272 / 8,626 / 8,877 | 4.2% | 1.4% | −0.12 | 4.2% |
| `efficient_su2(100, reps=3, "circular")` | 300 on every seed | 300 on every seed | 0 | 0 | — | — |

Standard deviations use `ddof=1`. The last column is
`sd(ln D2) * sqrt(1 - correlation**2)`, the residual spread after a linear fit of log
depth on log count. It is not a measured conditional spread at an identical `N2`, nor
proof that the same-count compilations differ by that amount.

Reading it:

- **`D2` varies two to three times more than `N2`**, with 4–7% residual spread after
  linear adjustment for log count. These output metrics motivate investigating depth;
  they establish neither semantic correctness nor the gain from a depth-aware change.
- **One seed is unreliable for small differences.** With independent search outcomes
  and equal baseline/candidate variances, a single-seed log-`D2` comparison has a
  standard deviation of about 6–12% (`sqrt(2)` times the fourth column).
- **Ten seeds can miss small gains.** Assuming independent cases and revisions with
  unchanged variance, `SE = sqrt(sum_c 2 * sd_c**2 / |S|) / number_of_cases` gives about
  1.8% for ten seeds and 0.6% for 100 seeds. These are planning estimates, not measured
  acceptance thresholds for the expanded panel. Use seeds 0–99 for the quality score,
  superseding the plan's ten-seed diagnostic panel, and compute uncertainty from the
  candidate and reference observations using the rule below. Measure the actual cost
  on the selected runner.
- **The `cx`, `cz`, and `ecr` targets are not independent cases.** Over seeds 0–9, `D2` and
  `N2` were identical on `cz` and `ecr` for QFT and Heisenberg. `cx` was identical for
  Heisenberg and within 2% for QFT, tracking seed by seed. These observations do not
  establish independence or native-basis invariance: basis-dependent preprocessing can
  change the interaction DAG reaching routing. The focused tier scores one fixed target
  (`cz`) and uses the others as guards. The general tier includes balanced basis strata,
  keeping their results paired within each input instance; they are coverage, not extra
  independent circuit samples.
- **This SU2 instance is a focused-panel canary.** No SWAPs are needed (`N2` equals the input's
  300 CX) and the emitted circular CX ladder is serial (`D2 = N2`). The observed panel
  has no seed variation; any change needs an explanation. This is not a proof that no
  equivalent shallower circuit exists or that a future candidate has zero variance.
  G7 in the general panel adds other ansatz instances and keeps this canary's role fixed.

For each metric, using the frozen weights and the same seed panel on both revisions,
calculate (for a marginal summary, restrict cases and renormalize their weights):

```text
x_r(s)  = sum_c w_c * ln(m_r(c, s))
delta_s = x_candidate(s) - x_reference(s)
ln(score) = mean_s(delta_s)
SE        = sample_sd(delta_s) / sqrt(number_of_seeds)
```

This retains covariance between workloads and between revisions at a shared seed;
it does not require the seed to produce the same search trajectory. For a per-case
guard, omit the mean over cases. If using independent seed panels instead, use
`SE = sqrt(var(x_candidate)/n_candidate + var(x_reference)/n_reference)`. Do not
substitute baseline-only variances when a candidate changes the distribution.

The `2 * SE` rules are approximate screening tolerances, not proof of non-regression
or a campaign-wide confidence guarantee. Repeated candidate selection, heavy tails,
and reuse of the tuning panel affect inference. Fresh confirmation seeds reduce
overfitting to that panel; repeated confirmation attempts still require a prespecified
multiple-testing or sequential-testing policy if a formal false-acceptance rate is claimed.
This `SE` conditions on the selected circuit inputs. Repeating one circuit on many
seeds does not estimate generalization across inputs. Section 5.2 adds an uncertainty
estimate across instance groups and requires fresh circuits as well as fresh seeds.

## 5. Scoring panel and protocol

### 5.1. Focused panel: iteration and a limited claim

Use this smaller subset while developing a candidate; its pass result alone is not
general qualification. The held-out roles in this table apply to the focused tier
only; section 5.2 assigns broader tuning and validation roles by instance.

| Role | Cases | Use |
| --- | --- | --- |
| Scored | `qft_N100`, `square_heisenberg_N100`, `qaoa_barabasi_albert_N100_3reps` with B3's setup on `cz`; seeds 0–99 | Primary `D2` score and `N2` guard on every candidate; equal circuit weights |
| Other native bases | The same circuits and seeds on `cx` and `ecr` | Per-target guards below; these do not increase the primary score's sample size |
| Confirmation | The scored circuits on all three targets with a fresh block of 100 seeds per decision (100–199, then 200–299, …) | Re-check quality rules against both reference revisions on that same block. Retire the block after any decision, including rejection |
| Held-out circuits | QUEKO (`queko.py`), frozen QV, `hwb12` and the BV circuits in `utility_scale.py` | Guards only, at acceptance: for each case, `ln(case_ratio) <= 2 * SE_case` and ratio at most 1.05 for both `D2` and `N2`, against the frozen campaign baseline. They need not improve |
| Canaries | `efficient_su2(100, …)` (observed `D2 = N2 = 300`); B2's "large QASM" at levels 2–3 (observed `D2 = N2 = 3`) | Any departure from the observed constant values needs an explanation and fresh validation |
| Timing | B1–B3 plus existing `utility_scale.time_qaoa` for all three bases | Every scored workload has timing coverage. B1 and the long two-qubit QASM remain useful overhead/optimization cases even with little observed quality headroom |

On each native target separately, require `ln(D2_score) <= 2 * SE` and no `D2`
case ratio above 1.05 against the frozen campaign baseline. Apply section 1's `N2`
guard separately on each target too. Thus a primary-target improvement cannot hide a
material regression on another basis or workload. Keep the strict `D2` improvement
test against the current accepted best on `cz`.

For timing, use the geometric mean of positive per-case time ratios over the declared
timing panel, with equal weights per parameterized timing case; exclude depth trackers.
Check both the aggregate noise allowance and every +10% cap. Since ASV uses fixed seeds,
also measure a fixed multi-seed timing companion on the scored workloads when changing
randomized search; keep manager construction outside `pm.run` timings consistently.
Use arithmetic mean elapsed time per circuit over the seed panel for that companion's
per-case time estimate and apply the same timing guards to it as a separate panel.
Fixed-seed timings alone do not constrain expected cost over the quality panel.

QV inputs are not reproducible until the fixture is fixed (the plan's section 7.3). QUEKO's
known-optimal depth is an absolute yardstick — distance from optimal, not just from the
baseline — but only once basis, allowed rewrites, and depth model match its reference
(the plan's section 3). Until that match is established, QUEKO remains usable as a
fixed regression fixture without an optimality claim. Freeze the exact held-out files,
QV matrices, targets, levels, seed panels, and weights before a campaign starts;
repeatedly inspected held-out circuits eventually become tuning cases and need refresh.

### 5.2. General-improvement qualification

**Required workloads: all eight families below.** Each family must contribute scored
cases to both the tuning panel and an independent validation panel. They are required
coverage for this policy, not optional extensions or only regression guards. The three
circuits in section 5.1 remain useful for iteration, but cannot qualify a general result.

| ID | Required workload family | Existing inputs to reuse as tuning anchors | Why it is included |
| --- | --- | --- | --- |
| G1 | Quantum Fourier transform (QFT) | `qft_N100` in `utility_scale.py`; `time_qft_16` in `transpiler_qualitative.py` | Structured, nonlocal interactions at different widths |
| G2 | Hamiltonian simulation | `square_heisenberg_N100` in `utility_scale.py` | Lattice interactions and repeated evolution layers |
| G3 | QAOA | `qaoa_barabasi_albert_N100_3reps`, used by `time_qaoa` in `utility_scale.py` | Graph-dependent interactions and repeated optimization layers |
| G4 | Quantum volume (QV) | 14×14 and 50×20 cases in `transpiler_levels.py`; 50×50 case in `utility_scale.py` | Random dense interactions and matrix-gate synthesis |
| G5 | Reversible logic | `hwb12` in `utility_scale.py`; `4gt10`, `4mod5`, `mod8` and `cnt3` fixtures in `transpiler_qualitative.py` | Boolean/arithmetic structures and their synthesis and simplification paths |
| G6 | Bernstein–Vazirani (BV) and BV-like circuits | `time_bv_100` and `time_bvlike` inputs in `utility_scale.py` | Algorithm-specific interaction patterns and simplification opportunities |
| G7 | Variational ansatz circuits | 89- and 100-qubit `efficient_su2` inputs in `utility_scale.py` | Repeated entanglement patterns and symbolic versus bound parameters |
| G8 | Routing challenge circuits | BNTF, BSS and BIGD fixtures in `queko.py` | Additional routing structures with fixed circuit/target pairs |

Sources: [`utility_scale.py`](../test/benchmarks/utility_scale.py),
[`transpiler_levels.py`](../test/benchmarks/transpiler_levels.py),
[`transpiler_qualitative.py`](../test/benchmarks/transpiler_qualitative.py), and
[`queko.py`](../test/benchmarks/queko.py). The named inputs already exist; the expanded
panel below is **proposed and requires additional fixtures and measurement code**.
The existing 100-qubit SU2 canary retains its guard role; other G7 instances supply
scored coverage. B2, qualitative and QUEKO total-depth trackers need `D2` companions.

**Required coverage within those families:**

| Dimension | Minimum required by this policy |
| --- | --- |
| Circuit size | Small: 4–16, medium: 17–64, large: 65–100 logical qubits, wherever the family supports the band; all three bands represented in each panel |
| Distinct inputs | At least three independent tuning input groups and three independent validation groups per supported family/size cell; new transpiler seeds do not count as new inputs |
| Connectivity | Heavy-hex, line and 2D grid, plus an all-to-all control; each family on at least two sparse topology classes unless a fixture contract fixes its target |
| Native gates and target details | `cx`, `cz` and `ecr` variants, a supported asymmetric directed target, and both fully occupied targets and targets with spare qubits |
| Optimization levels | 0, 1, 2 and 3 for every supported input/target pair |
| Transpiler seeds | 0–99 for tuning; a fresh 100-seed block for qualification, on both tuning and independent validation inputs |

Add widths and structural variants of the existing anchors: lattice sizes and evolution
depths for G2; graph instances, graph structures and repetitions for G3; frozen random
matrices and depths for G4; different Boolean/arithmetic circuits for G5; widths and
interaction patterns for G6; entanglement patterns, repetitions and parameter variants
for G7; independent circuit/target pairs for G8. Freeze QFT conventions for G1. Reserve
previously unused instances from **every** family for validation; the listed tuning
anchors cannot also serve as independent validation inputs.

The full fixture rules are in
[the plan, section 1.1](transpilation-benchmarks.md#11-general-improvement-panel--proposed).
Freeze exact case IDs, input/target hashes, supported cells, exclusions, splits and
weights in the campaign manifest before tuning. Both panels must contain all eight
families, all size bands and all required topology classes. Record unsupported cells
with reasons in advance; missing required coverage blocks qualification. Do not claim
this tier until the expanded fixtures and harness exist. The acceptance rules below
require improvement in at least four families and in the aggregate on both panels;
running the required workloads alone is not sufficient.

**Weighting.** Give each of the eight families weight `1/8`. Within a family, divide
weight equally among supported size bands, then topology classes, native bases,
optimization levels, independent input groups and finally their declared variants,
in that order. Divide only among children present in the frozen manifest. Persist each
resulting `w_c`; sum them to one. Compute tuning and validation scores separately with
the same family and stratum policy. Unequal instance counts must not change family
weights, and adding many QAOA graphs must not outweigh the other seven families.
Zero-baseline guards are outside the logarithmic score: freeze their roles and remaining
weights before tuning. If a required family has no positive scored cases, qualification
is incomplete until the panel is redesigned; never silently redistribute its weight.

Use exactly these weights for `D2`, `N2` and the general workload timing panels.
Within each marginal summary (family, size band, topology, basis or level), normalize
the included case weights to sum to one. Report its coverage alongside its score;
marginal differences are not causal estimates because supported case mixes can differ.

**Uncertainty across inputs.** In addition to section 4's seed `SE`, estimate the
spread of the weighted mean log-ratio across independent input groups. Use a frozen,
paired cluster-bootstrap procedure: resample instance-group IDs with replacement
within each family/size stratum, carrying all their target, basis, level and related
parameter variants together. Resample transpiler-seed IDs as whole shared vectors
across the selected cases, preserving candidate/reference pairing and cross-case
covariance. Recompute the fixed-stratum weighted score for each replicate. Keep family
and configuration weights fixed; resampling changes instance multiplicities, not the
intended workload mix. Preserve the manifest's supported configuration coverage in
every replicate; the harness must validate that instance groups within a resampling
stratum have the same configuration support, or predeclare finer compatible strata.
Require at least three independent scored groups per resampling stratum in each split;
add inputs before freezing the campaign if splitting would create singleton strata.
If an instance group spans size bands, resample it as one block with its cross-band
variants, using a predeclared compatible-support stratum rather than breaking the group.

Use 10,000 replicates and record the bootstrap RNG seed and the 95th percentile of
the bootstrapped log-score as `U_instance`. This is a proposed approximate one-sided
upper bound for the declared instance population. Few independent inputs, systematic
fixture bias and repeated candidate selection can make it unreliable; report group
counts, and do not interpret basis variants, levels or 100 search seeds as 100 new
circuits. It supports no inference to omitted families. Formal confidence claims
require a prespecified multiplicity/sequential policy and a justified sampling model.

**Acceptance.** Freeze a minimum practical reduction `epsilon` before tuning; the
initial proposed default is 1%. Qualification requires all of the following on both
the broad tuning panel and the independent validation panel. Evaluate every comparison
against both the current accepted best and the frozen campaign baseline unless a
guard explicitly specifies only the frozen baseline:

1. **Meaningful aggregate improvement:** `ln(D2_score) + 2 * SE < ln(1 - epsilon)`
   and `U_instance < ln(1 - epsilon)`. Require both seed and instance checks; a small
   seed error bar alone is insufficient. The validation panel must improve, not merely
   avoid regression. Report each reference comparison separately.
2. **Breadth:** at least half of the eight families must satisfy
   `ln(D2_family_score) + 2 * SE_family < 0`. After removing each family in turn and
   renormalizing the remaining weights, every remaining aggregate must still have
   `D2_score < 1`. These are coverage/concentration screens, not eight independent
   significance claims. They prevent one large gain from supplying the entire result;
   no requirement says every individual circuit must improve.
3. **Regression guards:** against the frozen campaign baseline, each family, size-band,
   topology, native-basis and level summary must satisfy `ln(score) <= 2 * SE` for both
   `D2` and `N2`, as must the overall `N2` score. No individual positive case ratio may
   exceed 1.05 for either metric. Report worst-seed values, but apply the cap to the
   seed-aggregated case ratio. Zero-baseline cases use absolute-delta guards with no
   increase for automatic acceptance. Do not hide them or failed cases in an aggregate.
4. **Cost:** time every scored case in separate end-to-end and reusable-manager
   panels, including a multi-seed companion when changing randomized search. Use the
   arithmetic mean elapsed time across seeds for each case's companion estimate.
   Each timing panel and each of its family summaries must be at most 1 plus its
   premeasured baseline-versus-baseline relative noise allowance, with no case above
   1.10. Apply section 5.1's timing guards separately to B1–B3 overhead/regression
   timings. Record fresh-process peak RSS on the plan's frozen memory panel: its
   aggregate (equal family weights, renormalizing the selected weights within each family)
   must be at most 1 plus its measured relative noise allowance,
   and no case may exceed 1.10. Use repeated RSS measurements and their median per
   case; record the absolute noise floors and investigate noisy breaches before a
   decision. All cost comparisons use the frozen baseline. Gains in `D2` cannot
   compensate for failing a cost guard.
5. **Correctness and completeness:** C1–C5 where applicable, section 6's checks on
   scored outputs, and affected repository tests pass; required coverage and measurements
   are complete, with zero unexpected crashes/timeouts. An unsupported configuration
   is an exclusion only when declared before tuning, not after a candidate fails.

These are proposed acceptance thresholds, not measured properties of the expanded
suite. Establish baseline runtime, noise and independent-instance counts before using
them. The protocol can establish a broad empirical `D2` result within its declared
population and constraints, not a universal or Pareto improvement. Report a candidate
that passes only section 5.1 as a focused-panel improvement. Any reviewed exception to
the general criteria must be named as a trade or a narrower result, not a general-tier
pass. Use separate qualified claims for compiler-speed, scheduling or other objectives.

### 5.3. Execution protocol for both tiers

1. **Freeze** the commit, fixtures, targets, and search budget as in the plan's section 5,
   items 1–3. Declare the qualification tier; for general qualification also freeze
   the section 1.1 manifest, input-group splits, weights, effect threshold, reserve
   validation panels and uncertainty procedure. Trial counts are part of the
   configuration. Raising them converts compile time into quality, so report that as
   a configuration change, not an algorithmic gain.
2. **Cache baseline measurements by configuration and seed block.** Start with quality
   on seeds 0–99 and the selected tier's fixtures. Keep validation results out of candidate
   selection until qualification. Time the panel above in the plan's serial
   environment. Derive aggregate and per-case timing noise floors from repeated
   baseline-versus-baseline measurements; a single comparison cannot estimate their spread.
   Do the same for the general tier's memory panel. Cache identities include the harness
   revision, manifest, input/target hashes, options, worker settings and seed block.
3. **Inner loop, per candidate.** Correctness gates first, then the quality score and
   guards on seeds 0–99. The focused subset may screen candidates cheaply; a general-tier
   candidate must then pass the full broad tuning panel before validation. Time only
   the candidates that pass quality: timing is the slow, noise-sensitive step. Use the
   same recorded worker settings for both revisions.
   A fixed trial budget is necessary for reproducibility, but does not establish thread
   invariance for every pass or future candidate. Validate any alternate worker setting
   before mixing its quality results with the reference configuration.
4. **Acceptance.** Freeze the candidate, then evaluate it, the current accepted best,
   and the frozen campaign baseline on the fresh confirmation block. Cache/reuse results
   only for the same revision, options, and seeds; never compare that block with baseline
   seeds 0–99. Check held-out circuits and canaries. For general qualification, also
   evaluate independent validation instances from every family on this fresh seed block
   and enforce all section 5.2 rules; rerun the broad tuning panel's quality rules on the
   fresh block too. Retire exposed validation inputs after every decision, including
   rejection, and use the next predeclared reserve for the next attempt. Retire seed
   blocks as well. Accept only if all quality, cost, coverage and correctness rules
   hold. Record failed attempts as well as successful ones. Do not claim a formal
   campaign-wide error rate from repeated `2 * SE` or bootstrap checks.
5. **Record** the plan's report row per case and seed, plus the seed-block identifier.
   The accepted candidate becomes the "current accepted best" for the `D2` rule. The
   frozen baseline for the guards does not move.

## 6. Correctness where the metric is scored

C1 tops out at six qubits because a full operator has `4**n` entries. The focused panel
uses 100-logical-qubit inputs compiled onto a 193-qubit target; the general panel adds
other widths, targets and circuit families. C2/C3's structural
checks can scale to these outputs, but the existing ASV trackers do not run them. A
change can break large-circuit routing and still pass small tests. Two **proposed**
additions improve coverage without proving the generic-angle outputs equivalent:

1. **Legality and mapping on every scored output.** Apply C3 (every instruction legal on
   the actual target, in the supported direction) and C2's mapping checks
   (`final_index_layout()` is a valid injective map) to the very outputs whose `D2` is
   recorded. This is cheap. It catches illegal outputs, not wrong ones.
2. **Equivalence at scale with Clifford variants.** For suitable CX/rotation fixtures,
   replace each single-qubit rotation angle by an explicitly chosen odd multiple of
   π/2. This restriction would not make arbitrary controlled rotations or arbitrary
   matrix gates Clifford. Confirm the constructed input is accepted by `Clifford`.
   Compare polynomial-size tableaux with `approximation_degree=1.0` and
   `qubits_initially_zero=False`; equality verifies the Clifford unitary up to global
   phase. Floating-point angle recognition still has numerical tolerances.

   Unlike `Operator.from_circuit`, `Clifford.from_circuit` does **not** apply transpiler
   layout metadata. For a fixture whose extra wires must be acted on as identity,
   construct an expected circuit at the full output width. Compose the logical variant
   on `initial_index_layout(filter_ancillas=False)[:input_width]`, then append the wire
   permutation mapping each full initial position to its full final position from
   `final_index_layout(filter_ancillas=False)`. With `PermutationGate`, whose pattern
   lists input positions in output order, set `pattern[final[i]] = initial[i]`.
   Compare `Clifford(expected) == Clifford(output)`; do not discard ancillas or compare
   a 100-qubit tableau directly with a 193-qubit one. A zero-initialized-only ancilla
   contract needs a restricted-state check instead of this full-unitary assertion.

Scope and limits of the Clifford check:

- General-panel families need fixture-appropriate oracles. Arbitrary QV unitaries and
  reversible nonlinear logic cannot be verified just by replacing rotation angles.
  Use exact small counterparts, targeted synthesis/rewrite tests and applicable
  structured large-instance checks; record what remains unverified. Numeric bindings
  do not prove symbolic equivalence. No universal large-circuit semantic oracle is
  supplied by this proposal, and seed or workload diversity does not replace correctness.
- It checks all inputs to the Clifford variant, but represents the *scored* routing
  problem only if the variant keeps its two-qubit interaction structure. Angle 0 deletes a rotation and lets
  the surrounding CX pair cancel. Angle π can turn a `CX·rz·CX` block into a product of
  single-qubit gates. Even odd multiples can simplify differently. Inspect the ordered
  interaction DAG entering layout/routing, including dependencies; similar gate counts
  alone do not establish the same routing problem.
- SABRE's distance heuristic does not score rotation angles, but synthesis,
  simplification, and VF2 can change the problem before SABRE sees it. Layout/routing
  paths and final `D2` therefore need not match the scored circuit. Generic-angle
  rewrites still rely on C1/C4 and targeted regressions.
- Two-qubit resynthesis is not guaranteed to emit gates that are individually Clifford.
  If `Clifford(output)` rejects an angle, the result is "unverified", not "wrong" or a
  pass. A separate run ending after translation can avoid optimization-stage
  resynthesis, but earlier synthesis may still need a Clifford-preserving fixture or
  configuration. Label exactly which stages were checked: a prefix check does not
  verify the optimized output. Keep any uncovered stage explicitly unverified until
  an appropriate oracle or targeted regression covers the changed behavior.

## 7. What exists and what must be built

| Need | State |
| --- | --- |
| `D2` per case | **Existing** for B3: `track_*_depth` in `utility_scale.py`, one seed; **proposed** native-depth companions for B2, QUEKO, qualitative and new fixtures |
| Compile time per case | **Existing**: ASV `time_*` methods for B1–B3 and the QAOA extension; **proposed** full general-panel end-to-end/reuse timings and multi-seed companions for search changes |
| `N2` per case | **Proposed**: `output.count_ops()` summed over the target's native two-qubit names, from the same run as `D2` |
| Multi-seed scorer: per-case ratios, `SE`, frozen-baseline guards, seed blocks | **Proposed**: a script outside ASV that varies `seed_transpiler` and rebuilds the preset per seed |
| Legality and mapping on scored outputs; Clifford variants | **Proposed** (section 6) |
| Confirmation blocks and held-out circuits | **Proposed** (section 5) |
| Reproducible QV inputs; QUEKO depth-model match | **Proposed**: freeze QV before comparisons; match QUEKO's model before optimality claims |
| G1–G8 input/target/level matrix and manifest | **Proposed**: reuse existing anchors, add missing sizes and independent instances, hash inputs/targets and validate coverage before tuning |
| General scorer and breadth report | **Proposed**: fixed family-balanced weights, marginal summaries, paired instance/seed bootstrap, practical-effect and leave-one-family-out checks |
| Independent circuit validation and reserves | **Proposed**: split by input group, fresh circuits and seeds per decision; fresh seeds alone do not qualify |
| General-panel memory and cost guards | **Proposed**: fresh-process RSS, per-case/per-family timings, noise calibration and complete reports |

Build in this order: `N2` and the multi-seed scorer; at-scale correctness checks;
reproducible held-out fixtures (including QV); then confirmation and held-out guards.
All are prerequisites for automatic acceptance under the focused policy. For general
qualification, additionally build and validate the expanded manifest and fixtures, cost
and memory companions, weighted scorer, instance uncertainty and independent validation
protocol. Section 4's measurements do not validate those additions. QUEKO's
depth-model validation is additionally required before claiming distance from optimum.
