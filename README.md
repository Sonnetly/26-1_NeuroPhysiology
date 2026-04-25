# CA1 LFP & Sharp-Wave Ripple Analysis Pipeline

> ⚠️ **GitHub repository link:** `https://github.com/Sonnetly/26-1_NeuroPhysiology.git`

---

## Purpose

This repository contains an analysis pipeline for detecting and characterizing hippocampal **sharp-wave ripples (SWRs)** from CA1 local field potential (LFP) recordings, and computing **spike–ripple phase locking** of individual CA1 units.

The pipeline is developed in the context of a research proposal investigating how elevated homeostatic sleep pressure alters NREM spindle oscillatory quality (o-Quality) and SO–spindle–SWR coupling in memory-relevant circuits. Because no open sleep-specific Neuropixels dataset currently exists, this notebook serves as a **methodological validation** of the SWR detection and spike-phase locking components of the proposed pipeline, using the publicly available Allen Brain Observatory Visual Coding dataset.

### What the pipeline does

| Step | Description |
|---|---|
| Session selection | Identifies sessions with CA1 LFP coverage from the Allen Visual Coding Neuropixels dataset |
| Unit QC | Applies Siegle et al. (2021) quality criteria to CA1 spike-sorted units |
| LFP loading | Loads CA1 LFP and selects the optimal ripple channel by RMS power profile |
| CSD | Visualizes flash-evoked current source density for pyramidal layer localization |
| SWR detection | Detects SWRs using ripple-band (100–250 Hz) RMS envelope thresholding |
| SWR characterization | Computes duration, amplitude, and inter-event interval distributions |
| Spike–phase locking | Computes mean vector length (MVL) and Rayleigh test per CA1 unit |
| SWR-triggered average | Generates ripple waveform average aligned to SWR peak |
| Export | Saves event tables and summary statistics as CSV |

---

## Dataset

All data are downloaded automatically via the AllenSDK from the **Allen Brain Observatory — Visual Coding Neuropixels** dataset.

- **Source:** Siegle, J.H. et al. (2021). Survey of spiking in the mouse visual system reveals functional hierarchy. *Nature*, 592, 86–92. https://doi.org/10.1038/s41586-020-03171-x  
- **Access:** https://allensdk.readthedocs.io/en/latest/visual_coding_neuropixels.html  
- **AWS mirror:** https://registry.opendata.aws/allen-brain-observatory/  
- **Format:** Neurodata Without Borders (NWB), accessed via AllenSDK  
- **No manual download required** — the AllenSDK fetches data on first run and caches locally

**Storage requirement:** approximately 1–2 GB per session (LFP NWB files). Set `OUTPUT_DIR` in the notebook to a directory with sufficient space.

---

## Dependencies and Environment

### Recommended: Conda environment

```bash
conda create -n swr_pipeline python=3.10 numpy=1.26.4 ipykernel xarray matplotlib scipy=1.10.1 allensdk
conda activate swr_pipeline
```

This matches the environment specification from the course tutorial (Week 4 — Systems Electrophysiology II).

### Package versions

| Package | Version | Purpose |
|---|---|---|
| `python` | 3.10 | Base interpreter |
| `allensdk` | ≥ 2.15 | Allen Brain Observatory data access |
| `numpy` | 1.26.4 | Numerical arrays |
| `scipy` | 1.10.1 | Signal processing (Butterworth filter, Hilbert transform) |
| `pandas` | ≥ 1.5 | Event tables and dataframes |
| `matplotlib` | ≥ 3.6 | Visualization |
| `xarray` | ≥ 2022.11 | LFP DataArray handling (Allen SDK output format) |

### Manual install (pip alternative)

```bash
pip install allensdk numpy==1.26.4 scipy==1.10.1 pandas matplotlib xarray
```

> **Note:** AllenSDK currently requires `numpy < 2.0`. Use `numpy=1.26.4` as specified above to avoid compatibility issues.

### Jupyter

```bash
conda install jupyterlab
# or
pip install jupyterlab
```

---

## File Structure

```
ca1-swr-pipeline/
├── README.md                        ← this file
├── ca1_lfp_swr_pipeline.ipynb       ← main analysis notebook
└── outputs/                         ← generated on first run
    ├── swr_events.csv               ← detected SWR event table
    ├── spike_phase_locking_ca1.csv  ← per-unit MVL and Rayleigh p
    ├── ca1_units_qc.csv             ← QC-filtered CA1 unit table
    └── summary.csv                  ← session-level summary statistics
```

---

## Instructions for Reproducing the Analysis

### Step 1 — Clone the repository

```bash
git clone https://github.com/[your-username]/ca1-swr-pipeline.git
cd ca1-swr-pipeline
```

### Step 2 — Create and activate the environment

```bash
conda create -n swr_pipeline python=3.10 numpy=1.26.4 ipykernel xarray matplotlib scipy=1.10.1 allensdk
conda activate swr_pipeline
```

### Step 3 — Set the cache directory

Open `ca1_lfp_swr_pipeline.ipynb` and set `OUTPUT_DIR` in **Section 1** to a local path with ≥ 5 GB free space:

```python
OUTPUT_DIR = '/your/local/path/ecephys_cache_dir'
```

### Step 4 — Run the notebook

```bash
jupyter lab ca1_lfp_swr_pipeline.ipynb
```

Run all cells top to bottom (`Kernel → Restart & Run All`).  
On first run, AllenSDK will download the session manifest and LFP NWB files automatically. This may take **10–30 minutes** depending on connection speed.

### Step 5 — (Optional) Change session

To analyze a different session, modify `SESSION_ID` in Section 2:

```python
# Use index 0 for the first CA1 session, 1 for the second, etc.
SESSION_ID = ca1_sessions.index.values[0]
```

All downstream cells adapt automatically to the selected session.

---

## Key Analysis Parameters

| Parameter | Default | Location | Description |
|---|---|---|---|
| `OUTPUT_DIR` | `./ecephys_cache_dir` | Section 1 | Local Allen cache path |
| `SESSION_ID` | `ca1_sessions.index[0]` | Section 2 | Which session to analyze |
| `threshold_sd` | `3.0` | `detect_swrs()` | SWR envelope threshold (SD above mean) |
| `min_dur` | `0.025 s` | `detect_swrs()` | Minimum SWR duration |
| `max_dur` | `0.150 s` | `detect_swrs()` | Maximum SWR duration |
| `SWR_WINDOW_S` | `0.1 s` | Section 9 | Spike collection window around SWR peak (±) |

---

## Quality Control Criteria

Following Siegle et al. (2021) and the Allen Neuropixels white paper:

| Metric | Threshold | Direction | Meaning |
|---|---|---|---|
| `presence_ratio` | 0.80 | > | Unit detected in >80% of session |
| `isi_violations` | 0.50 | < | <50% refractory period contamination |
| `amplitude_cutoff` | 0.10 | < | <10% of spikes below detection threshold |

---

## SWR Detection Method

Ripple detection follows the method applied by Nitzan et al. (2022) to this same Allen dataset:

1. Bandpass CA1 LFP to 100–250 Hz (4th-order Butterworth SOS, zero-phase)
2. Compute RMS envelope with 10 ms sliding window
3. Threshold at mean + 3 SD of baseline envelope
4. Retain events with duration 25–150 ms
5. Peak defined as maximum absolute ripple-band amplitude within each event

**Reference:** Nitzan, N. et al. (2022). Propagation of hippocampal ripples to the neocortex by way of a subiculum-retrosplenial pathway. *Nature Neuroscience*, 25, 641–649. https://doi.org/10.1038/s41593-022-01026-8

---

## Connection to Research Proposal

This pipeline validates the **SWR detection and spike-phase locking** components described in Section 5 (Data Analysis Plan) of the midterm report. In the proposed experiment, these functions would be applied to CA1 LFP recorded during NREM sleep from mice undergoing sleep deprivation and recovery, rather than to awake visual-coding data. The detection algorithms, quality control thresholds, and phase-locking metrics are identical; only the input recordings and behavioral context differ.

---

## Acknowledgements

- Allen Institute for Brain Science — Visual Coding Neuropixels dataset and AllenSDK
- Course materials: Week 4 Systems Electrophysiology II
