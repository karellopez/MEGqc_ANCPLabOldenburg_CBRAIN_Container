# MEGqc — Container Testing & CLI Reference

> **Assumes** `MEGqc.sif` is already built (see `TUTORIAL_Build_Apptainer_Image.md`).  
> All commands are run from the `tutorial/` folder:
> ```bash
> cd /path/to/MEGqc_ANCPLabOldenburg_CBRAIN_Container/tutorial
> ```

---

## Quick-reference: path conventions

| Host path | Container mount | Purpose |
|-----------|----------------|---------|
| `./settings.ini` | `/mnt_config/settings.ini` | Config file |
| `/path/to/bids_dataset` | `/mnt_IN` | Input BIDS dataset |
| `./outputs` | `/mnt_OUT` | Output derivatives (external) |

> **Tip:** without `--derivatives_output` the results land inside the BIDS
> dataset at `<dataset>/derivatives/Meg_QC/`. Using `--derivatives_output`
> (→ `/mnt_OUT`) keeps the source dataset untouched.

---

## 0. Before you start — export the config

```bash
apptainer exec \
  --bind "$PWD":/mnt_config \
  MEGqc.sif \
  get-megqc-config --target_directory /mnt_config
```

Edit `settings.ini` as needed. Key settings for quick testing:

```ini
[GENERAL]
data_crop_tmin = 0
data_crop_tmax = 10   # crop to 10 s for fast iteration
```

---

## 1. Full profiled run — the recommended way

This is the **primary workflow**: calculation + all QA and QC reports in one
shot, results stored in a timestamped profile under `outputs/`.

```bash
mkdir -p outputs

apptainer run \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/settings.ini":/mnt_config/settings.ini \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  --inputdata /mnt_IN \
  --config /mnt_config/settings.ini \
  --derivatives_output /mnt_OUT \
  --analysis_mode new \
  --run-all --all
```

The analysis profile ID is auto-generated (`YYYYMMDD_HHMMSS`).  
Results: `outputs/mnt_IN/derivatives/Meg_QC/profiles/<analysis_id>/`

---

## 2. Single-subject test run

Fastest way to verify the container works on your data:

```bash
apptainer run \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/settings.ini":/mnt_config/settings.ini \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  --inputdata /mnt_IN \
  --config /mnt_config/settings.ini \
  --derivatives_output /mnt_OUT \
  --subs 009 \
  --analysis_mode new \
  --run-all --qa-subject
```

`--qa-subject` only generates per-subject HTML reports (no group statistics),
which is much faster and sufficient for a smoke test.

---

## 3. Multiple specific subjects

```bash
apptainer run \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/settings.ini":/mnt_config/settings.ini \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  --inputdata /mnt_IN \
  --config /mnt_config/settings.ini \
  --derivatives_output /mnt_OUT \
  --subs 009 012 015 \
  --analysis_mode new \
  --run-all --all
```

---

## 4. Parallel multi-subject run

```bash
apptainer run \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/settings.ini":/mnt_config/settings.ini \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  --inputdata /mnt_IN \
  --config /mnt_config/settings.ini \
  --derivatives_output /mnt_OUT \
  --n_jobs 4 \
  --analysis_mode new \
  --run-all --all
```

> **RAM guide:** 8 GB → 1 job · 16 GB → 2 jobs · 32 GB → 6 jobs · 64 GB → 16 jobs

---

## 5. Multiple datasets in one call

```bash
apptainer run \
  --bind /path/to/ds_1:/mnt_ds1 \
  --bind /path/to/ds_camcan:/mnt_ds2 \
  --bind "$PWD/settings.ini":/mnt_config/settings.ini \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  --inputdata /mnt_ds1 /mnt_ds2 \
  --config /mnt_config/settings.ini \
  --derivatives_output /mnt_OUT \
  --analysis_mode new \
  --run-all --all
```

---

## 6. Per-dataset config and subject overrides

```bash
apptainer run \
  --bind /path/to/ds_1:/mnt_ds1 \
  --bind /path/to/ds_camcan:/mnt_ds2 \
  --bind "$PWD/settings_ds1.ini":/mnt_cfg1/settings.ini \
  --bind "$PWD/settings_ds2.ini":/mnt_cfg2/settings.ini \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  --inputdata /mnt_ds1 /mnt_ds2 \
  --config_per_dataset \
      /mnt_ds1::/mnt_cfg1/settings.ini \
      /mnt_ds2::/mnt_cfg2/settings.ini \
  --subs_per_dataset \
      /mnt_ds1::009,012,015 \
      /mnt_ds2::all \
  --derivatives_output /mnt_OUT \
  --analysis_mode new \
  --run-all --all
```

---

## 7. Analysis modes

| Mode | When to use |
|------|-------------|
| `new` | Default — creates a new timestamped profile |
| `reuse` | Re-run plotting on an existing profile (needs `--analysis_id`) |
| `latest` | Auto-resolves the most recently modified profile |
| `legacy` | Classic path (`derivatives/Meg_QC/`), no profile subfolder |

### Reuse an existing profile (add more plots)

```bash
apptainer run \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/settings.ini":/mnt_config/settings.ini \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  --inputdata /mnt_IN \
  --config /mnt_config/settings.ini \
  --derivatives_output /mnt_OUT \
  --analysis_mode reuse \
  --analysis_id 20260318_094626 \
  --run-all --all
```

### Legacy mode (flat derivatives, no profile)

```bash
apptainer run \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/settings.ini":/mnt_config/settings.ini \
  MEGqc.sif \
  --inputdata /mnt_IN \
  --config /mnt_config/settings.ini \
  --analysis_mode legacy \
  --run-all --all
```

---

## 8. run-megqc-plotting — plotting only (no recalculation)

Useful when calculations are already done and you only want to re-generate
reports with different plot settings.

```bash
apptainer exec \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  run-megqc-plotting \
  --inputdata /mnt_IN \
  --derivatives_output /mnt_OUT \
  --analysis_mode latest \
  --all
```

### Subject reports only

```bash
apptainer exec \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  run-megqc-plotting \
  --inputdata /mnt_IN \
  --derivatives_output /mnt_OUT \
  --analysis_mode latest \
  --qa-subject
```

### Group reports only

```bash
apptainer exec \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  run-megqc-plotting \
  --inputdata /mnt_IN \
  --derivatives_output /mnt_OUT \
  --analysis_mode latest \
  --qa-group
```

### QC group + multisample

```bash
apptainer exec \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  run-megqc-plotting \
  --inputdata /mnt_IN \
  --derivatives_output /mnt_OUT \
  --analysis_mode latest \
  --qc-group --qc-multisample
```

---

## 9. globalqualityindex — GQI summary

```bash
apptainer exec \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  globalqualityindex \
  --inputdata /mnt_IN \
  --derivatives_output /mnt_OUT \
  --analysis_mode latest
```

---

## 10. Plotting flags reference

| Flag | Scope | Reports generated |
|------|-------|-------------------|
| `--qa-subject` | Per subject | Subject-level HTML reports |
| `--qa-group` | Per dataset | Group-level QA summary |
| `--qa-multisample` | Across datasets | Cross-dataset QA (≥ 2 datasets required) |
| `--qa-all` | All QA | Subject + Group + Multisample |
| `--qc-group` | Per dataset | Group-level QC summary |
| `--qc-multisample` | Across datasets | Cross-dataset QC (≥ 2 datasets required) |
| `--qc-all` | All QC | Group + Multisample |
| `--all` | Everything | All QA + all QC |

> Add `--run-all` to `run-megqc` to invoke plotting immediately after
> calculation finishes (equivalent to calling `run-megqc-plotting --all`).

---

## 11. Subject / processed data policies

### Skip already-processed subjects (default)

```bash
  --processed_subjects_policy skip
```

### Force reprocessing of all subjects

```bash
  --processed_subjects_policy rerun
```

### Use the saved config from the profile (not the one you provide)

```bash
  --existing_config_policy latest_saved
```

---

## 12. Error handling / debugging

### Keep temp files on error

```bash
  --keep-temp-on-error
```

Intermediate `.tmp` files are preserved in the profile folder for inspection.

### Run entirely headless (explicit)

```bash
apptainer exec \
  --containall \
  --env QT_QPA_PLATFORM=offscreen \
  --env MPLBACKEND=Agg \
  --bind /path/to/bids_dataset:/mnt_IN \
  --bind "$PWD/settings.ini":/mnt_config/settings.ini \
  --bind "$PWD/outputs":/mnt_OUT \
  MEGqc.sif \
  run-megqc \
  --inputdata /mnt_IN \
  --config /mnt_config/settings.ini \
  --derivatives_output /mnt_OUT \
  --analysis_mode new \
  --run-all --all
```

---

## 13. Launch the GUI (desktop only)

```bash
apptainer exec \
  --env DISPLAY=$DISPLAY \
  --env QT_QPA_PLATFORM=xcb \
  --env XDG_RUNTIME_DIR=$XDG_RUNTIME_DIR \
  --env DBUS_SESSION_BUS_ADDRESS="$DBUS_SESSION_BUS_ADDRESS" \
  --env XAUTHORITY=$XAUTHORITY \
  --bind /tmp/.X11-unix:/tmp/.X11-unix \
  --bind /etc/machine-id:/etc/machine-id:ro \
  --bind /run/user/$(id -u):/run/user/$(id -u) \
  MEGqc.sif \
  megqc
```

> The GUI is **not available on headless HPC nodes**. The container defaults
> to `QT_QPA_PLATFORM=offscreen` for HPC safety; the GUI command above
> overrides that only for the GUI invocation.
>
> **Why shell variables instead of hard-coded values?**  
> On multi-user servers `$DISPLAY` is rarely `:0` (e.g. `:130.0`), and the
> D-Bus socket may live under `/tmp/` rather than `/run/user/<uid>/bus`.
> Using `$DISPLAY`, `$DBUS_SESSION_BUS_ADDRESS`, and `$XAUTHORITY` directly
> makes the command work correctly on both local desktops and shared servers.

---

## 14. CBRAIN / Slurm job script template

```bash
#!/bin/bash
#SBATCH --job-name=megqc
#SBATCH --time=04:00:00
#SBATCH --mem=32G
#SBATCH --cpus-per-task=6

module load apptainer   # or: singularity

BIDS=/scratch/$USER/bids_dataset
SIF=/scratch/$USER/containers/MEGqc.sif
CONFIG=/scratch/$USER/containers/settings.ini
OUT=/scratch/$USER/megqc_results

mkdir -p "$OUT"

apptainer run \
  --bind "${BIDS}":/mnt_IN \
  --bind "${CONFIG}":/mnt_config/settings.ini \
  --bind "${OUT}":/mnt_OUT \
  "${SIF}" \
  --inputdata /mnt_IN \
  --config /mnt_config/settings.ini \
  --derivatives_output /mnt_OUT \
  --n_jobs 6 \
  --analysis_mode new \
  --run-all --all
```

---

*Built and tested on Ubuntu 22.04 LTS · Apptainer 1.4.5 · meg_qc 0.7.8*  
*Maintainer: karel.mauricio.lopez.vilaret@uni-oldenburg.de*

