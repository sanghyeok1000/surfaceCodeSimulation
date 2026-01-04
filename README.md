# surfaceCodeSimulation
# Coordinate-Faithful Multi-Patch Surface Code Simulation (MWPM + Unmatched Post-Correction)

This repository provides a **coordinate-faithful planar surface-code simulation** on a **multi-patch lattice** using **Qiskit Aer**.  
It supports:
- automatic **lattice generation** (data/ancilla coordinates),
- automatic **stabilizer generation** (X/Z plaquette operators),
- **syndrome extraction** via ancilla circuits,
- **minimum-weight perfect matching (MWPM)** decoding on a **diagonal metric**,
- **post-correction** for **unmatched defects** using exclusivity rules,
- and **dynamic logical-basis interpretation** (logical 0/1 sets derived from stabilizers).

The notebook is designed to test scaling behaviors by varying `(n, l, m)` and to evaluate logical outcomes such as `balanced / constant / inconclusive` and composite logical-state distributions.

---

## 1. Core Idea

### Coordinate-faithful lattice
Each surface-code patch `L_k` is represented by explicit coordinates:
- **Data qubits**: `D{k}_{i}` on odd-odd coordinates `(x,y)`
- **X-ancilla qubits**: `Ax{k}_{i}` on a checkerboard subset of even-even points
- **Z-ancilla qubits**: `Az{k}_{i}` on the complementary subset of even-even points

Adjacency is defined **geometrically**: an ancilla couples to data qubits at diagonal distance `(|dx|,|dy|)=(1,1)`.

### Multi-patch layout
The system builds an `l × m` array of patches with spacing `d`.  
Patch labels are `L_1, L_2, ..., L_{l*m}`.

---

## 2. What This Notebook Does (Pipeline)

### Step A. Generate lattice coordinates and adjacency
- `generate_single_grid(n, grid_index, offset)`  
  Returns coordinate dictionaries and adjacency maps:
  - `di_coords`: data qubit coordinates (`Dk_i → (x,y)`)
  - `axi_coords`: X-ancilla coordinates (`Axk_i → (x,y)`)
  - `azi_coords`: Z-ancilla coordinates (`Azk_i → (x,y)`)
  - `adj_axi`: mapping `Axk_i → [Dk_j,...]`
  - `adj_azi`: mapping `Azk_i → [Dk_j,...]`

- `plot_multi_grids(n, l, m, d)`  
  Visualizes the full lattice and returns `grids_info`.

### Step B. Generate stabilizer strings
- `generate_stabilizers(grids_info)`  
  Builds stabilizers per patch:
  - Z-stabilizers from `Az` adjacency (Pauli `Z` on its neighboring data qubits)
  - X-stabilizers from `Ax` adjacency (Pauli `X` on its neighboring data qubits)

Each stabilizer is stored as a Pauli string over data-qubit ordering.

### Step C. Build Qiskit registers and encode into stabilizer codespace
For each patch `L_k`:
- `ql_L_k`: data register
- `qx_L_k`: X-syndrome ancillas
- `qz_L_k`: Z-syndrome ancillas

Encoding:
- Uses `qiskit.synthesis.stabilizer.synth_circuit_from_stabilizers(...)`
- `allow_underconstrained=True` (useful when stabilizer sets do not fully constrain the state)

### Step D. Inject Pauli noise on **data qubits only**
- `inject_pauli_error(circuit, logical_registers, error_probability)`
  Applies random single-qubit `X/Y/Z` errors to **data registers only** with probability `p`.

### Step E. Syndrome extraction with deep-copy simulation
- `extract_syndrome(qc, grids_info, logical_registers, x_ancilla_registers, z_ancilla_registers)`
  1. deep-copies the circuit (`qc_sim`)
  2. applies syndrome extraction circuits per patch:
     - **Z syndrome**: `CX(data → Z-ancilla)`
     - **X syndrome**: `H(ancilla)`, `CX(X-ancilla → data)`, `H(ancilla)`
  3. measures ancillas into classical registers
  4. runs `AerSimulator` with `shots=1`
  5. returns the `counts` dictionary as the syndrome

### Step F. MWPM decoding on diagonal metric
- `error_correction(qc, syndrome, grids_info, logical_registers, n, m, verbose=False)`

Key features:
- syndrome bitstring is split into per-lattice slices:
  - `x_slice`: interpreted as **Az defects** (X errors)
  - `z_slice`: interpreted as **Ax defects** (Z errors)
- each defect is represented as an ancilla coordinate
- edge weights are computed using **BFS shortest path** under diagonal moves only:
  - allowed moves: `(±2, ±2)`
- MWPM is performed with `networkx.min_weight_matching(...)`
- correction is applied on **midpoints** of each diagonal step:
  - if midpoint matches a data-qubit coordinate, apply `X` or `Z` on that data qubit

Outputs per patch:
- corrected data qubits (`X_correction`, `Z_correction`)
- correction paths (`X`, `Z`)
- unmatched defects (`unmatched_X`, `unmatched_Z`)

### Step G. Post-correction for unmatched defects (exclusivity rule)
If MWPM cannot pair a defect (odd number of defects, or insufficient defects), the pipeline applies an additional heuristic:

- `generate_unmatched_syndrome_all(grids_info, correction_paths)`
  Generates per-lattice unmatched syndrome strings:
  - `L_k: {"X": "...", "Z": "..."}`

- `apply_unmatched_syndrome_correction(qc, unmatched_syndrome, grids_info, logical_registers, n, m)`
  For each unmatched ancilla:
  1. collect candidate adjacent data qubits
  2. keep only **exclusive** data qubits (connected to this ancilla only)
  3. if exclusive set size is 1 or 2, apply correction on the first one

- `full_error_correction(...)`  
  Runs MWPM correction + unmatched post-correction in one call.

### Step H. Logical operations and lattice transforms
The notebook includes logical operations acting on each patch:
- `xl(...)`: applies logical X along the central row (coordinate median)
- `zl(...)`: applies logical Z along the central column (coordinate median)
- `Hl(...)`: transversal Hadamard
- `lcx(...)`: transversal CNOT between two patches (pairwise across all data qubits)
- `rot(...)`: rotates the patch by permuting qubits using `SWAP` based on coordinate mapping

### Step I. Dynamic logical-basis interpretation (logical 0/1 sets)
The notebook classifies measured data-bitstrings using a dynamic logical basis:
- `generate_logical_basis_sets(x_stabs_L1, n)`  
  (Defined elsewhere in the notebook)  
  Produces:
  - `logical0_set_dynamic`
  - `logical1_set_dynamic`

Final result aggregation:
- `interpret_balanced_constant_from_counts(...)`
  Outputs `{balanced, constant, inconclusive}` counts.
- `interpret_composite_logical_state(...)`
  Produces distribution of multi-patch logical state strings, e.g. `logical |0010?>`.

---

## 3. How To Run

### Recommended environment
- Python 3.10+
- Qiskit Aer (tested with Qiskit 1.x series)
- networkx, numpy, matplotlib

Install:
```bash
pip install qiskit qiskit-aer networkx numpy matplotlib
