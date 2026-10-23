# HPC Workflow Status

A config-driven Python monitoring system for Rocoto-based HPC workflows with a GitHub Pages dashboard, email alerting, and multi-layer failure detection.

## Features

- **Untracked [`myexps.yml`](config.yml) user configuration** — copy the tracked template [`config.yml`](config.yml) to `myexps.yml` (git-ignored) so sensitive information (email addresses, local experiment paths) stays out of GitHub and `git pull` never conflicts.
- **Full `rocotostat` structured parsing** — uses `rocotostat -s` to discover all `Active` cycles + the last $N$ `Done` cycles (works identically for **realtime** and **retrospective** workflows), then parses full task state and cycle wall-clock duration.
- **Dead job detection** — alerts on new `DEAD` jobs with MD5 deduplication (no repeated emails for the same failure).
- **Stall detection** — alerts when no jobs are running/queued/submitting beyond a configurable threshold.
- **Hung job detection** — optionally checks one or more `RUNNING` task types (`fcst`, `jedivar`, etc.) for stale log files, with optional `cancel_and_reboot` auto-remediation.
- **Ensemble forecast failure auto-rescue (`ens_fcst_rescue`)** — automatically rescues failed ensemble forecast tasks (`fcst_m*`) in active cycles:
  - Triggers only when $\le 5$ (configurable via `max_dead_fcst`) dead ensemble forecast tasks are detected in an active cycle.
  - Submits a lightweight Slurm job (spec from `<task name="prep_ic">` in workflow XML) running [`ens_fcst_rescue.sh`](ens_fcst_rescue.sh).
  - Preserves the existing `prep_ic/enkf/mem*/mpasout.nc` by renaming it to `bad.mpasout.nc` for debugging.
  - Copies the pure 1h forecast background `mpasout.*.nc` from the previous cycle to the current cycle's `prep_ic` member directory.
  - Cleans leftover files in the member's `fcst` directory to avoid corrupted restarts.
  - Rewinds the dead task via `rocotorewind` so `rocotorun` resubmits it cleanly.
  - Rescues each `(cycle, fcst_mXXX)` at most once (guaranteed by an `ens_fcst_rescue.done` marker).
  - Sends alert emails containing the rescue batch job ID, output log path, and saved `bad.mpasout.nc` path.
- **Zero-conflict per-HPC branches (`status-<MACHINE>`)** — each HPC pushes `<exp>.json` directly to its own dedicated branch `status-<MACHINE>` (`https://raw.githubusercontent.com/<owner>/<repo>/status-<machine>/<exp>.json`) without modifying the working tree or `main` branch.
- **Three-layer failure detection**:
  1. **healthchecks.io heartbeat** — detects monitor or cluster/scrontab death within ~15 min.
  2. **GitHub Actions watchdog** ([`.github/workflows/stale-check.yml`](.github/workflows/stale-check.yml)) — opens a GitHub Issue if any `<exp>.json` on any `status-*` branch goes $>30$ min stale.
  3. **Dashboard UI** ([`docs/index.html`](docs/index.html)) — color-coded staleness indicator visible at a glance.

## Prerequisites

1. **Git SSH access** (`git@github.com:...`) configured on the HPC cluster (with an SSH key that does not prompt for an interactive passphrase when running under `scrontab`).
2. **`pyDAmonitor` Conda environment** (`Miniforge3/envs/pyDAmonitor/bin/python3`), invoked directly by [`run.sh`](run.sh) without needing `conda activate`.
3. **Rocoto** module on the target HPC cluster (automatically loaded by [`run.sh`](run.sh)).

## Supported HPC Systems (`MACHINE` values)

| `MACHINE` | Status Branch | Rocoto Module Path | `pyDAmonitor` Base Directory (`Miniforge3`) |
| :--- | :--- | :--- | :--- |
| **`gaeac6`** | `status-gaeac6` | `/gpfs/f6/arfs-gsl/world-shared/gge/rocoto/modulefiles` | `/gpfs/f6/bil-fire10-oar/world-shared/gge/Miniforge3` |
| **`gaeac7`** | `status-gaeac7` | `/gpfs/f7/arfs-gsl/world-shared/gge/rocoto/modulefiles` | `/gpfs/f7/wrfruc/world-shared/gge/Miniforge3` |
| **`hera`** | `status-hera` | `/scratch4/BMC/zrtrr/gge/rocoto_hera/modulefiles` | `/scratch3/BMC/wrfruc/hera/Miniforge3` |
| **`ursa`** | `status-ursa` | `/scratch4/BMC/zrtrr/gge/rocoto/modulefiles` | `/scratch3/BMC/wrfruc/gge/Miniforge3` |
| **`orion`** | `status-orion` | `/work/noaa/zrtrr/gge/rocoto/modulefiles` | `/work/noaa/zrtrr/gge/Miniforge3` |
| **`hercules`** | `status-hercules` | `/work/noaa/zrtrr/gge/hercules/rocoto/modulefiles` | `/work/noaa/zrtrr/gge/hercules/Miniforge3` |
| **`derecho`** | `status-derecho` | `/glade/work/geguo/rocoto/modulefiles` | `/glade/work/geguo/Miniforge3` |

## Quick Start

### 1. Clone the Repo via SSH on HPC

```bash
git clone git@github.com:guoqing-noaa/workflow_status.git
cd workflow_status
```

### 2. Copy `config.yml` to `myexps.yml` and Edit

Copy the example template [`config.yml`](config.yml) to `myexps.yml` (which is ignored by git so sensitive information stays local) and configure `common:` defaults and your `experiments:` list:

```bash
cp config.yml myexps.yml
```

```yaml
common:
  default_exp: ursa/rrfsv2x_det
  workflow_xml: rrfs.xml
  workflow_db: rrfs.db
  lookback_cycles: 72
  recipients:
    - first.last@noaa.gov
  checks:
    dead_jobs:
      enabled: true
    stall:
      enabled: true
      threshold_sec: 3600

experiments:
  - name: rrfsdet_rt
    expdir: /gpfs/f7/arfs-gsl/world-shared/gge/rrfs2/OPSROOT/conus12km/exp/rrfsdet
    subject_prefix: rrfsv2x_rt
```

*(Optional)* For `healthchecks.io` dead-man's-switch monitoring, put your UUID in an untracked `healthchecks_uuid.txt` file at the repo root (git-ignored):

```bash
echo 'YOUR-UUID-HERE' > healthchecks_uuid.txt
chmod 600 healthchecks_uuid.txt
```

### 3. Test (`--dry-run`)

By default, [`run.sh`](run.sh) reads `myexps.yml` in the repo root, or you can specify any custom `.yml` config file on the command line:

```bash
# Use default myexps.yml
MACHINE=gaeac7 ./run.sh --dry-run

# Or specify a custom YAML config file
MACHINE=gaeac7 ./run.sh custom_exps.yml --dry-run
```

### 4. Ensemble Forecast Auto-Rescue Configuration (`ens_fcst_rescue`)

To enable automatic rescue of dead ensemble forecast tasks (`fcst_m*`), add `ens_fcst_rescue:` to the experiment or shared defaults:

```yaml
experiments:
  - name: rrfsv2x_enkf
    expdir: /scratch3/BMC/rtwrfruc/RRFSv2X/NCO_dirs/RRFSv2X/exp/rrfsenkf
    subject_prefix: rrfsv2x_enkf
    checks:
      ens_fcst_rescue:
        enabled: true
        max_dead_fcst: 5    # Max dead members in cycle before giving up (default: 5)
        mem: 64G            # Memory allocated for the rescue copy job (default: 64G)
```

#### Standalone / Rescue-Only Mode
If you want to run `run.sh` exclusively for ensemble forecast rescue without writing dashboard `.state` JSON, updating git branches, or sending dead/stall alert emails:

1. Set `rescue_only: true` in `myexps.yml` under `common:` or at the top level:
   ```yaml
   common:
     rescue_only: true
   ```
   *(Or disable all other checks with `dead_jobs: enabled: false`, `stall: enabled: false`, `hung_jobs: enabled: false` while keeping `ens_fcst_rescue: enabled: true`).*

2. Run with or without `--dry-run`:
   ```bash
   MACHINE=ursa ./run.sh --dry-run
   ```

### 5. Add to `scrontab`

```
#SCRON --partition=cron_c7
#SCRON --account=arfs-gsl
#SCRON --time=00:10:00
#SCRON --mem=8G
#SCRON --dependency=singleton
#SCRON --job-name=workflow_status
#SCRON --output=/dev/null
*/10 * * * * MACHINE=gaeac7 /path/to/workflow_status/run.sh
# Or with a custom config file:
# */10 * * * * MACHINE=gaeac7 /path/to/workflow_status/run.sh /path/to/custom_exps.yml
```

### 6. Multiple Experiment Groups on One Machine

Several config files can be run independently on the same HPC (e.g. `run.sh groupA.yaml` and
`run.sh groupB.yaml`). Each config is a *group*, named by a top-level or `common:` `group:` key
(default: the config file name without extension). Every `<exp>.json` carries its `group`, and
`_index.json` on `status-<MACHINE>` maps each file to its group. A push replaces only the files of its
own group(s) and keeps the others (guarded by `--force-with-lease` with retry), so groups never
overwrite each other. The dashboard sidebar shows a header per group when more than one is present.

Files of a group that is no longer run stay on the branch (only a push from their own group removes
them). To retire a group, set `experiments: []` in its config and run it once; that push removes all
of the group's files. Then drop the config.

## Repository Structure

```
workflow_status/
├── README.md
├── config.yml                   # Example config template (copy to myexps.yml)
├── myexps.yml                   # Untracked local user config (git-ignored)
├── run.sh                       # Thin launcher (uses MACHINE to set Rocoto + pyDAmonitor python3)
├── workflow_status.py           # Core monitoring engine (pushes <exp>.json to branch status-<MACHINE>)
├── ens_fcst_rescue.sh           # Slurm rescue script (copies 1h forecast, cleans dir, rocotorewind)
├── docs/                        # GitHub Pages dashboard (reads status-* branches across all HPCs)
│   └── index.html
└── .github/
    └── workflows/
        └── stale-check.yml      # GitHub Actions 30-min staleness watchdog across status-* branches
```
