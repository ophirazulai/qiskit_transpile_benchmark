# Standalone Qiskit transpilation benchmark: high-level plan

## 1. Purpose and scope

Build a dedicated repository that depends on Qiskit and answers:

> Given a baseline Qiskit source folder and an evolved Qiskit source folder,
> does the evolved version improve transpilation under the declared constraints?

The repository owns the workloads, measurement runner, correctness checks, decision
policy, and reports. Each supplied Qiskit version is built and installed as a dependency
in its own isolated environment. The benchmark does not live inside either checkout
and does not select its workload definitions from the evolved checkout.

The initial objective is **lower native two-qubit depth (`D2`)**, subject to correctness,
native two-qubit gate count (`N2`), and compilation-time limits. Broad qualification
also constrains peak memory. This is a constrained decision, not a weighted sum:
better depth cannot compensate for incorrect output or a failed cost guard.

The scored population is static, gate-based transpilation with exact synthesis.
Dynamic circuits, symbolic-parameter semantics, scheduling, and batch execution have
separate regression checks where applicable. Hardware execution, noise characterization,
simulator speed, parsing speed, and fault-tolerant compilation are outside this score.
A pass does not imply that every circuit improves or that hardware execution is faster.

This is a proposal, not an implemented runner or a measured improvement. It is
self-contained: implementation and interpretation do not require another design document.
All thresholds below are initial policy defaults to freeze before evaluating candidates.

## 2. User interface and result

The two required inputs are local source-folder paths. A proposed command is:

```text
qiskit-transpile-bench compare \
  --baseline /path/to/baseline-qiskit \
  --evolved /path/to/evolved-qiskit
```

With no other options, use the repository's versioned focused profile, create a new
comparison campaign, and write results to a new run directory. Optional arguments
select a frozen campaign manifest, the general profile, output location, or an existing
campaign. The command must print the selected scope and expected work before measurement.

Two reduced modes can never return `PASS`. A quick smoke mode checks execution only and
writes no acceptance decision. A screening mode stops after the tuning panel, so
iterating on a candidate exposes no held-out inputs, confirmation seeds, or validation
reserves; when its rules hold it reports `INCONCLUSIVE` with the unrun checks marked
`not_evaluated`.

The output includes a short terminal verdict, a Markdown report, a machine-readable
`decision.json`, raw measurements, logs, and reproduction metadata. The decision
contains `status`, `improved_under_constraints`, profile, comparison identities,
depth change and uncertainty, every constraint's result, and reasons with case IDs.

| Status | Meaning | `improved_under_constraints` |
| --- | --- | --- |
| `PASS` | Improvement demonstrated and all required constraints satisfied for the named profile | `true` |
| `NO_IMPROVEMENT` | A complete, valid quality panel does not demonstrate the required improvement; no evaluated constraint has a confirmed violation | `false` |
| `CONSTRAINT_VIOLATION` | A correctness failure or a sufficiently established quality/cost breach blocks acceptance | `false` |
| `INCONCLUSIVE` | Missing evidence, unstable measurements, or unresolved semantic coverage prevents a decision | `null` |
| `ERROR` | Invalid input, incompatible build, or harness failure prevents a valid comparison | `null` |

Only `PASS` gets a successful acceptance exit code; other statuses have documented,
distinct exit codes. Reports retain all failures, even when execution stops early.
Confirmed failures take precedence over a lack of improvement; missing measurements
can never produce a pass. A baseline correctness failure, crash, or timeout invalidates
the comparison and yields `INCONCLUSIVE`; matching an incorrect baseline is not success.
An unexpected evolved-version compilation crash or timeout on a supported case is a
constraint violation, while a broken measurement environment is an error.

`NO_IMPROVEMENT` means improvement was not demonstrated under this policy, not that
the two versions are equivalent. It can stop before cost or confirmation runs; mark
unrun checks `not_evaluated` without claiming their constraints were satisfied.
Complete evidence is required for `PASS`. A complete quality panel that misses the
statistical improvement threshold yields `NO_IMPROVEMENT`; an incomplete or invalid
panel yields `INCONCLUSIVE` unless an established failure already blocks acceptance.

## 3. Repository responsibilities and architecture

| Component | Responsibility |
| --- | --- |
| Campaign configuration | Versioned profiles, concrete case IDs, weights, exclusions, limits, seeds, splits, and policy thresholds |
| Fixtures and targets | Repository-owned circuit data/generators and frozen synthetic target descriptions |
| Environment builder | Snapshot each input folder, build Qiskit including its native extension, install locked dependencies, and verify provenance |
| Revision worker | Load one installed Qiskit, reconstruct inputs, compile, run the API-behavior checks that need the live library, and emit canonical outputs with structured observations |
| Verifier | In a repository-pinned trusted environment, extract `D2`/`N2` and check legality and semantics from the canonical outputs |
| Coordinator | Preflight, correctness, seed scheduling, paired measurements, timeouts, caching, and campaign state |
| Evaluator | Compute ratios, scores, uncertainty, coverage, and every acceptance guard from saved observations |
| Reporter | Produce the verdict, summaries, per-case evidence, failures, and reproducible run bundle |

The coordinator communicates with workers through versioned files or a process protocol.
Never import both Qiskit versions in one Python process or exchange live Qiskit objects
between them. Keep scoring and reporting independent of either installed Qiskit so the
same saved observations can be evaluated again without recompiling circuits.

The revision under test must not grade itself. Workers export each output circuit and
layout in a canonical, Qiskit-independent form. Harness-owned code computes `D2`, `N2`,
and target legality from that form, and semantic oracles run in the verifier's pinned
environment. A candidate that changes `depth()`, `Operator`, or `Target` queries then
cannot alter its own score or oracle.

Prefer public Qiskit interfaces through a small version adapter. Unsupported interfaces
are reported explicitly; the adapter must not silently alter target properties, options,
or the search budget to make a candidate run. Existing Qiskit benchmark fixtures can be
curated into this repository with provenance and licensing retained. Qiskit's internal
ASV suite can provide supplementary diagnostics, but is not the standalone decision engine.

## 4. Reproducible execution of the two folders

1. **Snapshot the actual sources.** Accept clean or modified folders. Record resolved
   paths, commit IDs when available, dirty state, and content hashes, including relevant
   untracked source files. Exclude environment directories, build outputs, and Git
   metadata from the build input. Do not modify the supplied folders. Snapshot first so
   edits during a long run cannot change what is measured.
2. **Build independently.** Use separate environments and fresh build directories with
   the same Python, compatible locked non-Qiskit dependencies, Rust toolchain, release
   profile, allocator and Cargo feature flags, and compiler settings. Sanitize
   build-affecting environment variables such as `QISKIT_BUILD_PROFILE`, `RUST_DEBUG`, and
   `QISKIT_BUILD_WITH_MIMALLOC`. Each checkout's `rust-toolchain.toml` can silently select
   a different compiler, so verify the resolved toolchain of both builds. Build native
   code from each snapshot; never reuse a checkout's possibly stale extension. Verify
   Python package and native-extension origins and save build logs and artifact hashes.
   Execute outside both source trees with a sanitized import path. An incompatible
   common dependency set blocks the code-only comparison; a changed environment requires
   a separately labeled experiment.
3. **Freeze benchmark inputs.** Build both versions' circuits from the same canonical
   fixture data and verify their normalized semantics and target descriptions agree.
   Hash matrices, numeric values, connectivity, directionality, supported instructions,
   durations, and errors. A generation seed alone is insufficient if generator behavior
   changes across Qiskit versions. Do not use live backend calibration data.
4. **Pin execution options.** Record optimization level, methods/plugins, initial layout,
   approximation and ancilla contracts (including `qubits_initially_zero`), search trials,
   and worker counts. Exact scored synthesis uses `approximation_degree=1.0`. The serial
   reference sets `QISKIT_PARALLEL=FALSE`, `QISKIT_IGNORE_USER_SETTINGS=TRUE`,
   `RAYON_NUM_THREADS=1`, and `OMP_NUM_THREADS=1`; unset `QISKIT_SABRE_ALL_THREADS`, which
   takes effect when set to any non-empty value, even `FALSE`. Freeze direct SABRE trial
   counts and record resolved preset budgets: a thread limit alone does not fix search
   effort. Report a budget that differs between the revisions as a configuration change,
   not an algorithmic gain.
5. **Control measurement conditions.** Measure cost for both revisions on the same
   machine, one measurement at a time, with balanced/interleaved order and warmups.
   Record CPU, OS, available cores, thread libraries, and resource limits. Keep circuit
   loading, verification, metric extraction, and diagnostic callbacks outside compilation
   timing. Run alternate parallel settings as separate profiles; do not mix their
   observations with serial results. Quality observations should be deterministic for a
   fixed build, configuration, and seed, so they may run concurrently in separate
   serially configured workers, but never alongside a cost measurement. Verify that
   determinism by repeating a sample in fresh processes; a mismatch is an unstable
   measurement and invalidates quality caching for that revision.

Cache keys include source/build identities, dependency and machine identities, harness
and policy versions, manifest and fixture hashes, options, worker settings, seed block,
and measurement mode. Reuse only exact matches. Timing and memory cache reuse additionally
requires valid environmental/noise calibration; stale cost data must be remeasured.

## 5. Workloads and qualification scope

Every case is a circuit, target, and explicit compilation configuration. The frozen
manifest enumerates actual cases rather than only listing desired workload families.
It records each case's role (scored, guard, canary), family, size, topology, native basis,
level, input-group ID, split, weight, oracle, timeout, and measurement modes.
Declare unsupported combinations before evaluation; a candidate failure is not an exclusion.

### Focused profile: first deliverable

Score three 100-logical-qubit circuits: QFT, square-lattice Heisenberg simulation, and
three-layer QAOA on a Barabási–Albert graph. Freeze the exact curated inputs. Use level 2
on a heavy-hex distance-9 target (193 physical qubits), with a frozen `cz` native basis
and target properties. Give the three cases equal weight.

Use the same inputs on `cx` and `ecr` target variants as guards. Include separate
held-out QUEKO routing instances, frozen quantum-volume circuits, reversible logic
(`hwb12`), and Bernstein–Vazirani patterns. Keep simplification canaries, including a
100-qubit circular efficient-SU2 circuit and a long two-qubit sequence that simplifies
strongly. Establish their expected values on the validated baseline before tuning;
an unexplained departure prevents automatic acceptance.

The timing panel covers tiny one-gate and cancellation inputs, 14-qubit quantum-volume
and a long fixed two-qubit circuit at levels 0–3, and the three scored circuits on all
three bases with reusable level-2 managers. Freeze precise call boundaries per case.
This retains wrapper overhead, complete compilation, and larger repeated compilations.

A pass means **focused-panel native two-qubit depth improvement**. These cases alone
cannot support a general claim, regardless of the number of transpiler seeds.

### General profile: subsequent qualification

Require all eight families in both tuning and independent validation: QFT, Hamiltonian
simulation, QAOA, quantum volume, reversible logic, Bernstein–Vazirani patterns,
variational ansatz circuits, and routing challenges.

- Cover small (4–16), medium (17–64), and large (65–100) logical widths where supported;
  each split includes all three bands. Each supported family/size cell needs at least
  three independent input groups per split. Declare unsupported cells with reasons.
- Cover heavy-hex, line, 2D grid, and an all-to-all control, in both splits. Each family
  uses at least two sparse topology classes unless its fixture fixes its target.
- Include `cx`, `cz`, and `ecr`, supported asymmetric directionality, targets with spare
  qubits and fully occupied targets, and levels 0–3 for every supported input/target pair.
- Keep related target, basis, level, and parameter variants in one input group and one
  split. Wire relabelings or fresh transpiler seeds do not create independent inputs.
  Freeze quantum-volume matrices; keep bindings of one ansatz together.
- Allocate weight equally to families, then within each family to supported size bands,
  topologies, bases, levels, input groups, and declared variants, in that order. Persist
  final weights. The all-to-all control is guard-only unless declared scored in advance.
  Zero-baseline guards remain outside logarithmic scores; every family still needs
  positive scored cases. Normalize included weights for each marginal summary.

Time every scored workload in separate end-to-end and reusable-manager panels. Freeze a
fresh-process peak-memory panel covering every family and size band, including the largest
cases. Profile baseline feasibility before freezing the campaign. If the full panel is
too costly, declare a narrower scope before tuning; never remove unfavorable cases later.

## 6. Metrics and correctness

| Measurement | Definition |
| --- | --- |
| `D2`, primary objective | Dependency depth filtered to executable native two-qubit gates in the final translated circuit |
| `N2`, quality constraint | Count of all executable native two-qubit gates in that same output |
| Compilation time | Wall time of either a complete `transpile()` call or `pm.run()` with preset construction outside timing; separate panels |
| Preset construction time | Separate measurement when pipeline construction changes |
| Peak memory | Repeated fresh-process peak RSS, with identical process boundaries and setup baseline recorded; median per case |
| Diagnostics | Total depth, stage times, pre-decomposition routing SWAPs, scheduled duration where applicable, worst-seed outcomes, and failures |

Do not treat total depth as `D2`, final zero SWAP count as zero routing overhead, or
depth as physical duration. Persist the target's native two-qubit instruction set and
exclude barriers, delays, and control-flow containers from `D2`/`N2`.

Both revisions must pass correctness independently. The repository owns these gates:

- **Small semantic tests:** exact 1–6-qubit unitary equivalence up to global phase,
  with layout accounted for, exact synthesis, the all-input contract
  (`qubits_initially_zero=False`), and declared numerical tolerances (initially
  `rtol=1e-7`, `atol=1e-8`). Include cancellations, routing, matrix gates, levels 0–3,
  and phase bookkeeping under controlled composition.
- **Layout and ancillas:** valid injective logical-to-physical maps, correct classical
  measurement destinations and observable interpretation, and declared clean/zero-state
  ancilla contracts. Do not compare operators of different widths directly.
- **Target legality on every scored output:** actual ordered physical qubits, supported
  instructions and parameters, and angle bounds. Recursively map control-flow wires
  for applicable regression fixtures. Basis-name checks alone are insufficient.
- **Behavioral regressions:** measurement/reset/control-flow semantics, symbolic
  parameter meaning under representative bindings, expected unsupported-input errors,
  input immutability, manager reuse, and batch ordering. Check scheduling resources,
  dependencies, durations, and alignment where relevant.
- **Large-instance evidence:** fixture-appropriate structured oracles, small counterparts,
  and targeted regressions. Clifford variants can permit scalable tableau comparison,
  but require full-width layout/ancilla alignment and may not preserve the scored
  routing problem. Optimized outputs need not remain representable as Clifford gates.

Record `verified`, `mismatch`, or `unverified`, with the oracle and stages covered.
An equivalence mismatch blocks acceptance. Structural legality is not semantic proof.
Automatic acceptance requires, for every scored circuit/target, a verified check whose
stage coverage contains every changed stage. A prefix check does not cover later
optimization. Determine changed stages conservatively from the source diff and a
recorded change-scope declaration; unknown scope requires full coverage or review.
Unresolved coverage yields `INCONCLUSIVE`, with any human-reviewed trade recorded
separately rather than converted into a pass. Run affected upstream Python/Rust tests
in addition to the harness checks; record required test selection and results, and
report test files that the evolved folder changed or removed.

This rule currently limits what can pass automatically. In a one-seed feasibility check
on Qiskit `2.6.0.dev0`, Clifford variants of the three focused circuits verified the
complete level-2 pipeline for only one of the nine circuit/target pairs, because
two-qubit resynthesis emits non-Clifford angles. A prefix through translation, with the
optimization stage removed and Clifford unitary synthesis selected, verified all nine.
Until a stronger at-scale oracle exists, automatic `PASS` is available only to
candidates confined to layout and routing. A candidate that changes the optimization
stage or two-qubit synthesis can reach at most `INCONCLUSIVE`, pending review.

## 7. Decision policy

Use matching transpiler seeds 0–99 for tuning. For each positive quality metric `m`
(`D2` or `N2`), case `c`, and revision pair, compute the ratio of geometric means across
seeds. Compute the weighted geometric mean of case ratios as the suite score. For
uncertainty, define:

```text
delta_s = sum_c weight_c * (ln(m_evolved(c, s)) - ln(m_reference(c, s)))
ln(score) = mean(delta_s)
SE = sample_standard_deviation(delta_s) / sqrt(number_of_seeds)
```

This preserves observed covariance across cases and revisions. Matching seed IDs does
not imply matching heuristic trajectories. For a family, stratum, or single-case guard,
restrict the sum to those cases and renormalize their weights; this defines `SE_family`
and `SE_case` below. Report each case ratio before aggregates. Zero baselines use
absolute deltas and permit no increase for automatic acceptance. If a positive scored
case reaches zero in the evolved version on any seed, report it separately and require
review; never take `ln(0)`, add arbitrary offsets, or silently change weights.

Cost measurements use their own declared estimators: a frozen aggregate of repeated
timing samples for fixed-seed panels, arithmetic means across seeds for timing
companions, and medians of repeated peak-RSS measurements. Do not apply the quality
seed estimator to cost.

For a new two-folder comparison, the supplied baseline is both the frozen campaign
baseline and the current accepted best. In a continuing campaign, retain the original
baseline for every regression guard and store the current best as a separate reference.
The two path arguments still identify the frozen baseline and proposed evolved version;
campaign state supplies the best snapshot. Never move the guard baseline after a pass;
a supplied baseline whose identity differs from the campaign's frozen one is an `ERROR`.

### Focused acceptance

- On the primary `cz` panel require `ln(D2_score) + 2 * SE < 0` against the current best.
- Against the frozen baseline, on each of `cx`, `cz`, and `ecr`, require
  `ln(D2_score) <= 2 * SE` and `ln(N2_score) <= 2 * SE`, with no seed-aggregated case
  ratio above 1.05 for either metric.
- Held-out cases each require `ln(case_ratio) <= 2 * SE_case` and ratio at most 1.05
  for both quality metrics against the frozen baseline. Canaries require explained,
  freshly validated outcomes; the tool reports any departure from their expected values
  as `INCONCLUSIVE` pending that review.
- The timing panel's geometric-mean ratio, equally weighted per timing case, must be
  at most `1 + calibrated_relative_noise`; no case may exceed 1.10. A randomized-search
  change, or one whose declared scope is unknown, also needs a separate multi-seed
  timing companion, using arithmetic mean elapsed time across seeds per case, under the
  same guards. All timing comparisons use the frozen campaign baseline.
- All required correctness, measurements, and confirmation checks must pass.

### General acceptance

The broad panel and rules below replace focused quality scoring; a general pass does
not additionally require improvement on the narrow three-circuit score. Retain applicable
correctness, canary, zero-baseline, overhead/regression timing, and confirmation checks.
Apply these rules on both broad tuning and independent validation panels:

1. Require at least a 1% practical `D2` reduction against both current best and frozen
   baseline: `ln(D2_score) + 2 * SE < ln(0.99)`, and the instance-uncertainty upper bound
   described below must also be below `ln(0.99)`.
2. At least four of eight families must have `ln(D2_family_score) + 2 * SE_family < 0`.
   Removing any one family and renormalizing must leave a `D2` score below 1. Apply
   these breadth checks against both references.
3. Against the frozen baseline, every family, size, topology, basis, and level summary
   must have `ln(score) <= 2 * SE` for both `D2` and `N2`; the overall `N2` score must
   also pass. No positive case ratio may exceed 1.05. Apply absolute zero-baseline guards
   and retain all-to-all guards even when that topology is outside the primary score.
4. Each general timing panel and family summary must stay within its calibrated noise
   allowance; no case may exceed 1.10. Use the quality weights. Apply the focused
   overhead/regression timing guards separately. The memory aggregate uses equal
   family weights with selected case weights normalized within family; it must stay
   within its measured noise allowance, and no memory case may exceed 1.10. All timing
   and memory comparisons use the frozen campaign baseline.
5. Required coverage, independent inputs, semantic evidence, and measurements must be
   complete, with zero unexpected crashes or timeouts.

Estimate instance uncertainty with a frozen paired cluster bootstrap: 10,000 replicates,
resampling independent input groups within compatible family/size strata and seeds as
whole shared vectors. Keep all related variants together, preserve paired revision
observations, and recompute scores with fixed configuration weights. Require at least
three scored groups per compatible stratum in each split; validate identical configuration
support within strata, refining strata before freezing if needed. A group spanning sizes
must remain one block. Record the RNG seed and use the 95th percentile of bootstrapped
log-scores as the upper bound. Guard-only all-to-all cases have their own normalized
topology summary and per-case caps, and are excluded from other scored summaries.

Calibrate timing and memory noise with repeated baseline-versus-baseline trials before
candidate selection. Freeze repetitions, aggregation, per-case absolute noise floors,
relative allowances, and a bounded confirmation rule for noisy breaches. A per-case cost
cap is breached only when the ratio exceeds the cap and the absolute increase exceeds
that case's noise floor, so tiny timings are not judged by ratios alone. An unresolved
breach is inconclusive, not a pass; do not rerun until favorable.

Guard multiplicity works against a good candidate. A one-sided `2 * SE` guard trips on
about 2% of comparisons with no true change, so `k` independent noisy guards reject a
neutral candidate with probability about `1 - 0.977**k`: roughly 20% at `k = 10` and 50%
at `k = 30`, and confirmation applies every guard again. Count the noisy guards in the
frozen manifest. Measure the false-rejection rate of the whole guard set with
baseline-only null comparisons between disjoint calibration seed blocks that are never
used for tuning or confirmation, and report it. If the rate is unacceptable, predeclare
the remedy before evaluating candidates: a wider per-guard multiplier, or one rerun of a
breached guard on a fresh seed block. Never rerun a failed guard until it passes.

These uncertainty rules are empirical screens, not a universal correctness proof or a
formal campaign-wide false-acceptance guarantee under repeated candidate selection.

## 8. Run lifecycle and retained evidence

1. Validate paths and profile; snapshot and build both revisions; verify provenance.
2. Validate baseline correctness, fixture compatibility, coverage, and available oracles.
   Profile cost, calibrate noise, and freeze the manifest before tuning candidates.
3. Run evolved correctness first, then tuning-panel quality and in-panel guards.
   Measure cost for candidates that pass quality. Reserve held-out inputs and
   acceptance-only canaries for confirmation. Persist observations as they finish;
   early rejection records which later measurements were not attempted.
4. Freeze a promising candidate and run confirmation on a fresh 100-seed block
   (100–199, then 200–299, and so on). Evaluate candidate and both references on that
   same block; never compare fresh candidate seeds with cached tuning seeds.
   Recheck the tuning panel's quality rules, and evaluate the held-out guards and
   acceptance canaries. General qualification additionally requires the independent
   validation panel to improve under all rules.
5. Retire exposed confirmation seed blocks after every decision, including rejection.
   Retire general validation inputs too; use predeclared reserves for the next attempt.
   Retain exposed cases as regressions, and refresh focused held-out inputs when repeated
   inspection makes them tuning data. Store unsuccessful attempts as well as passes.
   Retirement is recorded per campaign, so evaluate successive candidates from one line
   of work in the same campaign: a new campaign restarts at block 100–199 with the same
   held-out inputs and cannot know what earlier campaigns exposed. Report the number of
   decisions already made in the campaign.
6. Write the final report with every constraint marked passed, failed, unresolved, or
   not evaluated. Acceptance requires all applicable constraints to pass. Record a
   passing candidate as current best only in the selected campaign; the original
   baseline remains immutable.

Each observation records case/group/split IDs, hashes, revision/build identity, options,
seed and seed-block ID, weight, timing mode, raw samples, canonical output hash, `D2`,
`N2`, applicable memory or scheduling metrics, correctness status and stage boundary,
and errors. Inapplicable metrics are absent, not zero. Retain enough canonical
input/output and layout data to reproduce correctness failures, including output
artifacts for failed or disputed cases.

The report leads with the scope-qualified verdict and failed constraints, then gives
aggregate improvement with uncertainty, per-family/stratum results where applicable,
per-case ratios, absolute zero-baseline deltas, worst seeds, time/memory changes,
correctness coverage, exclusions, and reproduction instructions. QUEKO optimality claims
require matching its reference basis, allowed rewrites, and depth model; otherwise use
it only as a fixed regression workload.

## 9. Delivery sequence and completion criteria

1. **Standalone runner:** two paths, isolated reproducible builds, canonical fixtures,
   worker protocol, canonical output export, raw output, and provenance. Smoke runs
   cannot issue acceptance.
2. **Focused evaluator:** verifier-computed same-output `D2`/`N2`, correctness including
   changed-stage coverage, 100-seed scoring, screening mode, cost and false-rejection
   calibration, held-out guards, fresh confirmation, campaign state, and complete
   machine/human reports.
3. **General qualification:** expanded eight-family fixtures, validated independent
   splits and reserves, family-balanced scoring, bootstrap uncertainty, breadth checks,
   complete timing coverage, and fresh-process memory measurement.
4. **Automation and maintenance:** repeatable controlled-runner jobs, supported Qiskit
   compatibility range, fixture/policy versioning, and deterministic failure reproducers.

Before each profile can issue `PASS`, validate the evaluator with known outcomes:
identical builds yield identical quality observations and no claimed improvement;
injected semantic/legality errors fail, including a revision whose own depth or
equivalence functions misreport; synthetic metric regressions, missing cases, zero
values, and exhausted validation reserves cannot pass; verifier metric extraction,
paired weighting, and uncertainty calculations reproduce reference examples. Verify
process isolation and that source edits invalidate cached builds. Exercise report
generation and resumption from saved observations without accepting partial results.
Finally, run both real Qiskit folders end to end on the controlled runner and inspect
the evidence behind every decision field.
