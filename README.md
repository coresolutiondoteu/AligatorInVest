## AligatorInVest
# BAD USB Investigator Script for CAINE

## Full Automation & Investigation Suite

A single entry-point script that provisions its own tools, runs a menu-driven workflow end-to-end, shows **progress bars with ETA**, uses **multi-threading** where possible, logs everything for audit, and produces reproducible outputs. Designed for long-running data/forensics and DevOps tasks on Linux.

---

## Key capabilities

* **Self-bootstrap**

  * Verifies OS, shell, privileges.
  * Installs missing dependencies on Debian/Ubuntu/RHEL/Arch families.
  * Creates required folders and default config.

* **Menu-driven orchestration**

  * Clean, numbered steps with safe defaults.
  * Can run a single step or the entire pipeline.
  * Persists state to support resume.

* **Progress bars + timing**

  * Live progress bars for file and job operations.
  * Shows elapsed, ETA, throughput, CPU usage.
  * Per-step and total runtime summary.

* **Parallel execution**

  * Auto-detects cores and uses them efficiently.
  * Per-step concurrency controls and CPU affinity.
  * Work-stealing queues for uneven task sizes.

* **Data integrity**

  * Hashes (SHA-256/512) before and after operations.
  * Optional double-or triple-pass verification.
  * Deterministic output naming and manifests.

* **Robust logging**

  * Structured logs (`jsonl`) + human logs (`.log`).
  * Timestamps with time zone and monotonic clock.
  * Redaction of secrets in console output.

* **Idempotent + safe**

  * Dry-run mode shows exact commands and plan.
  * Atomic writes and temp file quarantine.
  * Checkpoints and resume after interruption.

* **Artifacts**

  * Uniform, versioned output structure.
  * Summary reports (HTML/Markdown/CSV).
  * Optional signed tarballs with checksums.

---

## Typical workflow (default menu)

1. **Environment check**
   OS, kernel, disk space, RAM, packages, locales, time sync.

2. **Bootstrap dependencies**
   Installs: `pv`, `pigz`, `jq`, `xz-utils`, `coreutils`, `parallel`, `numactl`, `rsync`, `openssl`, `parted`, `smartmontools`, `lsblk`, `util-linux`, `fio` (optional).

3. **Source discovery**
   Lists devices, mounts, or input paths. Smart filters to avoid system disks. Optional SMART health.

4. **Snapshot / read-only prep**
   Sets block devices RO, remounts ro when possible. Captures metadata (partition tables, UUIDs).

5. **Acquisition / copy**
   Block-level or file-level capture with live progress and throttling. Hash on the fly.

6. **Multi-threaded processing**
   Parallel compression, hashing, format conversion, and transforms. CPU affinity to spread across cores. This fixes the previous single-core bottleneck.

7. **Verification**
   Re-hash, compare manifests, optional second tool cross-check. Writes verification report.

8. **Packaging + reports**
   Creates signed, checksummed bundles. Generates HTML/Markdown summaries and CSV manifests.

9. **Cleanup + summary**
   Clears temp, prints timing table, outputs next-step hints.

> You can run any step directly or run all steps in sequence. Failed steps can be retried without redoing completed work.

---

## Requirements

* Linux with Bash 5+
* `sudo` privileges for device operations
* Disk space ≥ 2× expected artifact size
* Internet access for first-time dependency install (optional if you preinstall)

---

## Installation

```bash
git clone <your-repo-url> automation-suite
cd automation-suite
chmod +x full.sh
# Optional: pin GNU parallel to a recent version
```

No system files are modified beyond package installation unless you enable optional system tuning.

---

## Quick start

```bash
# See options
./full.sh --help

# Full guided run with menu
./full.sh

# Non-interactive run of selected steps
./full.sh run --steps 1,2,5-8

# Dry run to preview commands
./full.sh --dry-run run --steps all
```

---

## CLI usage

```text
full.sh [global options] <command> [command options]

Global options:
  --dry-run           Plan only. Do not change the system.
  --config PATH       Use a specific config file.
  --log-level LEVEL   error|warn|info|debug (default: info)
  --no-color          Disable ANSI colors.
  --yes               Assume “yes” to prompts.

Commands:
  check               Run environment checks.
  bootstrap           Install or verify dependencies.
  menu                Start interactive TUI.
  run                 Execute steps non-interactively.
  resume              Continue from last checkpoint.
  clean               Remove temp and caches.
  report              Regenerate reports from manifests.

run options:
  --steps LIST        e.g. 1,2,5-8 or “all”
  --threads N         Override CPU auto-detect.
  --throttle MBPS     Cap IO throughput.
  --hash ALGO         sha256|sha512 (default: sha256)
  --compress TYPE     none|gz|xz|zstd (default: zstd)
```

---

## Configuration

Create `config/default.env` or pass `--config`:

```env
# Input and output
INPUT_PATH=/data/input
OUTPUT_ROOT=/data/artifacts
WORK_DIR=/data/tmp

# Performance
THREADS=auto
IO_THROTTLE_MBPS=0          # 0 = unlimited
CPU_AFFINITY=auto           # auto|none|cpulist (e.g., 0-7)

# Integrity
HASH_ALGO=sha256
VERIFY_PASSES=2             # 1..3
CHUNK_SIZE_MB=512

# Packaging
COMPRESS=zstd               # none|gz|xz|zstd
SPLIT_SIZE_MB=0             # 0 = no split
SIGN_ARTIFACTS=false

# Logging
LOG_DIR=./logs
LOG_JSON=true
REDACT_SECRETS=true

# Safety
READ_ONLY_MODE=true
FAIL_FAST=false
```

---

## Output layout

```
artifacts/
  YYYYMMDD-HHMMSS-runID/
    01_meta/
      env.json
      devices.json
      checks.json
    02_raw/
      source-*.img
      source-*.sha256
    03_proc/
      *.zst
      *.sha256
    04_reports/
      summary.md
      summary.html
      manifest.csv
    99_bundle/
      bundle.tar.zst
      bundle.tar.zst.sha256
logs/
  run-YYYYMMDD-HHMMSS.log
  run-YYYYMMDD-HHMMSS.jsonl
```

---

## Progress bars and timing

* File operations use `pv` for byte-accurate progress and ETA.
* Multi-file jobs show per-task and aggregate bars.
* A timing table prints at the end with wall time and CPU time per step.

Example:

```text
[5/8] Processing: 7 tasks | 4/7 done | 2m13s elapsed | ETA 1m05s
  task-03  ████████████▋  87%  220 MB/s  ETA 0m12s
  task-04  ████████▍      52%  205 MB/s  ETA 0m41s
```

---

## Parallelism

* Uses `parallel` and `pigz`/`pzstd` where available.
* Auto-sets `--jobs` to `min(cores, tasks)`.
* Optionally pins workers with `taskset/numactl` to reduce contention.
* Ensures deterministic outputs across runs by stable task ordering.

---

## Safety model

* Does not write to source devices unless you disable `READ_ONLY_MODE`.
* Atomic temp files: writes to `WORK_DIR` then moves into place.
* Hash manifests stored alongside artifacts.
* On failure: records checkpoint and safe to resume.

---

## Logs and audit trail

* Human log: concise, colorized.
* Machine log: `jsonl` with fields: `ts`, `step`, `op`, `src`, `dst`, `bytes`, `hash`, `rc`, `duration_ms`.
* All commands recorded. Sensitive values masked.

---

## Extending the pipeline

* Add a new step under `steps/NN_name.sh`.
* Register in `steps/index.sh` to include in ordering.
* Steps get helpers: logging, progress, hashing, concurrency, retries.

Helper examples:

```bash
log_info "message"
with_progress dd if=... of=...
run_parallel 8 my_worker ::: "${items[@]}"
hash_file sha256 path/to/file
checkpoint save 06-processing
```

---

## Troubleshooting

* **Single-core behavior**
  Set `THREADS=auto` or pass `--threads $(nproc)` and ensure `parallel` is installed.

* **No progress bars**
  Install `pv`. For pipes without size, ETA is approximate.

* **Permission denied on block devices**
  Use `sudo` and confirm `READ_ONLY_MODE=true` for safe reads.

* **Low throughput**
  Disable laptop power saving. Increase `CHUNK_SIZE_MB`. Avoid slow external hubs. Check `iotop` and `dstat`.

* **Hash mismatch**
  Re-run verification. If persistent, isolate storage or RAM issues with `memtest` and `smartctl`.

---

## Non-interactive examples

```bash
# Full run with max compression and 2 verifies
./full.sh run --steps all --compress zstd --threads auto --hash sha512

# Acquisition only at 150 MB/s throttle
./full.sh run --steps 3-5 --throttle 150

# Rebuild reports from existing manifests
./full.sh report --config artifacts/20250905-1030-ABCD/01_meta/env.json
```

---

## Security notes

* Set `SIGN_ARTIFACTS=true` to sign bundles with OpenSSL CMS.
* Logs may contain filenames and sizes. Keep `logs/` secured.
* For network-less runs, preinstall dependencies and use `--dry-run` to verify plan.

---

## Versioning

* Script prints a semantic version `vX.Y.Z` at start.
* Each artifact folder embeds the version and git commit hash for provenance.

---

## License

Choose a license suitable for your use. Example: MIT for code, separate notice for bundled third-party tools.

---

## Credits

Uses standard GNU/Linux tools, `pv`, `parallel`, `pigz/pzstd`, `jq`, `openssl`, and `coreutils`. No vendor lock-in.

---

## Quick checklist

* [ ] `./full.sh check`
* [ ] Configure `config/default.env`
* [ ] `./full.sh bootstrap`
* [ ] `./full.sh menu` or `./full.sh run --steps all`
* [ ] Verify `04_reports/summary.html`
* [ ] Archive `99_bundle/` to your long-term storage
