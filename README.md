# Shor's Algorithm — Benchmarking on IBM Quantum Hardware

> **Master Thesis Experiment Notebook**
> Benchmarking Shor's order-finding algorithm (N = 15, a = 2) across ideal simulation, noisy simulation, and real IBM Quantum hardware.

Overleaf IEEE paper: [https://www.overleaf.com/read/mnhqjpfqdzgs#8afe03](https://www.overleaf.com/read/mnhqjpfqdzgs#8afe03)  
(includes detailed analysis and discussion)

---

## Table of Contents

1. [What This Project Does](#what-this-project-does)
2. [Background — Shor's Algorithm in a Nutshell](#background--shors-algorithm-in-a-nutshell)
3. [Project Structure](#project-structure)
4. [How the Notebook Is Organised](#how-the-notebook-is-organised)
5. [The Three Benchmarking Tiers](#the-three-benchmarking-tiers)
6. [Experiment Sweeps — What We Vary and Why](#experiment-sweeps--what-we-vary-and-why)
7. [Metrics — How We Measure Quality](#metrics--how-we-measure-quality)
8. [Key Functions Explained](#key-functions-explained)
9. [Output Artefacts](#output-artefacts)
10. [How to Run](#how-to-run)
11. [Dependencies](#dependencies)
12. [Configuration Reference](#configuration-reference)
13. [Troubleshooting](#troubleshooting)

---

## What This Project Does

This project implements **Shor's order-finding algorithm** for the case N = 15, a = 2 using IBM's Qiskit framework, and then **systematically benchmarks** it by running the *same* quantum circuit under three different conditions:

| Condition | What it tells us |
|-----------|-----------------|
| **Ideal simulation** | What the algorithm *should* produce with a perfect quantum computer |
| **Noisy simulation** | How hardware-like noise degrades the results (without waiting in a job queue) |
| **Real IBM hardware** | How the algorithm actually performs on a real quantum processor |

By comparing these three tiers across multiple configurations (different compiler settings, circuit sizes, and error-mitigation options), we get a **comprehensive picture** of the gap between theory and practice — exactly what is needed for a thesis-grade analysis.

---

## Background — Shor's Algorithm in a Nutshell

Shor's algorithm factors large integers efficiently on a quantum computer. The core quantum subroutine is **order finding**: given integers *a* and *N*, find the smallest positive integer *r* such that:

$$a^r \equiv 1 \pmod{N}$$

Once *r* is known, we can (with high probability) extract a non-trivial factor of *N* using:

$$\gcd(a^{r/2} \pm 1,\; N)$$

### How the circuit works

1. **Control register** — A set of qubits prepared in superposition (via Hadamard gates). These encode the phase estimation that reveals *r*.
2. **Target register** — 4 qubits representing the modular arithmetic workspace, initialised to |1⟩.
3. **Controlled modular multiplication** — For each control qubit *k*, we apply a controlled version of the gate that multiplies by $a^{2^k} \bmod N$.
4. **Inverse QFT** — Applied to the control register to convert phase information into a readable bitstring.
5. **Measurement** — We measure the control register. The result encodes a fraction $j/2^t$ close to $k/r$ for some integer $k$.
6. **Classical post-processing** — Continued fractions extract the denominator *r*, from which we compute factors.

### Why N = 15?

N = 15 is the **smallest non-trivial case** for Shor's algorithm (15 = 3 × 5). It requires only 4 target qubits for the modular arithmetic, making it feasible on today's quantum hardware. The modular multiplication gates ($M_2 \bmod 15$ and $M_4 \bmod 15$) can be implemented using just SWAP gates — no Toffoli or ancilla overhead.

For a = 2, the true order is **r = 4**, so the expected output phases are:

| Phase (θ) | Bitstring (4 control qubits) | Fraction k/r |
|-----------|------------------------------|--------------|
| 0.00 | 0000 | 0/4 |
| 0.25 | 0100 | 1/4 |
| 0.50 | 1000 | 2/4 |
| 0.75 | 1100 | 3/4 |

A perfect quantum computer would output **only** these four bitstrings with equal probability (25% each).

---

## Project Structure

```
Qiskit-v1/
├── Qiskit.ipynb              # Main experiment notebook (run this)
├── apikey.json                # IBM Quantum API token (auto-loaded)
├── instructions.md            # Detailed build instructions
├── README.md                  # This file
├── output.png                 # (optional) manually saved figure
└── results/                   # Auto-generated output directory
    ├── results.csv            # Flat table of all experiment metrics
    ├── results.json           # Full data (distributions + metadata)
    ├── success_prob_and_tvd.png   # Line plots
    └── success_prob_bar.png       # Bar chart
```

---

## How the Notebook Is Organised

The notebook has **36 cells** arranged in logical sections:

### Section 1 — Setup (Cells 1–2)

- **Cell 1**: Installs all Python dependencies via `pip`.
- **Cell 2**: Imports all libraries (Qiskit, NumPy, Pandas, Matplotlib, IBM Runtime).

### Section 2 — Configuration (Cells 3–4)

- **Cell 3**: Markdown explaining the configuration.
- **Cell 4**: A single code cell with **every tuneable parameter**. Change values here to reconfigure the entire experiment without touching any other cell.

### Section 3 — Authentication & Backend (Cells 5–6)

- **Cell 5**: Markdown explaining the auth flow.
- **Cell 6**: Connects to IBM Quantum Platform using a 3-step fallback:
  1. Saved IBM account (from a previous `save_account()` call)
  2. `apikey.json` file in the project directory
  3. Interactive password prompt (`getpass`)

  Then selects a backend — defaults to `ibm_marrakesh`, auto-falls-back to the least-busy backend if unavailable.

### Section 4 — Original Algorithm (Cells 7–18)

These cells are **preserved from the original notebook** and define the quantum circuit components:

| Cell | Content |
|------|---------|
| 8 | `M2mod15()` — gate that multiplies by 2 mod 15 (3 SWAPs) |
| 9 | Visualisation of M₂ |
| 10 | `controlled_M2mod15()` — controlled version |
| 11 | Visualisation of controlled-M₂ |
| 12 | `a2kmodN(a, k, N)` — computes $a^{2^k} \bmod N$ by repeated squaring |
| 13 | Demonstrates the sequence [2, 4, 1, 1, ...] for a=2, N=15 |
| 14 | `M4mod15()` — gate that multiplies by 4 mod 15 (2 SWAPs) |
| 15 | Visualisation of M₄ |
| 16 | `controlled_M4mod15()` — controlled version |
| 17 | Visualisation of controlled-M₄ |
| 18 | Assembles the **full order-finding circuit** (Hadamards → controlled multiplications → inverse QFT → measurement) and draws it |

### Section 5 — Experiment Functions (Cells 19–25)

Six reusable functions that form the experiment engine:

| Cell | Function | Purpose |
|------|----------|---------|
| 20 | `build_shor_circuit()` | Builds the circuit for any control-register size |
| 21 | `transpile_for_backend()` | Compiles the circuit for hardware and reports metrics |
| 22 | `run_ideal()` | Noiseless simulation (Tier A) |
| 23 | `run_noisy()` | Noisy simulation (Tier B) |
| 24 | `run_hardware()` | Real hardware execution (Tier C) |
| 25 | `compute_metrics()` | Computes TVD and success probability |

### Section 6 — Experiment Sweep (Cells 26–28)

- **Cells 26–27**: Markdown explaining the tiers and sweep dimensions.
- **Cell 28**: The main experiment loop that iterates over every configuration, runs all three tiers, and collects results into a list.

### Section 7 — Results & Plots (Cells 29–33)

- **Cell 30**: Builds a Pandas DataFrame, exports to CSV and JSON.
- **Cell 31**: Line plots — success probability and TVD across configurations.
- **Cell 32**: Grouped bar chart comparing Ideal / Noisy / Hardware.
- **Cell 33**: Styled summary table.

### Section 8 — Factor Extraction (Cells 34–35)

Takes the **best hardware run** (highest success probability), filters the measured bitstrings, and uses continued fractions to extract non-trivial factors of 15.

### Section 9 — How-to-Run Guide (Cell 36)

In-notebook instructions for setting up IBM Quantum access and running the experiment.

---

## The Three Benchmarking Tiers

### Tier A — Ideal Simulation

- **Tool**: Qiskit's `StatevectorSampler` (built-in, no Aer needed)
- **Noise**: None — mathematically perfect
- **Shots**: 100,000 (high count for a smooth reference distribution)
- **Purpose**: Establishes the **theoretical baseline**. This is what the algorithm *should* produce.

### Tier B — Noisy Simulation

- **Tool**: Qiskit Aer's `AerSimulator` with a noise model
- **Noise model priority**:
  1. Tries to build a noise model from the **real backend's calibration data** (`NoiseModel.from_backend()`).
  2. If that fails (which can happen with newer backends), falls back to a **generic depolarising model**: ~0.1% error on 1-qubit gates, ~1% error on 2-qubit gates.
- **Shots**: Same as hardware (default 1,024)
- **Purpose**: Provides an **intermediate reference** — shows what we'd expect from noisy hardware *without* waiting in a job queue. Useful for isolating which errors come from compilation vs. raw noise.

### Tier C — Real Hardware

- **Tool**: IBM Qiskit Runtime `SamplerV2` primitive
- **Backend**: A real IBM Quantum processor (e.g., `ibm_marrakesh`, 156 qubits)
- **Shots**: Configurable (default 1,024)
- **Error mitigation**: Configurable resilience level and dynamical decoupling
- **Purpose**: The **ground truth** of how the algorithm performs on actual quantum hardware today.

---

## Experiment Sweeps — What We Vary and Why

Each sweep axis isolates a different factor that affects quantum algorithm performance:

### 1. Transpilation Optimisation Level (0, 1, 2, 3)

**What it is**: The Qiskit transpiler converts a logical quantum circuit into physical gates that the hardware can execute. Higher optimisation levels try harder to reduce circuit depth (fewer operations = less decoherence).

| Level | Strategy |
|-------|----------|
| 0 | Minimal — just maps to hardware topology |
| 1 | Light optimisation (default) |
| 2 | Medium — gate cancellation, commutation |
| 3 | Heavy — tries many layouts, routing strategies |

**Why it matters for thesis**: Shows whether compiler improvements translate into measurable fidelity gains on real hardware.

### 2. Control-Qubit Precision (4, 6, 8)

**What it is**: The number of qubits in the phase-estimation (control) register. More qubits = higher phase resolution = more precise estimate of the order *r*.

| Control qubits | Total qubits | Phase resolution |
|----------------|-------------|-----------------|
| 4 | 8 | 1/16 = 0.0625 |
| 6 | 10 | 1/64 ≈ 0.0156 |
| 8 | 12 | 1/256 ≈ 0.0039 |

**Why it matters for thesis**: Tests the trade-off between precision and circuit depth. More qubits → a deeper circuit → more noise accumulation. At some point, adding precision *hurts* because the circuit becomes too deep for the hardware to execute reliably.

### 3. Resilience Level (0, 1)

**What it is**: An IBM Runtime option that enables built-in error mitigation:

| Level | Mitigation |
|-------|-----------|
| 0 | No mitigation |
| 1 | Twirling + basic readout error mitigation |

**Why it matters for thesis**: Quantifies the benefit of IBM's built-in error mitigation with zero user effort.

### 4. Dynamical Decoupling (Off, On)

**What it is**: When qubits are idle (waiting for other qubits to finish their gates), they accumulate decoherence errors. Dynamical decoupling inserts carefully timed X-pulses (specifically, the "XpXm" sequence) during idle periods to refocus the qubit and suppress this noise.

**Why it matters for thesis**: Tests whether a pure-pulse-level technique (no algorithmic overhead) can measurably improve results.

### Total Configurations

$$4 \text{ (opt levels)} \times 3 \text{ (control sizes)} \times 2 \text{ (resilience)} \times 2 \text{ (DD)} = 48 \text{ configurations}$$

Each configuration runs on all 3 tiers, producing a rich dataset for analysis.

---

## Metrics — How We Measure Quality

### Total Variation Distance (TVD)

$$\text{TVD}(P, Q) = \frac{1}{2} \sum_{x} |P(x) - Q(x)|$$

- Compares two probability distributions bitstring-by-bitstring.
- **Range**: 0 (identical distributions) to 1 (completely disjoint — no overlap).
- **We compute three variants**:
  - TVD(Hardware, Ideal) — how far hardware is from perfection
  - TVD(Noisy, Ideal) — how far the noise model is from perfection
  - TVD(Hardware, Noisy) — how well the noisy simulation predicts hardware behaviour

### Success Probability

The probability mass that falls within an ε-window of the **expected Shor peaks**.

For a = 2, N = 15, the expected phases are {0, 0.25, 0.5, 0.75}. For each measured bitstring:

1. Convert to a phase: $\theta = \text{int(bitstring)} / 2^t$
2. Check if $\theta$ is within ε of any expected phase (accounting for wrap-around at 0/1)
3. Sum up all the probability mass that falls in these windows

**Default ε**: $1 / 2^{t+1}$ where $t$ is the number of control qubits. This is the principled choice from phase-estimation theory — it's half the spacing between adjacent bitstring phases.

**Interpretation**: An ideal circuit should have success probability ≈ 1.0 (all mass on the peaks). Real hardware will be lower. The further below 1.0, the more the noise has smeared the output.

---

## Key Functions Explained

### `build_shor_circuit(num_control, num_target=4, a=2, N=15)`

Constructs the complete order-finding circuit:
1. Creates a control register (variable size) and a 4-qubit target register
2. Initialises the target to |1⟩
3. For each control qubit *k*, applies H then controlled-$M_{a^{2^k}}$
4. Applies inverse QFT to the control register
5. Measures the control register into classical bits

Returns a `QuantumCircuit` ready for simulation or transpilation.

### `transpile_for_backend(circuit, backend, optimization_level, seed)`

Compiles a logical circuit for a specific backend:
- Uses `generate_preset_pass_manager` from Qiskit
- Returns both the transpiled circuit and a metrics dict with:
  - `depth_2q`: Number of layers that contain 2-qubit gates (main predictor of fidelity)
  - `count_2q`: Total number of 2-qubit gates
  - `total_depth`: Full circuit depth

### `run_ideal(circuit, shots, seed)`

Runs a noiseless simulation using `StatevectorSampler`. Returns a dictionary mapping bitstrings to probabilities.

### `run_noisy(circuit, backend, shots, seed)`

Runs a noisy simulation:
1. Tries `NoiseModel.from_backend(backend)` to use the real device's error rates
2. Falls back to generic depolarising noise if that fails
3. Transpiles for the noisy simulator and executes

Returns a dictionary mapping bitstrings to probabilities.

### `run_hardware(transpiled_circuit, backend, shots, resilience_level, dd_enable)`

Submits a pre-transpiled circuit to IBM hardware:
1. Creates a `SamplerV2` instance pointing at the backend
2. Sets runtime options (resilience level, dynamical decoupling, gate twirling)
3. Submits the job and **waits for results**
4. Prints the job ID for tracking

Returns `(distribution_dict, job_id)`.

### `compute_metrics(dist, ideal_dist, noisy_dist, num_control, a, N, epsilon)`

Computes all analysis metrics for one experiment run:
- Three TVD values (hardware vs ideal, hardware vs noisy, noisy vs ideal)
- Success probability on expected Shor peaks
- Returns a dictionary ready to be stored in the results DataFrame

---

## Output Artefacts

All outputs are saved to the `results/` directory:

| File | Format | Contents |
|------|--------|----------|
| `results.csv` | CSV | One row per experiment configuration. Columns: run_id, timestamp, num_control, opt_level, resilience_level, dd_enable, depth_2q, count_2q, ideal/noisy/hw success probs, TVDs, job_id |
| `results.json` | JSON | Everything in the CSV **plus** the full probability distributions for every run (ideal, noisy, hardware). Suitable for re-analysis without re-running. |
| `success_prob_and_tvd.png` | PNG | Two-panel figure: (left) success probability vs optimisation level, (right) TVD vs optimisation level, both grouped by control-qubit count |
| `success_prob_bar.png` | PNG | Grouped bar chart showing Ideal / Noisy / Hardware success probability for every configuration side-by-side |

---

## How to Run

### Prerequisites

- Python 3.10+ (tested with 3.14)
- An IBM Quantum account — free at [quantum.cloud.ibm.com](https://quantum.cloud.ibm.com/)

### Step 1 — Set up authentication

**Option A** (recommended): Save your token once and forget about it:
```python
from qiskit_ibm_runtime import QiskitRuntimeService
QiskitRuntimeService.save_account(
    channel="ibm_quantum_platform",
    token="YOUR_TOKEN_HERE",
    overwrite=True,
)
```

**Option B**: Place your token in `apikey.json`:
```json
{"apikey": "YOUR_TOKEN_HERE"}
```

**Option C**: The notebook will prompt you interactively if neither of the above works.

### Step 2 — Configure the experiment

Open the notebook and edit the **Configuration** cell (cell 4):

| Variable | Default | What to change |
|----------|---------|---------------|
| `SHOTS` | 1024 | Increase for smoother distributions (costs more queue time) |
| `BACKEND_NAME` | `"ibm_marrakesh"` | Set to any backend, or `None` for auto-select |
| `RUN_HARDWARE` | `True` | Set `False` to skip hardware (simulations only) |
| `CONTROL_QUBIT_SWEEP` | `[4, 6, 8]` | Reduce to `[4]` for a quick test |
| `OPT_LEVEL_SWEEP` | `[0, 1, 2, 3]` | Reduce to `[1, 3]` for fewer runs |

### Step 3 — Run the notebook

Click **Run All** (or execute cells sequentially). The experiment loop (cell 28) will:
1. Print progress for each configuration
2. Print job IDs for every hardware submission
3. Wait for each job to complete before moving to the next

**Estimated time**:
- Simulations only (`RUN_HARDWARE = False`): ~2–5 minutes
- With hardware (48 configs × ~30s queue + execution each): 30–90 minutes depending on queue

### Step 4 — Check results

- Plots appear inline in the notebook
- CSV/JSON/PNG files are saved in `results/`
- Factor extraction output appears in cell 35

---

## Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `qiskit` | ≥ 2.1.0 | Core quantum circuit framework |
| `qiskit-ibm-runtime` | ≥ 0.40.1 | IBM Quantum cloud access (SamplerV2 primitive) |
| `qiskit-aer` | ≥ 0.17.0 | Local noisy simulation |
| `numpy` | any | Numerical operations |
| `pandas` | any | DataFrame for results |
| `matplotlib` | any | Plotting |
| `pylatexenc` | any | LaTeX rendering in circuit diagrams |
| `jinja2` | any | Pandas styled table rendering |

All are installed automatically by cell 1 of the notebook.

---

## Configuration Reference

All variables are in cell 4 of the notebook:

```python
SHOTS               = 1024          # Shots per circuit execution
BACKEND_NAME        = "ibm_marrakesh"  # Preferred backend (None → auto)
RUN_HARDWARE        = True          # False → simulations only
OUTPUT_DIR          = "./results"   # Where to save artefacts
SEED_TRANSPILER     = 42            # Reproducible compilation
SEED_SIMULATOR      = 42            # Reproducible simulation

N                   = 15            # Number to factor
a                   = 2             # Base for order finding
num_target          = 4             # Target register size

OPT_LEVEL_SWEEP     = [0, 1, 2, 3]
CONTROL_QUBIT_SWEEP = [4, 6, 8]
RESILIENCE_LEVEL_SWEEP = [0, 1]
DD_SWEEP            = [False, True]
EPSILON             = None          # Auto-computed from control size
```

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `ModuleNotFoundError: No module named 'qiskit'` | Run cell 1 to install dependencies, or activate the virtual environment |
| `AttributeError: The '.style' accessor requires jinja2` | Run cell 1 again (jinja2 is now included). The summary cell also has a fallback that works without jinja2. |
| `AccountNotFoundError` or authentication failure | Check that your token is valid at [quantum.cloud.ibm.com](https://quantum.cloud.ibm.com/). Re-run `save_account()` or update `apikey.json`. |
| Backend unavailable | Set `BACKEND_NAME = None` to auto-select the least-busy backend. |
| Hardware jobs stuck in queue | This is normal. Jobs queue behind other users. You can monitor status at the IBM Quantum Dashboard. Set `RUN_HARDWARE = False` to iterate on analysis without waiting. |
| `No noise model from backend` warning | Expected with some newer backends. The notebook automatically falls back to a generic depolarising model. |

---

## License

This is a thesis research project. Please cite appropriately if reusing.
