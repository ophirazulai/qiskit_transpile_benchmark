# Benchmarks and regression checks for transpilation

This is the validation plan for [the transpilation flow](transpilation-flow.md):
`QuantumCircuit` → staged pass manager → target-compatible circuit, layout, and
optional timing metadata. Source baseline: Qiskit `2.6.0.dev0`, commit `0131cbbcc`.

To know whether a code change broke transpilation, check three independent things:

1. **Correctness:** the circuit still implements the intended computation, respects
   the target, and returns valid mappings and timing information.
2. **Output quality:** compilation has not unexpectedly increased native two-qubit
   gate count, depth, or scheduled duration.
3. **Compiler cost:** compilation has not unexpectedly become slower or used more memory.

A fast compiler can produce a wrong circuit, and a correct compiler can produce a
much worse circuit. Passing performance benchmarks alone is insufficient. No finite
suite proves correctness for all inputs; the plan combines small strong checks with
representative larger workloads and preserves every discovered bug as a regression test.

**Status:** this document selects existing benchmarks and specifies missing coverage.
The general-improvement panel in section 1.1 and its acceptance/reporting harness are
**proposed**. Existing ASV results alone do not satisfy that protocol.
For an input-by-input explanation of what the code actually executes, see
[the detailed benchmark walkthroughs](#7-detailed-walkthrough-what-each-selected-benchmark-actually-does).

It does not implement new benchmarks, add CI gates, or report measured results. Labels
**existing** and **proposed** distinguish what can run today from additions needed for
complete coverage. Suggested thresholds below are policy proposals, not current Qiskit CI rules.

## 1. Scope and the minimum useful subset

Keep the standard gate-based pipeline, including its Python/Rust boundary, target
handling, symbolic parameters, supported control flow, and optional scheduling.
Exclude hardware execution, noise characterization, primitives, visualization,
serialization, import time, circuit-construction speed, and simulator performance.
Simulation is used only as a correctness oracle, outside compilation timing.
Clifford+T and Clifford+RZ benchmarks in `transpiler_ft.py` are outside this design's
standard pipeline; include them only when changing those specialized pipelines.

Use this selection rather than running every file under `test/benchmarks`:

| Tier | Selected coverage | When to run |
| --- | --- | --- |
| Required correctness | The tests in section 4, with a new regression test for the behavior being changed | Every relevant code change; failures block acceptance |
| Core compiler benchmarks | B1–B3 below: tiny circuits, representative complete compilation, and larger reusable-pass-manager workloads | Every transpilation change, on a controlled benchmark machine |
| Stage-specific benchmarks | The relevant rows in section 3 | When changing the corresponding stage or diagnosing a core regression |
| Extended transpilation checks | Larger scale cases, multiple seeds, memory, batch mode, scheduling, and randomized equivalence | Before accepting heuristic, parallelism, scheduling, or shared infrastructure changes; otherwise periodically |
| General-improvement qualification — proposed | G1–G8 in section 1.1, across sizes, targets and levels, with independent validation inputs | Required before claiming a general improvement in standard-pipeline `D2`; the core subset alone supports only a panel-specific claim |

The core is small in **workload families**, not guaranteed to take a few seconds.
It selects 10 methods and 30 parameterized measurements before ASV repetitions:
2 in B1, 16 in B2, and 12 in B3. Establish its actual runtime before deciding which
cases belong in a pull-request job versus a nightly job.

### B1. Small-circuit compilation overhead — existing

Source: [`transpiler_benchmarks.py`](../test/benchmarks/transpiler_benchmarks.py),
`TranspilerBenchSuite`. Select only:

- `time_single_gate_compile`: one H gate.
- `time_cx_compile`: two H gates and four CX gates on two qubits; all can cancel.

Both call `transpile()` with a fixed 27-qubit coupling map, an `rz/sx/x/cx`-based
vocabulary, and seed `20220125`. The optimization level is **omitted**, so these
measure the checkout's default behavior. Circuit construction is in `setup()`.

**Measures:** wrapper overhead, preset construction, circuit/DAG conversion, physical
embedding, and passes whose fixed overhead dominates these tiny inputs.
**Metric:** elapsed seconds per complete call; lower is better, subject to correctness.
**Detects:** expensive initialization, redundant graph copies, repeated target analysis,
and overhead that is hidden by large workloads.

Do not interpret the CX case as a standalone cancellation timing measurement. Its
asserted simplification and equivalence belong in correctness tests. If the default
optimization level changes, report that configuration change explicitly; use an
explicit-level companion benchmark to distinguish it from an implementation slowdown.

### B2. Complete compilation across optimization levels — existing

Source: [`transpiler_levels.py`](../test/benchmarks/transpiler_levels.py),
`TranspilerLevelBenchmarks`. Select these four methods at levels **0, 1, 2, and 3**:

| Methods | Workload and target | Why retain it |
| --- | --- | --- |
| `time_transpile_qv_14_x_14`, `track_depth_transpile_qv_14_x_14` | Quantum-volume-style circuit of width/rounds 14/14 (matrix-seeding caveat in section 7.3); seeded 14-qubit `GenericBackendV2` with Melbourne connectivity | Dense interactions exercise synthesis, layout, routing, translation, and optimization with a backend target |
| `time_transpile_from_large_qasm`, `track_depth_transpile_from_large_qasm` | Checked-in `test_eoh_qasm.qasm`, loaded during setup; explicit legacy basis and Rochester coupling map | A structured fixed circuit exercises the separate-constraints API and guards against overfitting to random circuits |

**Measures:** total `transpile()` latency and the resulting circuit's total depth.
**Metrics:** seconds per call and `output.depth()` in dependency layers; compare the
same case and optimization level across revisions. These depth trackers do not measure
nanoseconds, two-qubit-only depth, fidelity, or equivalence. File parsing is outside
the timed methods despite the QASM names.

**Detects:** interactions between stages, level-specific regressions, and cases where a
faster compiler trades away circuit quality. Do not require level 3 to beat level 2
on every circuit: the heuristics make no such guarantee. For a broader integration
change, add the existing 50×20 quantum-volume timing/depth pair and the backend-based
`*_transpile_from_large_qasm_backend_with_prop` pair.

### B3. Larger circuits and multiple native entangling gates — existing

Source: [`utility_scale.py`](../test/benchmarks/utility_scale.py),
`UtilityScaleBenchmarks`. Select:

- `time_qft` and `track_qft_depth`: 100-qubit QFT with nonlocal interactions.
- `time_square_heisenberg` and `track_square_heisenberg_depth`: structured 100-qubit
  simulation circuit with a different interaction pattern.

Run each for the existing `cx`, `cz`, and `ecr` parameters. Setup builds a seeded
`GenericBackendV2` on `CouplingMap.from_heavy_hex(9)` and a reusable level-2 preset.

**Measures:** `pm.run(circuit)` latency after preset construction, and output depth
filtered to the selected native entangling instruction. **Metrics:** seconds per run
and native two-qubit dependency layers. Here the `track_*_depth` values differ in
meaning from B2's total depth. Compare each native basis with itself across revisions;
a CX count/depth is not automatically interchangeable with an ECR count/depth.

**Detects:** scaling problems, costly routing, and translation or optimization changes
that help one gate vocabulary while hurting another. Preset construction and QASM
loading happen in setup and are not included in these timings. The class still prepares
its other fixtures during setup even when only these methods are selected.

Retain existing `time_circSU2`/`track_circSU2_depth` as the targeted extension for
symbolic-parameter changes, and `time_qaoa`/`track_qaoa_depth` for further routing
coverage. Do not select `time_parse_*`: they measure parsing, outside this boundary.

### 1.1. General-improvement panel — proposed

B1–B3 are an iteration and regression subset. Three 100-qubit circuits on one graph at
level 2, even with many transpiler seeds, cannot establish broad workload coverage.
For a general-improvement campaign, use the following additional contract. Here
“general improvement” means **lower native two-qubit depth across the declared static,
gate-based workload population, with bounded gate-count and compiler-cost regressions**.
It does not mean every circuit improves, physical execution is faster, or all compiler
features improve. Scheduling, dynamic circuits, approximate synthesis, and fault-tolerant
pipelines need separate objectives and evidence; retain their applicable correctness
and regression checks when shared code changes.

The existing fixtures below are starting points, not a complete implemented panel.
Add versioned generators or fixtures to fill the size and validation requirements.
Reuse whole-pipeline workloads; isolated pass timings are supporting diagnostics.

| Family | Existing anchors | Required diversity in the expanded panel |
| --- | --- | --- |
| G1: Fourier transforms | B3 `qft_N100`; `time_qft_16` in [`transpiler_qualitative.py`](../test/benchmarks/transpiler_qualitative.py) | Several widths; freeze exact QFT conventions and final-swap options |
| G2: Hamiltonian simulation | B3 `square_heisenberg_N100` | Multiple lattice sizes, evolution depths and nontrivial numeric angles |
| G3: QAOA | `time_qaoa`/`track_qaoa_depth` in [`utility_scale.py`](../test/benchmarks/utility_scale.py) | Multiple graph instances, graph structures, widths and repetition counts |
| G4: Random dense interactions | QV in B2 and `utility_scale.py` | Multiple widths, depths and independently generated, frozen matrices |
| G5: Reversible logic | `hwb12` in `utility_scale.py`; `4gt10`, `4mod5`, `mod8`, and `cnt3` fixtures in `transpiler_qualitative.py` | Different Boolean/arithmetic circuits, including new larger instances; do not count each file as a separate family |
| G6: Bernstein–Vazirani patterns | `time_bv_100`, `time_bvlike` and depth companions in `utility_scale.py` | Several widths and interaction patterns; include simplification-sensitive instances |
| G7: Variational ansatz circuits | `time_circSU2`, `time_circSU2_89` and depth companions in `utility_scale.py` | Several widths, entanglement patterns and repetition counts; numeric and symbolic inputs reported separately |
| G8: Routing challenge circuits | BNTF, BSS and BIGD in [`queko.py`](../test/benchmarks/queko.py) | Independent instances and sizes with frozen input/target pairs; additional fixtures required for independent validation |

Freeze a machine-readable **campaign manifest** before inspecting candidate results:

- **Inputs and splits.** For each family, include small (4–16), medium (17–64), and
  large (65–100 logical qubits) instances where the family supports them. In each
  supported family/size cell, require at least three distinct tuning inputs and three
  distinct validation inputs, counted by independent input group. These are initial
  coverage minima, not a power guarantee.
  A fixed 100-qubit file cannot fill other size cells. For deterministic families, vary
  widths within a band or structural parameters; new transpiler seeds and wire relabeling
  alone do not make new input instances. Record unsupported cells and reasons before
  measuring candidates. All eight families must occur in both splits, and each split
  must cover all three size bands; missing required coverage blocks the general claim.
- **Targets.** Cover heavy-hex, line and two-dimensional grid connectivity, plus an
  all-to-all control. Record exact edges, directionality, native instructions and target
  width. Include both fully occupied targets and targets with spare physical qubits.
  Use `cx`, `cz` and `ecr` variants and a supported asymmetric directed target. Require
  each family on at least two sparse topology classes unless its fixture contract fixes
  the target (for example a QUEKO reference pair); include all topology classes in both
  splits. Match target properties across revisions, not across unlike hardware models.
- **Levels and configuration.** Include optimization levels 0, 1, 2 and 3 for every
  supported input/target pair. Freeze method/plugin choices, search budgets,
  approximation settings, ancilla contracts and worker counts. Use exact synthesis for
  this campaign. A campaign restricted to level 2 or heavy-hex supports that narrower
  claim only. Unsupported combinations must be declared before candidate evaluation.
- **Weights and roles.** Assign family, size band, topology, native basis, optimization
  level, instance-group ID and tuning/validation split to every case. Fix scored,
  zero-baseline and canary roles using baseline data before tuning. Use the family-balanced
  score in [the metric policy, section 5.2](transpilation-metric.md#52-general-improvement-qualification).
  Enumerate actual case IDs, weights and exclusions; a list of intended families is not
  an executable manifest. Related basis, target, level and parameter variants share an
  instance-group ID for uncertainty estimation and split assignment.
- **Budget.** Start with the metric policy's 100 transpiler seeds per quality case;
  profile baseline runtime and memory before freezing the panel. Use a fixed smaller
  diagnostic subset during development. If the full qualification run is too costly,
  narrow the declared scope or redesign and freeze a new campaign before tuning; do not
  silently remove expensive or unfavorable cases from a finished comparison.

Reserve validation inputs by instance group, keeping near-duplicates and variants of
the same generated circuit or problem instance in one split. For G7, keep bindings of
one ansatz instance together. Freeze and hash QV matrices, not just wiring seeds. A
fresh transpiler seed tests search noise on an existing input; it does **not** test
generalization to unseen circuits. Inputs already used to guide optimization belong in
the tuning split. After an acceptance decision, retire the exposed validation panel
from future independent qualification, including after rejection, and use a predeclared reserve panel or a newly
frozen campaign. Preserve retired cases as regressions.

Measure final `D2` and `N2` from the same output, with legality/mapping checks, on every
supported case. B2, QUEKO and qualitative depth trackers currently report total depth;
add native two-qubit metrics rather than treating their values as `D2`. Keep zero-depth
and known-simplification cases as absolute-delta checks, and do not remove nonzero cases
merely because they fail to improve. QUEKO's named optimum is comparable only under its
validated reference model (section 3).

Measure compilation cost for every scored workload: end-to-end `transpile()` and
reusable `pm.run()` in separate panels, plus preset construction when affected. Keep
B1's overhead cases. Record peak RSS in fresh processes on a predeclared panel covering
every family and size band, including the largest cases. Dynamic-circuit behavior,
scheduling and batching remain separate regression panels; they cannot be represented
by a static `D2` score. Section 5.1 describes the decision and report requirements.

## 2. Metrics to record and how to interpret them

ASV `time_*` methods measure execution time; `track_*` methods return a scalar chosen
by the benchmark author. Neither naming convention implies a correctness assertion.
The selected existing cases provide timing and some depth metrics. The additional
metrics in this table are **proposed companions**, not already emitted by those cases.

| Metric | Definition and unit | Interpretation and limitations |
| --- | --- | --- |
| End-to-end compilation time | Wall time of `transpile(input, fixed_options)`; seconds/circuit | Includes preset creation and execution; excludes fixture construction and verification |
| Reusable manager time | Wall time of `pm.run(input)` with the manager built outside timing; seconds/circuit | Separates repeated compilation from preset setup; do not subtract it from a differently configured end-to-end case |
| Preset construction time — proposed | Wall time of `generate_preset_pass_manager(...)` with a prebuilt fixed target; seconds | Needed when changing option resolution or pipeline assembly |
| Native two-qubit count `N2` — proposed | Number of executable two-qubit gate instructions after final translation; gates | Count all native two-qubit types, not only `cx`; exclude barriers, delays, and control-flow container nodes |
| Total depth `D` | `output.depth()` for flat circuits; dependency layers | Includes the circuit's operation dependencies and directive behavior; not physical duration |
| Two-qubit depth `D2` | Depth filtered to native two-qubit gates; layers | Captures entangling serialization; compare identical basis and target definitions |
| SWAP count — proposed diagnostic | Number of routing-inserted SWAPs at a clearly identified stage boundary; swaps | Measure before SWAP decomposition. Final zero SWAPs can simply mean translation removed the SWAP instruction |
| Scheduled makespan `T` — proposed | `max(start_i + duration_i)` for scheduled operations; integer `dt` ticks or seconds with a recorded `dt` | Include delays and measurements consistently; total gate depth does not determine this value |
| Peak resident memory — proposed | Fresh-process peak resident set size during a fixed large compilation; bytes, with harness/setup baseline recorded | Includes native Rust allocations. Python allocation tracking alone is insufficient; use identical process boundaries on both revisions |
| Batch throughput — proposed | Number of circuits divided by elapsed compilation time; circuits/second | Only for an explicitly fixed batch and worker configuration; also report total batch latency |
| Failure/timeout count and rate | Number of failed cases, failed/attempted fraction, and case identities | Must remain zero for supported cases. Skips and missing results are not successes |

For flat standard-basis outputs, `output.depth(filter_function=lambda inst:
inst.operation.num_qubits == 2 and inst.operation.name in native_2q_names)` is a
possible `D2` definition. Persist `native_2q_names` with the target. For dynamic circuits,
report branch-local counts/depth separately or define an explicit bounded execution
path: counting a loop body once is not an execution-cost estimate for the loop.

For each lower-is-better metric `m`, compare `ratio = m_candidate / m_baseline` and
`change_percent = 100 * (ratio - 1)`. When the baseline is zero, report the absolute
delta instead. Report per-case results before any aggregate; a geometric mean of
positive ratios can summarize a suite but must not hide one pathological slowdown.

## 3. Stage-specific selection and coverage gaps

These are additions to B1–B3 **only when relevant to a change**. Do not automatically
run every microbenchmark in the listed files. Timings diagnose cost; tests in section 4
establish behavior. New cases should also have small correctness counterparts.

| Flow boundary | Existing subset to reuse | What it measures; required companion coverage |
| --- | --- | --- |
| Circuit ↔ DAG | [`converters.py`](../test/benchmarks/converters.py): `ConverterBenchmarks.time_circuit_to_dag`, `time_dag_to_circuit` | Seconds per conversion over width/depth parameters. Use small and medium supported combinations first; add metadata/parameter/control-flow round-trip assertions. Exclude `time_circuit_to_instruction` |
| Init and synthesis | [`passes.py`](../test/benchmarks/passes.py): `PassBenchmarks.time_unroll_3q_or_more`; B2/B3 for integration | Seconds for lowering larger operations. **Proposed:** isolated `HighLevelSynthesis` and `UnitarySynthesis` workloads with controlled input size, native gate counts, and operator/state checks. Unrolling alone does not cover synthesis |
| Layout | [`mapping_passes.py`](../test/benchmarks/mapping_passes.py): `PassBenchmarks.time_sabre_layout`, `time_apply_layout`, `time_full_ancilla_allocation`, `time_enlarge_with_ancilla`; [`vf2.py`](../test/benchmarks/vf2.py): `VF2LayoutSuite.time_heavy_hex_trivial`, `time_heavy_hex_impossible` | Seconds for search, successful/no-match paths, and physical embedding. SABRE layout can also route, so its time is not pure assignment cost. Check valid initial/final mappings; add target-error-sensitive `VF2PostLayout` coverage when changing final layout improvement |
| Routing | [`mapping_passes.py`](../test/benchmarks/mapping_passes.py): `PassBenchmarks.time_sabre_swap`, `time_check_map`; [`queko.py`](../test/benchmarks/queko.py): `QUEKOTranspilerBench.time_transpile_bigd`, `track_depth_bigd_optimal_depth_45` | Isolated routing seconds plus a fixed end-to-end routing-quality case, across QUEKO's levels and default/SABRE options. Add pretranslation SWAP count and final `N2`/`D2`; compare semantic behavior after the final permutation |
| Translation and direction | [`passes.py`](../test/benchmarks/passes.py): `MultipleBasisPassBenchmarks.time_basis_translator`; [`mapping_passes.py`](../test/benchmarks/mapping_passes.py): `RoutedPassBenchmarks.time_gate_direction`, `time_check_gate_direction` | Seconds across gate vocabularies and directional checks. **Proposed:** asymmetric directed target, per-qubit instruction restrictions, and angle-bound cases. The existing symmetric mapping fixture does not demonstrate a costly direction repair |
| Optimization and stopping | [`passes.py`](../test/benchmarks/passes.py): `MultipleBasisPassBenchmarks.time_optimize_1q_decompose`, `CommutativeAnalysisPassBenchmarks.time_commutative_cancellation`, `PassBenchmarks.time_cx_cancellation`, `time_depth_pass`, `time_size_pass` | Seconds for simplification and loop analyses. **Proposed:** direct `TwoQubitPeepholeOptimization` workload; output `N2`/`D2`, loop iteration count, bounded completion, and retranslation after a rewrite. These older microbenchmarks do not represent the entire current preset |
| Scheduling | [`transpiler_levels.py`](../test/benchmarks/transpiler_levels.py): `TranspilerLevelBenchmarks.time_schedule_qv_14_x_14` (levels **0/1 only**); [`scheduling_passes.py`](../test/benchmarks/scheduling_passes.py): `SchedulingPassBenchmarks.time_time_unit_conversion_pass` | Complete ALAP compilation seconds and isolated duration normalization. **Proposed:** ASAP/ALAP analysis + `PadDelay` on one fixed native circuit, alignment/rescheduling, makespan, and timing-validity checks |
| Wrapper, reuse, batches | B1, B2 and B3; optionally [`randomized_benchmarking.py`](../test/benchmarks/randomized_benchmarking.py): `RandomizedBenchmarkingBenchmark.time_ibmq_backend_transpile`, `time_ibmq_backend_transpile_single_thread` | Total batch compilation seconds for lists of small RB circuits. This is compiler timing on RB-shaped inputs, not device randomized benchmarking. **Proposed:** explicit worker-count sweep, throughput, list order, input immutability, repeated manager use, and serial/parallel semantic equivalence |

Important fixture limitations:

- The QUEKO names encode reference depths (25, 100, or 45), but the current trackers
  return **raw final depth**, not a normalized optimality ratio. Its timing methods
  also call `.depth()`. Do not claim a proof of optimal native-gate depth or divide by
  the named reference without checking that basis, allowed rewrites, and depth model match.
- The ASAP/ALAP methods in `scheduling_passes.py` include
  `PadDynamicalDecoupling`, not the standard preset's `PadDelay`. Exclude those two
  methods from the standard scheduling subset unless testing that extension. Setup's
  preliminary `transpile()` also omits `seed_transpiler`; freeze that result or add a
  seed before using these fixtures for reproducible scheduler comparisons.
- Several older pass/converter fixtures call the local `random_circuit(...,
  conditional=True)` helper. That helper assigns the legacy `op.condition` attribute;
  it is not evidence of coverage of modern structured `if_else`/loop blocks. Use
  explicit structured-control-flow correctness fixtures.
- Some pass benchmarks operate directly on a stored mutable DAG. Before trusting
  repeated timings, verify each invocation sees equivalent input. If necessary use
  a fresh DAG per invocation and separately account for copying; otherwise later
  iterations may time an already simplified circuit. In the mapping fixture, setup
  itself already completes ancilla allocation/enlargement on shared objects, so those
  timed methods exercise the completed-work paths; see section 7.9 for the needed correction.
- The RB method named `single_thread` sets `QISKIT_IN_PARALLEL=TRUE` to suppress
  nested process parallelism; it does not independently pin Rust worker threads.
  Under the serial environment below, both RB methods can run serially, so their
  names alone do not establish a serial-versus-parallel comparison. Use explicit
  process/thread settings and record the effective worker counts for batch studies.
- For changes to SABRE scaling, the large mapping cases in
  [`qft.py`](../test/benchmarks/qft.py) and
  [`quantum_volume.py`](../test/benchmarks/quantum_volume.py) are useful extensions.
  Their track-suite `setup_cache()` can compute the full parameter grid, so a narrow
  selection may still have substantial setup cost. They are not core smoke tests.

## 4. Correctness gates: how to know a change did not break behavior

These are regression **tests**, even when run beside benchmarks. A correctness failure
blocks the change regardless of a speedup. Validate the baseline too: agreement with
an already incorrect baseline is not a correctness oracle.

### C1. Exact, small unitary circuits

Use deterministic 1–6-qubit circuits containing cancellations, noncommuting gates,
three-qubit gates, matrix-defined operations, and interactions requiring routing.
Cover levels 0–3; include unrestricted connectivity, a directed line, a nontrivial
initial layout, and a routing permutation. Keep target width equal to logical width
for the simplest operator checks.

Set `approximation_degree=1.0` and `qubits_initially_zero=False` for an all-input unitary
contract. Compare `Operator(original).equiv(Operator.from_circuit(compiled),
rtol=1e-7, atol=1e-8)`, as explained in the flow document. The metric is the number of
failed equivalence assertions; require zero. `Operator.from_circuit` accounts for
recorded layouts; plain `Operator(compiled)` compares physical wire order.

A full operator has `4**n` entries, so use this as a small test, not as part of large
benchmark timing. Global phase is ignored by `equiv`; transformations later used
under coherent control also need tests of phase bookkeeping and controlled composition,
where a formerly global phase can become relative.

Existing anchors: [`test_transpiler.py`](../test/python/compiler/test_transpiler.py),
[`test_basis_translator.py`](../test/python/transpiler/test_basis_translator.py),
[`test_unitary_synthesis.py`](../test/python/transpiler/test_unitary_synthesis.py), and
[`test_high_level_synthesis.py`](../test/python/transpiler/test_high_level_synthesis.py).

### C2. Layout, ancillas, and observable/measurement interpretation

Check nonidentity initial layout, routing permutation, idle wires, and a target wider
than the input. Verify original logical qubits map to distinct valid output positions
using `output.layout.final_index_layout()` where a layout exists. Verify measurements
still write to the intended classical bits and logical observables give the same
expectations after `observable.apply_layout(output.layout)`.

When extra ancillas are present, initialize them as required by the synthesis contract,
reorder the resulting state into logical/ancilla order, and compare the logical reduced
state. Check clean ancillas are restored only where the operation promises that.
Do not compare different-sized operators or assume `Operator.from_circuit` removes
ancillas. For zero-initialized synthesis, test the valid zero-state contract separately
from the arbitrary-input contract. Selected-state tests are weaker than all-input checks.

**Metrics:** invalid mapping/bit-association count (must be zero), state infidelity or
expectation residual against a fixture-specific numerical tolerance. Existing anchor:
[`test_transpile_layout.py`](../test/python/transpiler/test_transpile_layout.py).

### C3. Target legality and supported/unsupported configurations

Inspect every executable instruction against the **actual target**, including ordered
physical qubits, supported parameters, and angle bounds where defined. Recursively
validate control-flow blocks with their block-local wires mapped to the outer physical
wires; treat permitted directives separately. A basis-name list or undirected coupling
check alone is insufficient. Require zero illegal instructions/edges/parameters.

Include target/backend/separate-constraint input forms, native CX/CZ/ECR targets, empty
circuits, missing-duration errors, and unsupported configurations. Current presets
reject built-in ASAP/ALAP scheduling with control flow and reject basic/lookahead
routing with control flow; assert those failures rather than counting them as successful
compilations. Verify useful exception behavior when it is part of the API contract.

Existing anchors: [`test_target.py`](../test/python/transpiler/test_target.py),
[`test_gate_direction.py`](../test/python/transpiler/test_gate_direction.py), and
[`test_preset_passmanagers.py`](../test/python/transpiler/test_preset_passmanagers.py).

### C4. Measurements, resets, control flow, and parameters

Use small explicit examples with mid-circuit measurement, reset, both outcomes of an
`if_else`, and bounded loops on a supporting target. Compare exact classical outcome
probabilities and conditional remaining states where feasible. For distributions use
`TVD = 0.5 * sum(abs(p_original[x] - p_compiled[x]))`; deterministic exact-oracle
fixtures should meet a documented numerical tolerance (start with `1e-8` for small cases).

If using finite shots, define a statistical acceptance rule from shot count, number of
outcomes, and a chosen false-positive rate. Do not apply the exact-oracle tolerance to
sampled counts. Do not remove measurements and compare unitaries: measurement-aware
optimizations may intentionally change those unitaries while preserving outcomes.

The existing [randomized equivalence test](../test/randomized/test_transpiler_equivalence.py)
uses Aer, 4,096 shots and a per-outcome absolute-count tolerance of `0.05 * shots`.
It is useful broader coverage, not a universal error bound; it does not generate modern
structured control-flow blocks. Preserve failing inputs, options, and random seeds so a
failure becomes a deterministic regression case.

For symbolic parameters, verify retained parameters preserve their identity and meaning
and no unexpected free parameters are introduced. A valid optimization can eliminate
a parameter whose effect cancels or is unobservable, so identical parameter sets are
not a universal requirement. Bind the same input-parameter mapping to both circuits
(filter it to retained output parameters, or use `assign_parameters(..., strict=False)`
on the output), including zero, boundary angles, and generic nonzero values, then apply
C1/C2/C4 as appropriate. A few bindings provide evidence, not a symbolic proof. If deliberately
approximate synthesis is changed, use a separate declared error budget, for example
unitary process infidelity `1 - abs(trace(U†V))**2 / d**2` after logical alignment for
same-dimensional unitary outputs. `approximation_degree` itself is not that error bound.

### C5. Schedule validity and pass-manager contracts

For static scheduled fixtures, require zero negative start times, resource overlaps,
dependency violations, or applicable alignment violations. Use the target's durations,
`dt`, pulse/acquire alignment, and duration constraints; check inserted delays and classical
dependencies. Record makespan alongside validity. Equal makespan does not prove validity,
and ASAP versus ALAP can move operations while retaining the same makespan.

For pipeline orchestration, check that input circuits remain unchanged, a reused manager
does not leak a previous run's `PropertySet`, list inputs preserve order and cardinality,
and output mappings/timing metadata belong to the right circuit. Ensure optimization
loops terminate and final output returns to the target basis after rewrites. Do not
require byte-identical output circuits when a different valid heuristic result is allowed.

Existing anchors: [`test_constrained_reschedule.py`](../test/python/transpiler/test_constrained_reschedule.py),
[`test_scheduling_padding_pass.py`](../test/python/transpiler/test_scheduling_padding_pass.py),
[`test_passmanager_run.py`](../test/python/transpiler/test_passmanager_run.py), and
[`test_staged_passmanager.py`](../test/python/transpiler/test_staged_passmanager.py).

## 5. Reproducible comparisons and acceptance policy

Compare a recorded known-good commit with the candidate using identical fixtures,
options, dependencies, compiler/build profile, and hardware. In particular:

1. Freeze circuit data/generation seeds (including the unseeded QV matrices identified
   in section 7.3), target connectivity/instructions/durations/errors,
   optimization level, methods/plugins, initial layout, approximation settings, and
   `qubits_initially_zero`. Use synthetic seeded targets, not changing live calibrations.
2. Record Python/Qiskit/NumPy and Rust versions, OS/CPU, available cores, allocator/build
   settings, and process/thread settings. For a serial reference run set
   `QISKIT_PARALLEL=FALSE`, `QISKIT_IGNORE_USER_SETTINGS=TRUE`, `RAYON_NUM_THREADS=1`,
   and `OMP_NUM_THREADS=1`. Run a separate fixed-worker configuration for parallel changes.
3. Freeze the **search budget** as well as the random seed. Direct SABRE pass defaults
   can use available CPU counts for trial counts; a thread limit does not necessarily
   pin the number of trials. Set `trials`, `layout_trials`, and `swap_trials` explicitly
   in direct pass fixtures, and record the resolved preset search settings in pipeline
   comparisons. The built-in presets normally supply fixed trial counts, but
   `QISKIT_SABRE_ALL_THREADS` enables CPU-dependent counts when set, even to `FALSE`.
   Unset it for the reference configuration; ignoring user settings also prevents the
   equivalent user-configuration option from changing the budget.
4. Generate/parse fixtures outside timing, warm the environment, and let ASV collect
   repeated measurements. Verify outputs separately. For diagnosis, collect pass times
   and invocation counts with a callback in a separate run, since instrumentation adds
   overhead. Attribute SABRE routing done inside layout to that executed pass.
5. Compare the existing fixed seeds first. For heuristic changes, add the same proposed
   seed panel `{0, 1, ..., 9}` to both revisions and report median, worst case, and
   per-seed quality changes; one lucky seed is not evidence of a general improvement.
   This is diagnostic coverage, not a guarantee of statistical power for small gains.
   For a `D2` optimization campaign, use the larger panel, covariance-aware uncertainty,
   frozen-baseline guards, and fresh confirmation blocks specified in
   [the optimization objective](transpilation-metric.md). That policy also adds QAOA
   timing for the focused panel and cost companions for every general-panel workload.
   A general claim additionally requires section 1.1's input diversity and independent
   validation; seed replication alone cannot supply that evidence.

Suggested initial policy, to tune using baseline-versus-baseline noise measurements:

| Signal | Proposed decision |
| --- | --- |
| Semantic, legality, mapping, schedule, API-contract failure; new crash/timeout | Block immediately and reduce to a reproducer |
| Repeated timing or peak-memory increase over 10% | Flag for investigation; require reproduction on the controlled runner before calling it a regression. Choose an absolute noise floor per case so tiny timings are not judged by ratios alone |
| Fixed-fixture `N2`, `D2`, or makespan increase | Report every delta. Investigate increases over 5%, any unexpected loss of a known exact simplification, and poor worst-seed results; smaller values do not establish correctness |
| Intended speed/quality tradeoff | Require explicit review of both metrics and affected workloads; never silently replace the baseline to make the result green |
| Missing/skipped cases | Explain each one; do not use an incomplete comparison as proof the suite passed |

Keep raw samples, per-case output metrics, failures, input/target identifiers and both
commit IDs. A useful report row is:
`case, input_hash, target_hash, options, seed, revision, seconds, N2, D, D2, makespan,
peak_RSS, correctness_status`. Leave inapplicable metrics absent rather than zero.

### 5.1. Evidence required for a general-improvement claim

Use the full section 1.1 manifest and [metric policy section 5.2](transpilation-metric.md#52-general-improvement-qualification),
in addition to C1–C5 and the affected repository tests. The general tier requires:

1. A family-balanced `D2` improvement against both the current accepted best and the
   frozen campaign baseline on the tuning panel and on independent validation inputs
   with fresh transpiler seeds. The metric policy specifies practical effect size,
   uncertainty and breadth requirements. Lower depth on three selected circuits is a
   focused-panel result even if all other benchmarks merely avoid regression.
2. Per-family, size-band, topology, native-basis and level results, plus per-case
   `D2`/`N2` guards. Show how many families improve and whether the aggregate still
   improves after removing the strongest family. Report both transpiler-seed uncertainty
   and uncertainty across independent circuit instances; do not count correlated target
   variants as new independent inputs.
3. Compilation-time and peak-memory guards, correctness on scored outputs, and zero
   unexpected failures/timeouts. No missing measurement can be silently removed from
   the score. A failed coverage or breadth requirement limits the claim even if the
   focused panel passes. Intended trades follow the explicit review policy and cannot
   be reported as passing all general-tier constraints.

Archive the manifest and harness revision, input/target hashes, both reference commits,
candidate commit, split and seed-block IDs, raw measurements and failure records. Extend
each report row with `family, size_band, topology, native_basis, optimization_level,
instance_group, split, weight, timing_mode`. Publish aggregate effects and uncertainty,
all marginal summaries, per-case ratios, absolute zero-baseline deltas, worst-seed
outcomes, timing and memory results, coverage/exclusions and all acceptance attempts.
The final claim must name the metric and scope, for example “native two-qubit depth
improved across the declared static-circuit families, sizes, targets and levels, within
the stated count/time/memory limits.” Do not infer faster hardware execution or improvement
on unseen families from this result.

## 6. Practical workflow when evolving the code

### Run correctness tests first

From the repository root, use an installed supported Python version; these commands
use the existing tox/stestr configuration in [`tox.ini`](../tox.ini):

```sh
# Targeted transpilation tests (regex includes compiler and pass-manager tests).
tox -e py312 -- 'test.python.(compiler|transpiler)'

# Before acceptance, run the broader Python suite for shared code changes.
tox -e py312

# For Rust transpiler changes, start with its crate, then follow repository guidance.
cargo test -p qiskit-transpiler
```

Add tests for the changed Rust crate, Python/native boundary and C API where affected;
follow [CONTRIBUTING.md](../CONTRIBUTING.md) for the broader required checks. The
transpilation subset is not a replacement for repository-wide contribution requirements.
Rebuild the native extension before testing Python against Rust edits.

In a development environment with Qiskit and the `test-random` dependencies from
[`pyproject.toml`](../pyproject.toml) installed, run the specific randomized oracle:

```sh
python -m unittest test.randomized.test_transpiler_equivalence
```

The normal stestr configuration discovers `test/python`, so running that alone does
not run this randomized module. Keep its simulation time out of compilation measurements.

### Run the existing core ASV subset

ASV is a separate prerequisite; install a compatible, pinned ASV version in your
benchmark environment. The repository's [`asv.conf.json`](../asv.conf.json) points at
`test/benchmarks`, builds isolated wheels with mimalloc, and lists multiple Python
versions. Select one installed version for an initial comparison. Build prerequisites
and package access must be available. See the [ASV command reference](https://asv.readthedocs.io/en/stable/commands.html)
for the `run` and `continuous` options used below.

The following regex selects exactly the existing B1–B3 methods, retaining their
current parameter grids. Replace `BASELINE_COMMIT` and `CANDIDATE_COMMIT` with actual
committed revision IDs; ASV's revision comparison does not include uncommitted edits.

```sh
export QISKIT_PARALLEL=FALSE
export QISKIT_IGNORE_USER_SETTINGS=TRUE
export RAYON_NUM_THREADS=1
export OMP_NUM_THREADS=1
unset QISKIT_SABRE_ALL_THREADS

# ASV matches parameterized benchmarks as `name(param)`, so allow an optional suffix;
# a bare `$` silently drops every B2 and B3 case.
TRANSPILE_CORE='^(transpiler_benchmarks\.TranspilerBenchSuite\.time_(single_gate|cx)_compile|transpiler_levels\.TranspilerLevelBenchmarks\.(time_transpile_qv_14_x_14|track_depth_transpile_qv_14_x_14|time_transpile_from_large_qasm|track_depth_transpile_from_large_qasm)|utility_scale\.UtilityScaleBenchmarks\.(time_qft|track_qft_depth|time_square_heisenberg|track_square_heisenberg_depth))(\(.*\))?$'

# Smoke-check benchmark execution; quick results are not performance evidence.
asv run --quick --python=3.12 --bench "$TRANSPILE_CORE"

# Measure both revisions on the same machine and inspect individual results.
asv continuous --environment=virtualenv:3.12 --no-only-changed --bench "$TRANSPILE_CORE" BASELINE_COMMIT CANDIDATE_COMMIT
```

Confirm selection and parameter coverage in ASV's output; abort on discovery/setup
errors. The quick command uses ASV's configured revision selection, not arbitrary
working-tree edits. The continuous comparison covers only the existing timing/depth
methods: it cannot enforce the proposed semantic, gate-count, memory, and schedule
checks until their harnesses exist. Do not treat its summary as the complete gate.

In particular, the selected B2 QV fixture still generates unseeded matrices (section
7.3). These commands can smoke-check it, but its cross-revision results are not a
controlled comparison until both revisions use the same frozen matrices or the same
deterministically seeded fixture correction. Record the benchmark-harness revision
separately from the Qiskit revisions; do not silently compare different fixture inputs.

### Investigate, then accept

For every behavioral fix, first add a regression test that fails on the old bug and
passes on the corrected implementation. Run the core and affected stage subset. If
claiming general improvement, also complete the section 1.1 panel and section 5.1
qualification after freezing the candidate. The ASV core commands above do not run or
enforce this proposed qualification harness. If correctness fails, reduce the circuit
and inspect mappings/targets before interpreting
performance. If timing or quality regresses, compare per-case outputs and separately
instrument pass execution to locate the cause. Repeat only the affected comparisons
when noise or a subsequent fix warrants it.

Accept the change when required correctness checks pass, the selected benchmark cases
complete, and any material time/quality/memory tradeoff is understood and documented.
Preserve the known-good results for the next change. Implement missing coverage in this
order: correctness fixtures for the changed contract; `N2`/`D2` companions; stage-specific
missing benchmarks; then memory, multi-seed and batch extensions as those paths evolve.
For a general-improvement campaign, the expanded fixtures/manifest, family-balanced
scorer, independent validation and cost guards are required before making that claim;
they are not deferred optional extensions.

## 7. Detailed walkthrough: what each selected benchmark actually does

Read this section when a benchmark name does not explain the experiment. A **fixture**
is the circuit, graph, target, or pass manager prepared before measurement. A **native
basis** is the set of gate types the target machine accepts. A graph **layer** groups
operations that can occur without waiting for each other. Depth counts dependency
layers; physical duration requires a separate calculation using operation durations
and a valid schedule.

For each ASV case, setup prepares its fixtures outside the timed method. ASV repeatedly
calls a `time_*` method and records elapsed time. A `track_*` method instead returns a
number such as circuit depth. In most pairs below, the tracker runs compilation again;
it does not read the output of the timing method. This matters when input generation
or compilation is nondeterministic. None of these methods runs a quantum computer.

### 7.1. One H gate: `time_single_gate_compile` (B1)

**Input and preparation.** `TranspilerBenchSuite` creates a one-qubit circuit containing
just `H(0)`. H is a Hadamard gate: on a zero input it creates an equal superposition of
zero and one. The requested basis is `id, rz, sx, x, cx, reset`, which does not contain H.
The fixture also supplies the fixed 27-qubit connectivity graph.

**Inside the timer.** It calls `transpile(single_gate_circuit, coupling_map=...,
basis_gates=..., seed_transpiler=20220125)`. Qiskit resolves options, creates the preset,
converts the circuit to a DAG, assigns physical wires as needed, and expresses the H
using supported gates. It then constructs the output circuit and metadata. The timer
includes this entire call; it excludes building the input and coupling-map list.

**Result and interpretation.** ASV records seconds per call. The method discards the
output and does not count its gates. Because there is almost no circuit work, a large
increase usually points toward setup, target processing, graph conversion, or other
fixed overhead. The output need not literally contain H, and its exact decomposition
can change without breaking the computation.

### 7.2. An identity written as six gates: `time_cx_compile` (B1)

**Input and preparation.** The same suite constructs this two-qubit circuit:

```text
H(0), H(0), CX(0,1), CX(0,1), CX(0,1), CX(0,1)
```

H followed by H is identity; two identical consecutive CX gates are also identity.
Consequently the whole circuit can be removed without changing its unitary action.
The hardware description, basis, and seed are the same as in 7.1.

**Inside the timer.** It calls the complete `transpile()` pipeline with the default
optimization level. This is not a direct call to the cancellation pass: pipeline
construction, mapping, translation, optimization, and output conversion all contribute.

**Result and interpretation.** The sole recorded metric is seconds. This workload
checks whether the compiler remains inexpensive when the answer is very simple.
A missing cancellation could leave extra gates without causing a timing failure;
use a separate output-quality assertion to detect that regression. Do not infer that
the benchmark proves the output is empty merely because it completes successfully.

### 7.3. Random two-qubit matrices on 14 qubits: the QV pair (B2)

**Input and preparation.** `TranspilerLevelBenchmarks` calls
`build_qv_model_circuit(14, 14, 0)`. For each of 14 rounds, the helper permutes the
14 qubit indices, splits them into seven disjoint pairs, and puts a random 4×4 unitary
matrix on each pair. There are **98 matrix-defined two-qubit operations** before
compilation. A 4×4 matrix specifies the action on the four joint basis states of two
qubits; it has not yet been decomposed into the machine's native gates. Different
rounds use different pairings, so interactions are not confined to nearby wires.

Setup also builds a seeded 14-qubit `GenericBackendV2` with Melbourne connectivity.
For each optimization level 0–3, the two methods perform separate experiments:

| Method | Work performed | Recorded result |
| --- | --- | --- |
| `time_transpile_qv_14_x_14` | Call `transpile(qv, backend, optimization_level=level, seed_transpiler=0)` | Seconds for the entire compilation |
| `track_depth_transpile_qv_14_x_14` | Make the same compilation call, then call `.depth()` on its output | Integer total output depth |

**What stresses the compiler.** It must synthesize each matrix into gates, choose where
logical qubits live, make the changing pairs reachable on sparse hardware, translate,
and simplify. This combines synthesis and routing pressure; it is not an isolated
routing test. “14×14” describes construction rounds, not a promise that the compiled
circuit has depth 14. The benchmark does not measure a device's quantum-volume score.

**Reproducibility limitation found in the current helper.** Its `np.random.seed(0)`
seeds the permutations, but `random_unitary(4)` receives no seed. In this checkout,
[`random_unitary`](../qiskit/quantum_info/operators/random.py) uses a separate
`DEFAULT_RNG`, so the matrices are not fixed by that call. Before making reproducible
cross-revision timing or depth comparisons, freeze the generated input circuit or
change the fixture to pass an explicit seeded generator to every random-unitary call.
That fixture correction is proposed here, not implemented by this document. It also
applies to the 50×20 QV extension and QV-based scheduling/large-mapping cases.

### 7.4. A long two-qubit sequence: the “large QASM” pair (B2)

**Input and preparation.** Setup loads
[`test_eoh_qasm.qasm`](../test/benchmarks/qasm/test_eoh_qasm.qasm). Despite “large” in
the name, it has only **two logical qubits**. Its 3,505 instructions are 1,002 CX,
900 `u1`, 1,200 `u2`, and 403 `u3` gates. All interactions are between those same two
qubits. These counts describe the checked-in input, not a measured compiled result.

The selected methods supply a 53-qubit Rochester coupling graph and the explicit basis
`u1, u2, u3, cx, id`, rather than a backend object. They vary optimization level 0–3
and use transpiler seed 0.

| Method | Work performed | Recorded result |
| --- | --- | --- |
| `time_transpile_from_large_qasm` | Compile the already loaded circuit with the explicit constraints | Seconds per compilation; file parsing is excluded |
| `track_depth_transpile_from_large_qasm` | Compile it independently and return `.depth()` | Total output dependency layers |

**What stresses the compiler.** Long sequences on a very small active qubit set expose
repeated single-qubit simplification, cancellation, two-qubit block optimization, and
optimization-loop overhead. The larger target also exercises embedding and target
processing. This is not a wide many-qubit routing workload; it complements 7.3.
A depth reduction accompanied by longer compilation can be a deliberate optimization
tradeoff, while increased time with unchanged quality deserves investigation.

### 7.5. A 100-qubit Fourier-transform circuit: the QFT pair (B3)

**Input and preparation.** Setup loads
[`qft_N100.qasm`](../test/benchmarks/qasm/qft_N100.qasm). A quantum Fourier transform
changes from the computational basis to a basis whose states encode different phase
patterns. This fixture already expresses its transform as rotations and CX gates:
100 `ry`, 100 `rx`, 14,850 `rz`, and 10,050 CX instructions, **25,100 gates in total**.
The CX interactions touch all **4,950 unordered pairs** of its 100 qubits.

The target instead has sparse heavy-hex connectivity, with 193 physical qubits from
`CouplingMap.from_heavy_hex(9)`. Setup builds the level-2 pass manager once. The three
cases change the allowed entangling gate to CX, CZ, or ECR while retaining
`rz, x, sx, id` and the backend's other supported operations.

| Method | Work performed | Recorded result |
| --- | --- | --- |
| `time_qft` | Run the prepared manager on the loaded QFT circuit | Seconds for `pm.run`; preset creation and parsing excluded |
| `track_qft_depth` | Compile independently, then filter `.depth()` to the chosen `cx`, `cz`, or `ecr` instruction name | Native entangling-gate layers |

**What stresses the compiler.** The circuit asks many distant qubits to interact, so
layout and routing must handle dense logical connectivity on a sparse machine.
Translation must express the original CX-based input in the selected target basis.
Changes can affect both compile time and the number of sequential entangling operations.

For illustration, suppose the CZ case changes from 800 to 880 filtered layers:
that is a 10% increase in CZ depth, not 80 extra nanoseconds and not necessarily
80 extra CZ gates. These numbers illustrate interpretation; they are not benchmark
results. Separate comparisons are needed for the CX and ECR cases.

### 7.6. A 100-qubit square-lattice interaction circuit: the Heisenberg pair (B3)

**Input and preparation.** Setup loads
[`square_heisenberg_N100.qasm`](../test/benchmarks/qasm/square_heisenberg_N100.qasm).
It is a circuit for simulated spin interactions on a 10×10 square grid. The checked-in
input has **7,660 gates**: 2,160 CX, 1,180 `rz`, 2,880 `rx`, and 1,440 `ry` operations.
Its two-qubit interactions use the grid's 180 nearest-neighbor pairs, unlike QFT's
all-pairs interaction pattern. Rotations before and after CX/phase sequences implement
different interaction components.

**Execution and metrics.** `time_square_heisenberg` calls
`self.pm.run(self.square_heisenberg_qc)` and records seconds.
`track_square_heisenberg_depth` makes a separate run and returns depth filtered to
the selected native entangler. Both use exactly the target/preset setup described in
7.5, for each of CX, CZ, and ECR.

**What stresses the compiler.** The logical square grid must fit a different sparse
hardware graph. Repeated local gate patterns also give simplification passes work to
do. It checks whether a change that helps globally connected QFT circuits hurts a
structured local-interaction circuit. It compiles a simulation circuit; it does not
simulate the spin system or test accuracy of the physical model.

### 7.7. Circuit/graph conversion benchmarks

Source: [`converters.py`](../test/benchmarks/converters.py), `ConverterBenchmarks`.
Setup constructs a seeded random circuit and its DAG. Requested widths are
`1, 2, 5, 8, 14, 20, 32, 53`; requested generation depths are `8, 128, 2048, 8192`.
The fixture skips width ≥20 with depth ≥2048 and width 14 with depth >2048.
Measurements and legacy conditional attributes are included; see section 3's caveat.

- **`time_circuit_to_dag`:** turn the prepared circuit's instruction sequence into
  operation nodes and dependency edges. Record seconds for the conversion alone.
  This exposes graph allocation and dependency-building overhead at pipeline entry.
- **`time_dag_to_circuit`:** turn the prepared DAG back into an ordered circuit.
  Record seconds for that direction alone. This exposes graph traversal and output
  construction overhead at pipeline exit.

Neither executes optimization, placement, or routing, and neither asserts a successful
semantic round trip. Larger generation depth means more input construction rounds,
not necessarily an identical final DAG depth for every random circuit.

### 7.8. Decomposing gates larger than two qubits

Source: [`passes.py`](../test/benchmarks/passes.py),
`PassBenchmarks.time_unroll_3q_or_more`. Setup makes a seeded random circuit with
5, 14, or 20 qubits and 1,024 construction rounds, permitting up to three-qubit
operations, resets, measurements, and legacy conditional attributes; it converts the
circuit to a DAG before timing.

The timed method constructs `Unroll3qOrMore()` and runs it directly on that DAG.
For instance, a three-qubit controlled-controlled-X can expand into a sequence of
smaller operations through its definition. ASV records seconds for pass construction
and execution. This catches expensive recursive decomposition or graph replacement.
It does not place the resulting gates on hardware, guarantee the final native basis,
or isolate high-level/matrix synthesis; those need the proposed cases in 7.15.

### 7.9. Choosing and applying a layout

Source: [`mapping_passes.py`](../test/benchmarks/mapping_passes.py), `PassBenchmarks`.
The common fixture starts from 5-, 14-, or 20-qubit random circuits with 1,024 rounds,
seed 42, and at most two-qubit operations. A fixed 20-qubit coupling graph describes
the hardware. Setup computes a dense layout, allocates all idle hardware positions,
and enlarges the DAG in place. Consequently `fresh_dag`, `full_ancilla_dag`, and
`enlarge_dag` refer to the same already enlarged 20-wire DAG when timing begins;
the stored layout already includes every ancilla. This affects what the methods measure.
The four selected methods each time pass construction plus the indicated `.run()`:

- **`time_sabre_layout`:** run `SabreLayout(..., seed=42)` on that enlarged DAG, whose
  original 5, 14, or 20 qubits carry the random workload and remaining wires are idle.
  SABRE searches starting assignments using routing costs and, by default, also routes
  the circuit. Seconds therefore include that search/routing work, not just writing
  a map from logical qubit numbers to physical qubit numbers.
- **`time_full_ancilla_allocation`:** run allocation with the already full layout.
  It validates and scans the layout but finds no remaining hardware positions to
  allocate. The current timing therefore covers the already-allocated path; it does
  not measure creating new ancillas.
- **`time_enlarge_with_ancilla`:** compare the supplied layout's registers with the
  already enlarged DAG. It finds no missing registers, so the current timing covers
  the already-enlarged path; it does not measure adding wires.
- **`time_apply_layout`:** rewrite the prepared enlarged DAG onto physical wire indices
  using the supplied layout. Record seconds for relabeling/rebuilding the physical DAG.
  The placement choice has already been made outside this timed call.

A slowdown in `time_sabre_layout` suggests search/routing cost; a slowdown in the
other three suggests embedding or graph-management cost. These timing methods do
not report placement quality. To benchmark actual allocation and enlargement, add
corrected fixtures that retain independent pre-allocation layouts and pre-enlargement
DAGs and restore them for each invocation. See the mutable-fixture warning in section 3;
copying the current post-setup objects alone would preserve the already-completed work.

**VF2 search alternatives.** In [`vf2.py`](../test/benchmarks/vf2.py), cached setup
creates targets with seeded synthetic error estimates and matching circuit graphs:

- **`VF2LayoutSuite.time_heavy_hex_trivial`:** the circuit follows the same 57-qubit
  heavy-hex connectivity as the target, so a placement exists without inserting SWAPs.
  Time construction and execution of `VF2Layout(seed=-1, strict_direction=False,
  max_trials=0)`. This measures successful graph-matching/search work, not necessarily
  an immediate single-match return.
- **`VF2LayoutSuite.time_heavy_hex_impossible`:** ask to embed a 57-qubit line circuit
  into the 57-qubit heavy-hex target, a fixture intended to have no valid embedding.
  Time the search with `call_limit=None` and `call_limit=1_000_000`. These expose costly
  unsuccessful searches and the effect of limiting search effort. Timing alone does
  not distinguish exhausted search from reaching the configured limit; check the stop
  reason separately when testing behavior.

Both return elapsed seconds through ASV, not a fidelity estimate or number of SWAPs.
Target creation and cached circuit construction are outside their timed methods.

### 7.10. Routing and checking connectivity

In `mapping_passes.PassBenchmarks`, setup has already assigned physical wires:

- **`time_sabre_swap`:** construct `SabreSwap(coupling_map, seed=42)`, attach the
  prepared layout, and route the physical DAG. When a requested interaction is not
  on a hardware edge, routing can insert SWAPs to move the logical states. ASV records
  seconds; it does not record SWAP count or final depth.
- **`time_check_map`:** construct `CheckMap(coupling_map)` and inspect the prepared
  DAG for unsupported interactions. It records an analysis result in the pass's
  property set and ASV records execution seconds. The pass does not fix the circuit,
  and this benchmark does not assert that the circuit passes the check.

These distinguish the cost of making a route from the cost of checking connectivity.
Gate direction is a separate constraint, described in 7.11.

**QUEKO integration case.** `queko.QUEKOTranspilerBench` loads
`20QBT_45CYC_.0D1_.1D2_3.qasm`: 20 qubits and 45 CX instructions in this checkout.
It compiles against Tokyo connectivity and the `id, rz, sx, x, cx` basis with seed 0,
levels 0–3, and either default methods or explicit SABRE for both layout and routing.
`time_transpile_bigd` times `transpile(...).depth()`, including the final depth query.
`track_depth_bigd_optimal_depth_45` independently compiles and returns that depth.
This checks overall placement/routing/optimization quality on a fixed constructed
workload. The 45 in the name does not turn the returned raw depth into an optimality
ratio; cancellation and allowed rewrites also affect the result.

### 7.11. Basis translation and directional gates

**`passes.MultipleBasisPassBenchmarks.time_basis_translator`.** Setup builds a seeded
random DAG with 5, 14, or 20 qubits and 1,024 rounds. The basis parameter selects
`[u, cx, id]`, `[rx, ry, rz, r, rxx, id]`, or `[rz, x, sx, cx, id]`. Inside the timer,
`BasisTranslator(SessionEquivalenceLibrary, basis).run(dag)` finds and applies known
gate decompositions. For example, a gate missing from the destination vocabulary must
be replaced by an equivalent sequence made from that vocabulary. The metric is seconds
for constructing and running the translator, not the length of the replacement circuit.
No hardware connectivity is supplied to this benchmark.

**`mapping_passes.RoutedPassBenchmarks.time_check_gate_direction`.** Setup performs
layout and SABRE routing first. The timed method constructs `CheckGateDirection` and
checks the already routed DAG. For directional gates, `(physical_0, physical_1)` and
`(physical_1, physical_0)` can have different support. Record seconds for the check;
no repairs are performed and the property-set result is not asserted by the benchmark.

**`mapping_passes.RoutedPassBenchmarks.time_gate_direction`.** Use the same routed
fixture but run the transformation `GateDirection`. In general this pass repairs
unsupported orientations using equivalent gate sequences. This fixture's coupling
map is bidirectional, however, so it mostly exercises the traversal/no-repair path.
Its seconds cannot establish repair performance on one-way hardware. Add the proposed
asymmetric-target fixture when changing that behavior.

### 7.12. Simplification and optimization-loop analyses

The following methods in [`passes.py`](../test/benchmarks/passes.py) record **seconds**
for constructing and running a pass. None records how many gates it removed.

- **`MultipleBasisPassBenchmarks.time_optimize_1q_decompose`:** use the same width,
  depth, and three-basis grid as the translator case. Combine consecutive single-qubit
  gates into an equivalent operation and synthesize a suitable sequence in the chosen
  basis. This isolates local single-qubit simplification cost; it does not route gates.
- **`CommutativeAnalysisPassBenchmarks.time_commutative_cancellation`:** setup builds
  the random DAG and runs `CommutationAnalysis`, saving which operations commute.
  The timed method gives that property to a new `CommutativeCancellation` and runs it.
  Commuting operations can sometimes move past one another so inverse operations can
  cancel. However, the current cancellation implementation does not read the stored
  `commutation_set`: it invokes its own Rust commutation/cancellation routine.
  Setup's explicit analysis call is outside timing, while that internal work is
  included. A disappearance of valid cancellations needs an output-quality or
  correctness assertion to detect.
- **`PassBenchmarks.time_cx_cancellation`:** construct `InverseCancellation([CXGate()])`
  and run it on the prepared random DAG. It targets cancelable self-inverse CX pairs.
  This is the isolated pass counterpart to the full identity-circuit experiment in 7.2,
  but uses a much larger random circuit rather than that six-gate example.
- **`PassBenchmarks.time_depth_pass`:** construct `Depth()` and compute the DAG's depth
  into the property set. The benchmark returns no depth number; ASV measures how long
  the analysis took. It catches increased cost of an analysis used by stopping logic.
- **`PassBenchmarks.time_size_pass`:** similarly construct `Size()` and compute operation
  count into the property set. ASV records seconds, not circuit size. A fast size pass
  is relevant when the optimization loop invokes it repeatedly.

The latter four use the 5/14/20-qubit, 1,024-round random fixtures with resets,
measurements and legacy conditional attributes. They do not themselves benchmark
`FixedPoint`, `MinimumPoint`, or the whole optimization loop. B2 supplies that integration
coverage; a dedicated peephole/loop diagnostic is still proposed.

### 7.13. Scheduling and time-unit normalization

**`TranspilerLevelBenchmarks.time_schedule_qv_14_x_14`.** Build the same 98-operation
QV-style input and backend as in 7.3. Call the complete `transpile()` pipeline with
`scheduling_method="alap"`, seed 0, and optimization level 0 or 1. After native-gate
compilation, scheduling uses instruction durations to place operations as late as
possible within the schedule and the preset pads idle intervals. ASV records seconds
for the entire compile-and-schedule call, not only the scheduling stage and not the
resulting circuit duration. The unused `self.durations` object in this benchmark class
is not passed to this call; the backend supplies the relevant duration information.

**`SchedulingPassBenchmarks.time_time_unit_conversion_pass`.** Setup generates
5/10/20-qubit random circuits with 500/1,000 rounds, compiles them at level 1 onto
a fixed coupling map, and converts them to DAGs. It separately supplies durations
in `dt` ticks: RZ 0, ID/SX/X 160, CX 800, measure 3,200, reset 3,600; `dt=1e-9` seconds.
The timer covers `TimeUnitConversion(durations).run(dag)`, which establishes a common
time unit for instruction durations and delays. It assigns no ASAP/ALAP start schedule.
Record seconds for normalization, not a makespan. The preliminary compilation is
outside the timer and currently lacks a fixed transpiler seed.

### 7.14. Optional batch, parameter, and scale extensions

These explain the extensions already identified above; they do not expand the core filter.

**Batch RB methods.** In `randomized_benchmarking.RandomizedBenchmarkingBenchmark`,
setup uses seed 10, one- or two-qubit Clifford circuits, and lengths `1, 5, ..., 197`.
Crucially, the helper uses `circuits.extend(...)`: it appends each sampled Clifford
as a separate circuit. The resulting input is **4,950 small circuits**, not 50 long
concatenated sequences with inverse recovery gates. Both selected methods time one
`transpile(list_of_circuits, ...)` at level 0, with a fixed directed coupling map and
legacy basis. `time_ibmq_backend_transpile_single_thread` additionally sets
`QISKIT_IN_PARALLEL=TRUE`; see the thread/process caveat in section 3. The recorded
metric is seconds for the whole batch, and proposed throughput would be `4950 / seconds`.
This stresses per-circuit overhead and batch dispatch, not hardware error estimation.

**Larger QV and backend alternatives.** The `transpiler_levels` 50×20 timing/depth pair
uses 20 rounds of 25 random matrix gates (500 operations) and explicit Rochester/basis
constraints. It increases synthesis/routing work while keeping the same measurement
pattern as 7.3. The `*_from_large_qasm_backend_with_prop` pair instead reuses the long
two-qubit sequence but supplies the 14-qubit backend target; compare that case with
itself across revisions. It tests a different target/configuration path, not a clean
experiment changing only one field of the explicit-constraint case.

**Symbolic circuits.** `utility_scale.time_circSU2` and `track_circSU2_depth` construct
`efficient_su2(100, reps=3, entanglement="circular")`: parameterized rotation layers
interleaved with entangling layers around a ring. They leave parameters unbound and
perform no numerical parameter search. They run the B3 preset and record compilation
seconds or native entangling depth. Use them for preservation and efficient handling of symbolic expressions, with
separate numerical binding checks for correctness.

**QAOA circuits.** `utility_scale.time_qaoa` and `track_qaoa_depth` load the fixed
100-qubit, three-repetition Barabási–Albert graph fixture. It contains 2,264 gates,
including 1,176 CX gates on 196 distinct pairs, plus numeric rotations. They compile
with the B3 setup and record seconds or native entangling depth. This stresses routing
on an irregular logical graph; no variational parameter search or objective evaluation
occurs in the benchmark.

**Very large routing.** `LargeQFTMappingTimeBench.time_sabre_swap` uses 115, 409, or
1,081 qubits, with controlled-phase/H QFT inputs. `LargeQuantumVolumeMappingTimeBench`
uses matrix-gate QV inputs at `(width, rounds)` of `(115,10)`, `(115,100)`, `(409,10)`,
or `(1081,10)`. Both construct `SabreSwap` with seed `2022_10_27`, `trials=1`, and
`lookahead` or `decay` heuristic. Their timers include pass construction and `.run()`,
with input/target construction outside timing. Their corresponding
track classes precompute routed DAGs in `setup_cache()`; `track_depth_sabre_swap`
returns DAG depth and `track_size_sabre_swap` returns DAG operation count from the cache.
These are pre-native-translation metrics: an inserted SWAP counts as one DAG operation,
not as its eventual native-gate decomposition. Use them to diagnose routing scaling,
with the cache-cost and random-matrix caveats already described.

### 7.15. What the proposed missing benchmarks would do

The following are specifications for new measurements, not existing callable methods.
Each should use an unchanged prepared input per invocation, measure only the stated
boundary, and check its output outside timing.

| Proposed experiment | Concrete operation to benchmark | What it would reveal |
| --- | --- | --- |
| High-level synthesis | Prepare a small and a larger circuit containing a structured operation such as a multi-controlled X; run `HighLevelSynthesis` with a fixed plugin, target and ancilla assumptions | Seconds to lower that operation and resulting native gate cost after a fixed downstream translation; pair with a small semantic test |
| Matrix synthesis | Prepare fixed seeded 2- and 3-qubit matrices; run `UnitarySynthesis` for each chosen native basis with fixed approximation options | Synthesis seconds and gate count; compare operator error separately so faster approximate output cannot masquerade as an exact improvement |
| Final error-aware layout | Prepare an already routed circuit and a target with distinct, frozen per-edge error estimates; run `VF2PostLayout` with a fixed search limit | Search seconds and whether the chosen assignment satisfies the intended target/error preference; this estimates placement quality, not real hardware fidelity |
| Direction, local support, and angles | Prepare separate fixtures with a reversed CX on a one-way edge, an instruction supported only on particular qubits, and a rotation requiring angle normalization; run the relevant translation pipeline | Cost and validity of repairs that symmetric, globally supported fixtures miss |
| Two-qubit peephole and loops | Prepare two-qubit regions with a known cheaper equivalent sequence; time the peephole pass separately, then run a complete preset with iteration instrumentation outside timing | Rewrite cost, native gate reduction, and repeated optimization/retranslation behavior |
| Standard scheduling | Prepare one native static DAG and fixed durations/alignment constraints; run ASAP or ALAP, required rescheduling, and `PadDelay` | Scheduling seconds, makespan, and validity without conflating the result with routing or dynamical decoupling |
| Preset creation | Prepare a fixed target and options, then time only `generate_preset_pass_manager(...)` | Configuration/assembly overhead before any circuit is processed |
| Native gate counts, memory, multi-seed quality and throughput | Reuse the relevant prepared pipeline fixtures and apply the measurement boundaries in sections 2 and 5 | Additional metrics on the same workloads; these do not require an unrelated new application benchmark |
