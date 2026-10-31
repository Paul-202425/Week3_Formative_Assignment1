# Week3_Formative_Assignment
# Systems Programming and Forensics Assignment

**Author:** Paul Masamvu, African Leadership University
**Environment:** GNU/Linux x86-64 (Ubuntu 24.04), GCC 13.3.0, Python 3.13, `strace`, GNU binutils

Four projects investigate how programs meet the Linux operating system: the binary format on disk (Project 1), the system-call boundary (Project 2), native code inside Python (Project 3) and asynchronous signals (Project 4). Every result in this folder was produced by running the code in this repository. Nothing in the traces, benchmarks or demonstrations is invented.

## Quick map

| Project | Topic | Start here | Main evidence |
|---|---|---|---|
| 1 | ELF binary forensics | `project1_elf/Forensic_Assessment_Report.pdf` (one page) | `analysis.md` (Tasks 1-5), `outputs/` (raw command output) |
| 2 | System-call forensics with `strace` | `project2_strace/ANALYSIS.md` | `trace_success.txt`, `trace_failure.txt`, `trace_failure_permission.txt` |
| 3 | CPython C extension | `project3_cpython/REPORT.md` | `test_results.txt`, `benchmark_results.md`, `benchmark_speedup.png` |
| 4 | Signal-driven service | `project4_signals/Project4_Technical_Explanation.pdf` (one page) | `demo/run1_transcript.txt`, `demo/run2_transcript.txt` |

## Project 1: ELF Binary Forensics and Comparative Threat Assessment

**Scenario.** Two unfamiliar "file synchronisation" executables, `data_sync` and `data_sync_v2`, must be understood using static analysis only.

**Safety.** Both binaries were compiled from source I wrote, and **neither was ever executed**. `ldd` was not used (`run_analysis.sh` contains only static tools).

**What I did.**
- `src/data_sync.c` scans two directories and reports files missing from the target. `src/data_sync_v2.c` adds timestamped logging, size/metadata checks and an FNV-1a checksum.
- Compiled with `gcc -Wall -Wextra -O2` (v2 also `-g`), with zero warnings. Commands are in `outputs/00_compilation.txt`.
- `run_analysis.sh` runs `file`, `readelf`, `nm` and `objdump` on both binaries and stores each output in `outputs/<binary>/`.
- Three control builds (v2 without `-g`, stripped, and statically linked) show what each build option changes (`outputs/supplementary/`).

**Findings.** Both are 64-bit little-endian x86-64 PIE executables, dynamically linked against `libc.so.6` only. v2 imports seven extra functions (`fopen`, `fclose`, `fread`, `fputc`, `time`, `localtime`, `strftime`) and has three extra functions (`checksum_file`, `differs`, `log_msg`), which matches its added logging and checksum features. Only v2 has debug information, because only v2 was built with `-g`. Stripping removes `main` and the helper names but leaves the dynamic imports. `analysis.md` contains the full interpretation and the completed comparison table.

## Project 2: Linux System Call Forensics and Execution Analysis

**Scenario.** A backup utility sometimes fails to copy files; use `strace` to find out why.

**What I did.** `backup_tool.c` creates a backup directory, writes a numbered log, and uses `fork()` and `waitpid()` so a child process copies the file with `open`/`read`/`write`/`close`. I traced three experiments with `strace -f -tt`:

| Experiment | Result | What the trace shows |
|---|---|---|
| A: valid source | exit 0, 35 bytes copied | `openat` → `read` 35 → `write` 35 → `read` 0 (EOF) → `close`, then `wait4` status 0 |
| B: missing source | exit 2 | `openat("no_such_file.txt") = -1 ENOENT`, child `exit_group(12)`, parent `wait4` status 12 |
| C: unwritable directory (run as an unprivileged user) | exit 1 | `mkdir(...) = -1 EACCES`; no `clone`, so failure happened during setup |

The analysis explains each call, separates the program's own calls from the 29 loader/runtime lines in the trace, provides a table of 11 observed system calls, lists unexpected calls (`getpid` per log entry, `fstat`, `brk`, `getrandom`, no `fsync`), and proposes one improvement: validate first, then write to a temporary file, `fsync()` and `rename()`.

## Project 3: CPython Extension for High-Performance Data Analysis

**Scenario.** Speed up statistics code (mean/variance, min/max, outlier count, normalisation) by moving the hot loops into C while keeping a Python interface.

**What I did.** `pure_stats.py` is the baseline and `faststats.c` is the extension. Build with `python3 setup.py build_ext --inplace`.
- **Correctness:** `test_equivalence.py` runs 125 checks (small, large, edge-case and invalid inputs, memory behaviour), all passing, with exact equality between implementations. The testing caught and fixed a real mismatch: numeric strings such as `"2.5"` and huge integers were treated differently by the two versions.
- **Performance:** at 1,000, 100,000 and 1,000,000 values, with repeats and standard deviations. The C extension is 3.6× to 31× faster across the two full runs; the full four-operation pipeline is about 9-16× faster. Speedup does **not** grow with dataset size, and `normalize` gains least because creating a million result objects costs the same in both versions. The report discusses why, and shows that results depend on the baseline (built-in `min()`/`max()` are only about 4× slower than the extension).
- Two complete benchmark runs are included (`benchmark_results_A.*`, `_B.*`) to show run-to-run variation of about ±30% per cell.

## Project 4: Signal-Driven Linux Service Controller

**Scenario.** A lightweight monitoring service that administrators control with Unix signals.

**What I did.** `server_controller.c` prints a status line every five seconds and keeps a `NORMAL`/`MAINTENANCE` state. Handlers (installed with `sigaction()`) only increment `volatile sig_atomic_t` counters or set flags; all printing, state changes and cleanup happen in the main loop, so no unsafe function runs inside a handler.

| Signal | Behaviour (verified in `demo/`) |
|---|---|
| SIGUSR1 | Prints state and PID (`STATUS REPORT: state=NORMAL pid=...`) |
| SIGUSR2 | Toggles the mode; two back-to-back signals both took effect |
| SIGINT | Graceful shutdown: message, closes the log file, exit status 0 |
| SIGTERM | Emergency message, immediate exit with no cleanup, exit status 143 |

`run_demo.sh` regenerates the transcripts (two runs, since SIGINT and SIGTERM both end the process).

## Rubric checklist

| Requirement | Where |
|---|---|
| P1: both binaries analysed statically, headers/sections/segments/symbols/dependencies, comparison table, one-page report, limits of static analysis | `project1_elf/analysis.md`, `Forensic_Assessment_Report.pdf` |
| P2: program, success and failure traces (with child activity), ≥ 8 calls table, trace-based analysis | `project2_strace/` |
| P3: baseline, working extension with validation, equivalence evidence, multi-size benchmark with speedup, analysis | `project3_cpython/` |
| P4: safe handlers for all four signals, state, PID, shutdown behaviour, terminal evidence, one-page explanation | `project4_signals/` |

## Reproducing everything

```
cd project1_elf  && ./run_analysis.sh                          # static analysis (binaries are rebuilt separately; see outputs/00_compilation.txt)
cd project2_strace && gcc -Wall -Wextra -o backup_tool backup_tool.c && strace -f -o t.txt ./backup_tool src.txt out_dir
cd project3_cpython && python3 setup.py build_ext --inplace && python3 test_equivalence.py && python3 benchmark.py
cd project4_signals && ./run_demo.sh
```

## Honest notes

- Timings and PIDs differ on every machine and every run; the conclusions rely on patterns, not exact numbers.
- Experiment C in Project 2 was run as an unprivileged user (`setpriv`), because the default user is root and bypasses permission checks.
- Static analysis shows what a binary *can* do, not what it *does*; Project 1's report states this limitation explicitly.
