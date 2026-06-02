# ADAN-86 Pressure-Wave Propagation Animation

A 1D pulse-wave simulation of the ADAN-86 arterial tree, rendered as a 20-second video
where vessel color encodes local pressure and a side panel streams a brachial-artery
pressure trace in real time.

The repository contains two parallel solver tracks (V1 and V2) and a single shared
animation notebook that exports MP4s from saved pressure snapshots.

---

## 1. Final deliverables

Both videos are 20 s @ 30 fps, ~8 MB each. The lower panel is a monitor-style
right-brachial pressure trace looping the chosen cardiac cycle.

| File | Track | Source candidate |
|---|---|---|
| `results/video_exports/candidate_A_v1_8cycles_adan86_20s_cycle_loop_period1_offset0.mp4` | V1 | `optuna_E_best_trial_4_long100.pkl` (Study E, trial 4) |
| `results/video_exports/candidate_B_bg_v2_8cycles_adan86_20s_cycle_loop_period1_offset0.mp4` | V2 | `nb04_bg_v2_final_animation_ready_8cycles.pkl` (Experiment 19) |

Both candidates are produced by the same animation notebook
(`06_final_animation_export.ipynb`) - only the active candidate switch differs.

## Demo video Simple

Click the preview below to watch the ADAN-86 pressure-wave propagation animation on YouTube.

[![ADAN-86 pressure-wave propagation animation](https://img.youtube.com/vi/NsJphGUbqeg/maxresdefault.jpg)](https://www.youtube.com/watch?v=NsJphGUbqeg)
---

## 2. Model and physics

- **Topology**: ADAN-86 (Anatomically Detailed Arterial Network, 86 large arteries).
  In the CellML representation, several anatomical arteries are split into multiple
  computational segments, so the parser yields **103 computational vessels** and
  **102 parent → child edges**.
- **Heart**: a single root node driving a synthetic flow waveform at the aortic root.
  No chambers, no valves.
- **Vessels**: 1D compliant tubes solved from the standard reduced Navier-Stokes form
  (continuity + momentum) with an elastic wall law
  `P = β · (√A − √A₀)`.
- **Numerical scheme**: MacCormack predictor-corrector on the interior of each vessel,
  with a CFL-limited global time step.
- **Junctions**: characteristic-impedance coupling using a shared junction pressure
  (`Pj` such that `Qj = Qp + (Pp - Pj)/Zp` on the parent side and the corresponding
  relation on each child, plus mass conservation).
- **Terminals**: 3-element Windkessel (R1, C, R2) - peripheral resistance and venous
  return collapsed into one outlet element. No veins are modeled.
- **Inlet BC**: Riemann-like compatibility at the aortic root - inlet flow is
  prescribed, inlet pressure is back-estimated from the first interior cell.
- **No CFD, no 3D**: the network animation is a 2D layout of nodes and edges; color
  encodes local mean pressure per vessel; line width can be scaled with area.

The 2D layout is derived from the CellML 3D coordinates, projected and cleaned up
for legibility (the goal is a readable schematic, not anatomical realism).

---

## 3. Repository layout

```
adan_project/
├── main_ADAN-86.cellml              # ADAN-86 model (CellML, from Physiome)
├── Parameters86.cellml              # Vessel parameters
├── BG_Modules_v2.cellml             # Bond-graph modules incl. wall-law constants
├── 01_parse_adan_cellml_v1.ipynb
├── 01_parse_adan_cellml_v2_BG_fixed.ipynb
├── 02_baseline_cpu_solver_v1.ipynb
├── 02_baseline_cpu_solver_v2_BG_adapted.ipynb
├── 03_characteristic_cpu_solver_v1.ipynb
├── 03_characteristic_cpu_solver_v2_BG_adapted.ipynb
├── 04_speedup_solver_numba_v1.ipynb
├── 04_speedup_solver_numba_v2.ipynb
├── 05_brachial_optimization.ipynb        # V1 only — Optuna studies A-E
├── 06_final_animation_export.ipynb       # Shared MP4 export
├── 06_final_animation_export_v2.ipynb    # Improved animation 
├── adan_parsed.pkl                       # V1 parser output
├── adan_parsed_bg.pkl                    # V2 parser output (BG wall law)
├── adan86_*.csv                          # Inspection / debug tables
├── requirements.txt
└── results/
    ├── optuna_E_best_trial_4_long100.pkl
    ├── nb04_bg_v2_final_animation_ready_8cycles.pkl
    └── video_exports/
        ├── candidate_A_v1_8cycles_*.mp4
        └── candidate_B_bg_v2_8cycles_*.mp4
```

`results/` and `*.pkl` are in `.gitignore`; the final MP4s and the two animation-ready
pickles are committed because they are the deliverables.

---

## 4. Two tracks: V1 and V2

Both tracks parse the same CellML model, run the same MacCormack + characteristic
+ Windkessel solver, and feed `06_final_animation_export.ipynb`. They differ in
**how vessel wall thickness - and therefore wave speed - is set**, and in **how
the brachial waveform was tuned**.

### V1 - original parser, Optuna-tuned brachial

Notebooks used:

1. `01_parse_adan_cellml_v1.ipynb` - first ADAN-86 parser. Extracts vessels,
   parent → child topology, geometry, stiffness, terminals; writes
   `adan_parsed.pkl`.
2. `02_baseline_cpu_solver_v1.ipynb` - first stable CPU solver: MacCormack
   interior, baseline impedance coupling at junctions, baseline 3-element
   Windkessel, Riemann-like root inlet. Numerical limiter on `A/A_ref` and on
   `u = Q/A`.
3. `03_characteristic_cpu_solver_v1.ipynb` - adds the *fast characteristic*
   variants for vessel-to-vessel coupling and for terminal Windkessel, plus an
   optional radius-scaled flow damping term to suppress reflected ringing.
4. `04_speedup_solver_numba_v1.ipynb` - moves the hot path to Numba: flattened
   `A_flat`/`Q_flat`/`offsets`/`lengths` arrays, Numba MacCormack kernel,
   Numba stabilizer, Numba junction and Windkessel kernels. Includes profiling
   evidence for both accepted and rejected rewrites (e.g. typed-list Windkessel
   was rejected as slower than expected).
5. `05_brachial_optimization.ipynb` - Optuna studies A → E searching the
   parameter space (Windkessel R/C, regional stiffness scaling, inlet
   waveform shape) for a brachial waveform that is realistic *and* stable
   across many cycles.

**Final V1 candidate**: Study E, trial 4 (`optuna_E_best_trial_4_long100.pkl`).
Study E was specifically broadened to add a drift-aware long-run validation -
short scoring windows can hide slow cycle-to-cycle drift that would ruin a
20-second video. Trial 4 was the candidate that combined a plausible
brachial morphology with low long-run drift.

### V2 - fixed wall-law parser, hand-tuned through controlled experiments

Notebooks used:

1. `01_parse_adan_cellml_v2_BG_fixed.ipynb` - fixed parser.
2. `02_baseline_cpu_solver_v2_BG_adapted.ipynb` - same baseline solver, adapted
   to consume `adan_parsed_bg.pkl` and the new `hv`, `betav`, `c0v`, `Z0v`,
   `dt_global` arrays coming from the BG wall law.
3. `03_characteristic_cpu_solver_v2_BG_adapted.ipynb` - characteristic variants
   ported to the BG-wall arrays.
4. `04_speedup_solver_numba_v2.ipynb` - *clean* Numba framework on top of the
   fixed parser. Rejected experimental branches from V1 (`riemann_relaxed`,
   `baseline_interior`, `maccormack_all_flat_interior_numba`) are intentionally
   removed. The accepted framework is **`baseline`** as the stable reference
   branch and **`fast_characteristic`** as the morphology branch with the
   MacCormack kernel `maccormack_all_flat_numba`. V2 then runs experiments
   01-19 directly inside this notebook (no Optuna): inlet calibration,
   morphology branch stabilization, multi-cycle convergence, compliance
   refinement, notch shaping, and timing sweeps.

**Final V2 candidate**: `E19_ref_a14_b22` (equivalent to `E16_T036_D050`),
saved as `nb04_bg_v2_final_animation_ready_8cycles.pkl`. Eight cardiac
cycles are recorded after a five-cycle warm-up.

#### What is fixed vs the earlier NB01 parser

The older parser used:

```python
h = 0.10 * r
```

That made `h/r` constant and collapsed wave speed into only a few discrete bands.
The V2 notebook reads the wall law from `BG_Modules_v2.cellml`:

```python
h = r * (a*exp(b*r) + c*exp(d*r))
```

For the uploaded BG module, the constants are:

```text
a =  0.2802
b = -505.3
c =  0.1324
d =  -11.14
```

This gives radius-dependent wall thickness and much more realistic peripheral
pulse-wave speed (`c₀ = sqrt(β · √A₀ / (2ρ))`), which is exactly what feeds the
brachial waveform shape and the reflections at every bifurcation.

---

## 5. Numerical assumptions

| Item | Value / choice |
|---|---|
| Equations | 1D continuity + momentum for compliant tubes |
| Scheme | MacCormack predictor-corrector on vessel interiors |
| CFL safety factor | 0.10 in the V1 baseline, tighter in some V2 experiments |
| Wall law | `P = β · (√A − √A₀)` |
| Wave speed | `c₀ = sqrt(β · √A₀ / (2ρ))` (solver-consistent), Moens-Korteweg kept as a diagnostic |
| Wall thickness | V1: `h = 0.10·r`; V2: BG exponential law from `BG_Modules_v2.cellml` |
| Junction coupling | Local impedance with a shared junction pressure |
| Terminal BC | 3-element Windkessel (R1, C, R2), characteristic version uses local `Z` |
| Root BC | Prescribed inflow waveform, pressure back-estimated via Riemann compatibility |
| Initial state | Uniform reference pressure (`P_REF ≈ 90 mmHg`), inverse wall law for area |
| Stabilizer | Area clipped to `0.85·A_ref ≤ A ≤ 1.35·A_ref`; velocity clipped |
| Flow damping | Optional radius-scaled phenomenological damping on `Q` (V2 uses ~0.50) |
| Reference pressure for color | Per-candidate `p_vmin`/`p_vmax`, falling back to 5th/95th percentile of the recorded pressure field |

The damping term and the area clipper are explicitly **numerical safety mechanisms**,
not physiological components.

---

## 6. Problems encountered and how they were handled

These are the issues that actually shaped the notebooks. 

- **Wall-thickness collapse (V1 → V2).** The constant `h = 0.10·r` made every
  vessel's `h/r` identical, which collapsed the wave-speed map into a handful
  of discrete bands and made peripheral reflections wrong. This is the single
  most important fix and is the reason V2 exists.
- **Over-constrained root.** Forcing both `P` and `Q` at the aortic root made
  the hyperbolic system unstable. Resolved by prescribing only `Q_in(t)` and
  back-estimating `P_in` from the first interior cell via a characteristic
  relation.
- **Cycle drift on the morphology branch.** The `fast_characteristic` junction
  + Windkessel branch gives a much more realistic dicrotic region than the
  pure baseline, but at default Windkessel resistance it drifts downward
  cycle-to-cycle. The drift sign reverses at higher resistance. V2 Experiment
  03 sweeps `R_scale` and finds `R_scale ≈ 1.10` as the cycle-balanced point.
  V1 instead added drift-aware scoring to Optuna Study E.
- **Jagged notch from inlet tampering.** Trying to create a dicrotic notch by
  modifying the inlet reverse-flow / shoulder terms produced sharp,
  artificial spikes (V2 Experiments 05-06). Switching to terminal-reflection
  tuning (`r1_frac`, Experiment 07) and gentle systolic-duration sweeps
  (Experiment 15-16) gave a cleaner shoulder.
- **Pressure too high.** With the BG wall law the V2 candidate sat around
  133/88 mmHg. Experiments 08-12 walked it down toward ~124/80 mmHg using
  `R_scale` and `STROKE_VOLUME_ML` while monitoring 6-cycle drift.
- **Python overhead in the time loop.** Plain Python with per-vessel arrays
  was too slow for the long Optuna and multi-cycle runs. NB04 progressively
  flattened state to `A_flat`/`Q_flat`/`offsets` and moved MacCormack,
  stabilizer, junctions, and Windkessel into Numba kernels. Some Numba
  attempts (typed-list Windkessel) were rejected as net-slower and are kept
  in the notebook as documented dead ends.
- **Snapshot ↔ waveform misalignment in the video.** Earlier animation drafts
  drew the lower-panel waveform from a separately saved per-vessel time
  series. Small recording offsets produced flat-line gaps. NB06 now derives
  the brachial trace directly from the saved pressure snapshots so the
  animation and the side panel stay in sync. The default `cycle_loop` mode
  repeats one well-behaved cardiac cycle across all 20 s.

---

## 7. Libraries

- **NumPy / SciPy / pandas** — array math, parameter handling, light analysis.
- **Numba** — JIT-compiled flat kernels for MacCormack, stabilizer, junctions
  and Windkessel. The flattened state representation
  (`A_flat`, `Q_flat`, `offsets`, `lengths`) is what makes the long runs and
  the Optuna sweeps feasible. See `requirements.txt` for the exact version.
- **Optuna** — V1-only. Used in `05_brachial_optimization.ipynb` for the
  staged search (Study A broad → B exploitation → C local refinement →
  D regional stiffness → E long-run / drift-aware). The chosen candidate is
  Study E, trial 4.
- **NetworkX** — topology bookkeeping during parsing.
- **Matplotlib** — animation rendering (`FuncAnimation` + `FFMpegWriter`),
  `LineCollection` for the vessel-by-vessel pressure coloring, `plasma`
  colormap.
- **FFmpeg** — required as a real binary on `PATH` for MP4 export
  (`pip install ffmpeg` alone is not enough — see the notice in NB06).

Full pinned environment in `requirements.txt`.

---

## 8. How to reproduce

1. Create the venv and install: `pip install -r requirements.txt`. Make sure a
   real `ffmpeg.exe` is on `PATH`.
2. Run **either** track end-to-end:
   - **V1**: `01_parse_adan_cellml_v1` → `02_baseline_cpu_solver_v1` →
     `03_characteristic_cpu_solver_v1` → `04_speedup_solver_numba_v1` →
     `05_brachial_optimization` (this produces `optuna_E_best_trial_4_long100.pkl`).
   - **V2**: `01_parse_adan_cellml_v2_BG_fixed` →
     `02_baseline_cpu_solver_v2_BG_adapted` →
     `03_characteristic_cpu_solver_v2_BG_adapted` →
     `04_speedup_solver_numba_v2` (this produces
     `nb04_bg_v2_final_animation_ready_8cycles.pkl`).
3. Open `06_final_animation_export.ipynb`, set
   `ACTIVE_CANDIDATE` to `"candidate_A_v1_8cycles"` or
   `"candidate_B_bg_v2_8cycles"`, and run all cells. The HTML preview cell
   renders inside the notebook; the final cell writes the MP4 into
   `results/video_exports/`.

---

## 9. Acknowledgements

This project uses the **ADAN-86** arterial network model, distributed publicly
in CellML form through the **Physiome Model Repository** (workspace `4ac`).
The wall-thickness law in V2 is taken from the bond-graph modules
(`BG_Modules_v2.cellml`) published alongside the model.

Reference paper:

> Safaei S., Blanco P. J., Müller L. O., Hellevik L. R., Hunter P. J. (2016).
> *Bond graph model of cerebral circulation: toward clinically feasible
> systemic blood flow simulations.* The Journal of Physiology / Frontiers
> work associated with the ADAN-86 release.
> Physiome Model Repository workspace `4ac`.

The CellML files (`main_ADAN-86.cellml`, `Parameters86.cellml`,
`BG_Modules_v2.cellml`) included here are redistributed under the licensing
terms of the upstream Physiome workspace. All credit for the anatomical
model and the wall-law constants belongs to the original ADAN-86 authors.
The solver, the Numba acceleration, the Optuna tuning, and the animation
code in this repository are original work built on top of that model.
