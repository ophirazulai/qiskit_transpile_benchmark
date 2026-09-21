# Qiskit — High-Level Architecture Overview

*Target version: 2.6.0 (dev). This document describes the main components/modules of the
`qiskit` package and the Rust workspace that backs it.*

---

## 1. The big picture

Qiskit is a **quantum circuit compiler and SDK**. Almost everything in the repository serves one
of four jobs:

| Job | Modules |
| --- | --- |
| **Describe** a quantum program | `qiskit.circuit`, `qiskit.circuit.library`, `qiskit.classical` (expressions), `qiskit.quantum_info` |
| **Compile** it for a target machine | `qiskit.transpiler`, `qiskit.passmanager`, `qiskit.dagcircuit`, `qiskit.synthesis`, `qiskit.compiler` |
| **Execute** it | `qiskit.primitives`, `qiskit.providers`, `qiskit.result` |
| **Move it in and out** of Qiskit | `qiskit.qasm2`, `qiskit.qasm3`, `qiskit.qpy`, `qiskit.visualization`, `qiskit.capi` |

The canonical user flow:

```
QuantumCircuit  ──transpile()──►  ISA QuantumCircuit  ──Sampler/Estimator──►  PrimitiveResult
   (abstract)                        (target-legal)                              (data)
      │                                    │
      │ circuit_to_dag()                   │ runs on
      ▼                                    ▼
  DAGCircuit  ◄── pass manager ──►    BackendV2 / Target
```

Qiskit is a **hybrid Python/Rust codebase**. The Python package is the API surface; the
performance-critical data model and algorithms live in a Rust workspace compiled into a single
extension module, `qiskit._accelerate`.

---

## 2. Core data model — `qiskit.circuit`

The heart of the SDK. Defines the quantum program IR that every other module consumes.

**Key objects**

- `QuantumCircuit` — the user-facing container: an ordered list of instructions over qubits,
  clbits and registers (`QuantumRegister`, `ClassicalRegister`, `AncillaRegister`). Mutable,
  builder-style. The data store itself is Rust-backed.
- `Operation` (protocol) → `Instruction` → `Gate` → `ControlledGate` — the instruction hierarchy.
  `Operation` is the minimal interface the transpiler needs; `Gate` adds unitarity.
- `AnnotatedOperation` (`annotated_operation.py`, `annotation.py`) — wraps a base operation with
  *modifiers* (control, inverse, power) so that synthesis can be deferred to compile time rather
  than eagerly expanded.
- `Parameter` / `ParameterExpression` / `ParameterVector` — symbolic, unbound circuit parameters,
  enabling parametric circuits and late binding.
- `Barrier`, `Measure`, `Reset`, `Delay`, `Store` — the non-unitary built-ins.
- `singleton.py` — the machinery that makes stateless standard gates (e.g. `XGate()`) shared
  immutable singletons, a major memory/allocation win.

**Sub-packages**

- `circuit/library/` — the standard circuit/gate library: `standard_gates`, `n_local` ansätze,
  `arithmetic`, `boolean_logic`, `data_preparation` (state prep, Pauli feature maps),
  `basis_change` (QFT), `generalized_gates`, plus named constructions (`grover_operator`,
  `quantum_volume`, `phase_estimation`, `iqp`, `pauli_evolution`, oracles) and `templates` used by
  template-matching optimization.
- `circuit/controlflow/` — structured classical control flow as instructions: `IfElseOp`,
  `ForLoopOp`, `WhileLoopOp`, `SwitchCaseOp`, `BoxOp`, `break`/`continue`, plus the *builder
  interface* (`with circuit.if_test(...)`) in `builder.py`.
- `circuit/classical/` — the real-time classical computation layer: `expr` (typed expression
  trees over clbits/registers/`Var`s) and `types` (the type system). This is what control-flow
  conditions and `Store` operate on.
- `circuit/equivalence.py`, `equivalence_library.py` — the `EquivalenceLibrary`: a graph of
  gate↔circuit equivalences used by basis translation.
- `commutation_checker.py` / `commutation_library.py` — commutation analysis used by several
  optimization passes.
- `circuit/random/`, `twirling.py`, `tools/` — random circuit generation, Pauli twirling, helpers.

---

## 3. Compiler IR — `qiskit.dagcircuit` and `qiskit.converters`

The transpiler does **not** operate on `QuantumCircuit`; it operates on a DAG.

- `DAGCircuit` — directed acyclic graph where nodes are operations (`DAGOpNode`) and wire
  endpoints (`DAGInNode`/`DAGOutNode`), and edges are qubits/clbits/vars. Gives topological
  ordering, layer extraction, substitution and node-level rewriting. Backed by `rustworkx`.
- `DAGDependency` / `DAGDependencyV2` — an alternative representation where edges encode
  *commutation-aware* dependencies rather than wire order; used by template matching and some
  optimizations.
- `collect_blocks.py` — grouping runs/blocks of nodes (e.g. 2-qubit blocks) for resynthesis.
- `qiskit.converters` — the explicit conversion layer between representations:
  `circuit_to_dag`, `dag_to_circuit`, `circuit_to_gate`, `circuit_to_instruction`,
  `dag_to_dagdependency`, and friends.

---

## 4. Pass infrastructure — `qiskit.passmanager`

A **domain-agnostic** compilation framework (it knows nothing about quantum circuits).

- `Task` / `GenericPass` (`base_tasks.py`) — the unit of work.
- `PassManager` — runs a sequence of tasks over an IR, with `PropertySet` state.
- `FlowController`s (`flow_controllers.py`) — `DoWhileController`, `ConditionalController` for
  loops and branching over tasks.
- `PropertySet` / `WorkflowStatus` / `PassManagerState` (`compilation_status.py`) — analysis
  results and run metadata shared between passes.
- `BasePassManager` (`passmanager.py`) — the IR-generic base; `MultiStagePassManager`
  (`multistage_passmanager.py`) — the staged composition model.

`qiskit.transpiler` specializes this for circuits (`transpiler/passmanager.py`):
`TransformationPass` / `AnalysisPass` (`basepasses.py`), the circuit-aware `PassManager`, and
`StagedPassManager`.

---

## 5. The transpiler — `qiskit.transpiler`

The largest and most important subsystem: rewrite an abstract circuit into one that is legal and
efficient on a specific device.

### 5.1 Target description

- `Target` — the single source of truth about a backend: which instructions exist, on which
  qubits, with what error/duration properties, plus global constraints. Rust-backed.
- `CouplingMap` — the device connectivity graph.
- `Layout` / `TranspileLayout` — the virtual-qubit → physical-qubit mapping, and the record of
  how it evolved (initial layout + routing permutation) so results can be mapped back.
- `InstructionDurations`, `TimingConstraints` — timing model for scheduling.
- `PassManagerConfig` — the bundle of options a preset pass manager is built from.

### 5.2 Stages

Compilation is organized as a `StagedPassManager` with six stages, each independently pluggable
via entry points (`qiskit.transpiler.<stage>`):

| Stage | Purpose | Representative passes (`transpiler/passes/…`) |
| --- | --- | --- |
| **init** | Normalize/unroll to something routable; early optimization | `passes/utils`, `passes/optimization` |
| **layout** | Choose initial virtual→physical qubit assignment | `VF2Layout`, `SabreLayout`, `DenseLayout`, `TrivialLayout` |
| **routing** | Insert swaps so 2q gates respect connectivity | `SabreSwap`, `BasicSwap`, `LookaheadSwap`, `StarPreRouting` |
| **translation** | Rewrite into the target's native basis | `BasisTranslator`, `UnrollCustomDefinitions`, `HighLevelSynthesis` |
| **optimization** | Reduce depth/gate count; runs to fixed point | `Optimize1qGatesDecomposition`, `CommutativeCancellation`, `ConsolidateBlocks`, `UnitarySynthesis`, `TemplateOptimization`, `ElidePermutations` |
| **scheduling** | Assign times, insert delays, dynamical decoupling | `ALAPScheduleAnalysis`, `ASAPScheduleAnalysis`, `PadDelay`, `PadDynamicalDecoupling` |

Additional pass groups: `passes/analysis` (depth, size, width, `VF2PostLayout`),
`passes/basis`, `passes/synthesis` (the plugin-driven synthesis passes).

### 5.3 Preset pipelines

- `generate_preset_pass_manager(optimization_level=0..3, target=..., backend=...)` — the
  recommended entry point; `level0.py`–`level3.py` define the presets.
- `common.py` — shared stage construction; `builtin_plugins.py` — the built-in stage plugins;
  `clifford_t.py` and `pbc/` — specialized pipelines (Clifford+T, Pauli-based computation).
- `plugin.py` — the `PassManagerStagePlugin` interface enabling external providers to replace any
  stage.
- `qiskit.compiler.transpile` / top-level `qiskit.transpile` — the thin user-facing wrapper.

### 5.4 Extensibility

Four documented plugin points, all via Python entry points:
`qiskit.transpiler.{init,layout,routing,translation,optimization,scheduling}` (stage plugins),
`qiskit.unitary_synthesis` (unitary decomposition), and `qiskit.synthesis`
(high-level-object synthesis, e.g. `clifford.*`, `mcx.*`, `permutation.*`, `qft.*`).

---

## 6. Synthesis — `qiskit.synthesis`

Algorithms that turn a mathematical description into a circuit. Consumed by the transpiler's
translation/optimization stages, and usable directly.

- `one_qubit`, `two_qubit` — Euler-angle decomposition, KAK/Weyl (`TwoQubitBasisDecomposer`),
  approximate 2q synthesis.
- `unitary` — general n-qubit unitary decomposition (Quantum Shannon Decomposition, AQC).
- `clifford`, `cnotdihedral`, `stabilizer`, `linear`, `linear_phase`, `permutation` — structured
  group/normal-form synthesis, including connectivity-aware (LNN) variants.
- `discrete_basis` — Solovay–Kitaev and Ross–Selinger/gridsynth for discrete (Clifford+T) bases.
- `multi_controlled` — MCX/MCMT constructions with varying ancilla budgets.
- `evolution` — Trotter/Suzuki product formulas and `LieTrotter`/`SuzukiTrotter` synthesizers for
  `PauliEvolutionGate`.
- `arithmetic`, `boolean`, `qft` — adders/multipliers, Boolean-function circuits, QFT variants.

---

## 7. Quantum information — `qiskit.quantum_info`

Simulation-free mathematical objects, independent of the circuit IR (but interoperable with it).

- **States** (`states/`): `Statevector`, `DensityMatrix`, `StabilizerState`, plus fidelity/entropy
  measures and random-state generators.
- **Operators** (`operators/`):
  - `Operator`, `ScalarOp` — dense matrix operators.
  - `symplectic/` — `Pauli`, `PauliList`, `SparsePauliOp`, `Clifford`: the efficient
    Pauli/stabilizer algebra that observables and many transpiler passes rely on.
  - `channel/` — quantum channel representations `Choi`, `SuperOp`, `Kraus`, `Chi`, `PTM`,
    `Stinespring` and conversions between them.
  - `dihedral/` — CNOT-dihedral group elements.
  - `measures.py` — process/state fidelity, diamond norm; `predicates.py` — matrix predicates.
  - `mixins/`, `op_shape.py` — the shared algebraic interface (`compose`, `tensor`, `adjoint`, …).
- `analysis/` — Z2 symmetries, distributions; `quaternion.py`, `random.py` — utilities.

---

## 8. Execution interface — `qiskit.primitives` and `qiskit.providers`

### 8.1 Primitives (the recommended execution API)

Two abstractions cover most quantum workloads:

- **Sampler** — takes circuits with measurements, returns bitstring samples.
- **Estimator** — takes circuits + observables, returns expectation values.

Structure:
- `base/` — `BaseSamplerV2`, `BaseEstimatorV2` (and the deprecated V1 bases).
- `containers/` — the V2 data model: a **PUB** ("Primitive Unified Bloc") bundles a circuit with
  its parameter bindings and (for Estimator) observables — `SamplerPub`, `EstimatorPub`,
  `BindingsArray`, `ObservablesArray`; results come back as `PrimitiveResult` / `PubResult` /
  `DataBin` / `BitArray`, all array-shaped.
- Reference implementations: `StatevectorSampler`, `StatevectorEstimator` (exact, local) and
  `BackendSamplerV2`, `BackendEstimatorV2` (adapters over any `BackendV2`).
- `PrimitiveJob` — the local job handle.

### 8.2 Providers

The hardware/simulator abstraction layer.

- `BackendV2` — the backend interface: exposes a `Target`, `run()`, and options.
- `Job` (`JobV1`) / `JobStatus` — asynchronous result handles.
- `Options` — backend run configuration.
- `basic_provider/` — `BasicSimulator`, an in-tree Python statevector simulator, useful as a
  reference and for testing.
- `fake_provider/` — `GenericBackendV2`, a configurable synthetic backend for building realistic
  `Target`s without hardware access.

Real hardware access lives in external packages (e.g. `qiskit-ibm-runtime`) implementing these
interfaces.

### 8.3 Results — `qiskit.result`

`Result`, `Counts`, quasi/probability `distributions`, marginalization and post-processing
(`postprocess.py`, `utils.py`, `sampled_expval.py`).

---

## 9. Serialization and interchange

- `qiskit.qpy` — Qiskit's own **binary** circuit serialization format. Versioned and
  backward-compatible: `formats.py` (struct layouts), `binary_io/` (readers/writers),
  `interface.py` (`dump`/`load`), `type_keys.py`. The only lossless format for Qiskit-specific
  constructs (parameters, control flow, annotations).
- `qiskit.qasm2` — fast OpenQASM 2 `loads`/`dumps` (parser in Rust, `crates/qasm2`).
- `qiskit.qasm3` — OpenQASM 3 `dumps` via an AST + printer (`ast.py`, `exporter.py`,
  `printer.py`); import goes through the external `openqasm3_parser` Rust crates.
- `qiskit.capi` — Python-side access to Qiskit's **public C API** (header and shared-library
  locations), so compiled extensions can work with Qiskit objects without the interpreter.

---

## 10. Visualization — `qiskit.visualization`

Drawing backends for the main objects: `circuit/` (text, matplotlib, LaTeX drawers),
`dag_visualization.py`, `state_visualization.py` (Bloch sphere, city/hinton/Q-sphere/Pauli-vector
plots), `counts_visualization.py` (histograms), `gate_map.py` (device coupling maps),
`timeline/` (scheduled-circuit timelines), `pass_manager_visualization.py`, and `style.py` for
theming. All heavy dependencies (matplotlib, pylatexenc, seaborn) are optional.

---

## 11. Cross-cutting utilities — `qiskit.utils`

- `optionals.py` + `lazy_tester.py` — the lazy optional-dependency system (`HAS_MATPLOTLIB`, …).
- `deprecation.py` — the decorators implementing the project's deprecation policy.
- `parallel.py` — `parallel_map` process-based parallelism.
- `units.py`, `classtools.py` — unit handling and class helpers.
- `qiskit.exceptions` — `QiskitError` and the exception hierarchy root.
- `user_config.py` — `~/.qiskit/settings.conf` user defaults.

---

## 12. The Rust workspace — `crates/`

All crates are `qiskit-*`. Only **one** Python extension module is produced
(`qiskit._accelerate`), because Rust FFI across dynamic-library boundaries is impractical — this
constraint shapes the whole layout.

| Crate | Role |
| --- | --- |
| `pyext` | The *only* crate that builds the Python C extension (`qiskit._accelerate`); it re-exports the `pyclass`/`pyfunction` definitions from all the others |
| `circuit` | Rust-space circuit data model — the base of the stack; everything circuit-related depends on it |
| `quantum_info` | Quantum-information objects/operations, with no circuit dependency (a base crate) |
| `transpiler` | Transpiler passes and machinery (`Target`, layout, routing, optimization) |
| `synthesis` | Synthesis algorithms |
| `circuit_library` | Circuit constructors built on the `circuit` data model |
| `providers` | Quantum-program representation and the backend interface that consumes it |
| `qasm2` | OpenQASM 2 parser |
| `qasm3` | Qiskit-side OpenQASM 3 import (grammar parsing is an external crate) |
| `qpy` | QPY serialization support |
| `accelerate` | Catch-all for one-off accelerators; top of the dependency tree |
| `cext` / `cext-vtable` | The public **C API** — standalone (no Python runtime) or embedded in `pyext` |
| `bindgen` / `bindgen-cli` | Generation and linting of the C header files and vtables |
| `util` | Qiskit-agnostic helpers shared by `circuit` and `quantum_info` |

**Initialization-order caveat:** `_accelerate` is the lowest layer of the Python package and must
initialize in one go, so Rust code cannot import other parts of the Python `qiskit` package at
module-init time (except exceptions via PyO3's `import_exception!`).

---

## 13. Dependency direction (simplified)

```
                     utils / exceptions
                             │
                    ┌────────┴────────┐
              quantum_info          circuit ──── circuit.library
                    │                  │
                    └────────┬─────────┘
                             │
                         synthesis
                             │
        passmanager ───► transpiler ◄─── dagcircuit / converters
                             │
                         compiler
                             │
          providers ───► primitives ───► result
                             │
        qpy / qasm2 / qasm3 / visualization / capi  (leaf consumers)
```

Rules of thumb that hold across the codebase:

1. `circuit` never imports `transpiler`.
2. `quantum_info` never imports circuit-level objects (the circuit↔quantum-info bridges live in
   `circuit` or in `synthesis`).
3. The transpiler consumes `Target`; it never talks to a backend directly.
4. Serialization and visualization are leaves — nothing in the core depends on them.

---

## 14. Repository layout

```
qiskit/          Python package (the API surface)
crates/          Rust workspace (data model + accelerators + C API)
test/            Test suite
docs/            Documentation sources
releasenotes/    Reno release-note fragments (one YAML per change)
tools/           Development and CI tooling
design/          Design/architecture notes (this document)
```
