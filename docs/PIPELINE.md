# CI/CD Pipeline Flow

This document explains the complete flow of the Simulink/MATLAB CI/CD pipeline, from repository changes to test execution, result logging, and artifact generation. It is written for maintainers who may need to extend, debug, or troubleshoot the system.

---

## Overview

The pipeline:

1. Detects changes in whitelisted project folders
2. Runs a Python controller script
3. Launches MATLAB in headless mode
4. Executes the latest Simulink Test `.mldatx` file
5. Exports results to disk
6. Logs MATLAB output
7. Produces a structured CSV summary
8. Returns unique status codes so CI can act on failures or crashes

The following sections break down each stage.

---

### Sequence Diagram

![Sequence Diagram](Diagrams/SequenceDiagram.png)

## 1. Repository Structure (Expected)

Every project folder must contain:

```
<BASE_PATH>/
    <subsystem>/
        tests/       → *.mldatx files (Simulink tests)
        results/     → Exported test results (.mldatx snapshots)
        <other project files>
```

These folder names come from `config.json`, not hard-coded.

---

## 2. Change Detection (Python)

The Python controller:

1. Reads the config file
2. Filters project folders using:

   * `whitelist.project_folders`
   * `blacklist.project_folders`

3. Detects modified subsystems from Git/Gitea (added, modified, renamed files)

If a subsystem has no changes, it logs:

```
ENUM_UNCHANGED = -1
```

Otherwise, only subsystems in the whitelist will be pulled (updated locally in `BASE_PATH/<subsystem>`) and tested.

---

## 3. Launching MATLAB (Headless Mode)

Python invokes MATLAB using:

```
matlab -batch "run('helper.m')"
```

Environment variables passed into MATLAB:

* `TEST_SUBSYSTEM`
* `BASE_PATH`
* `CONFIG_FILE`

MATLAB uses these to locate:

* The project directory
* Test suite directory
* Results output directory

If MATLAB cannot start or the test script crashes, Python receives:

```
ENUM_MATLAB_ERROR = -99
```

---

## 4. MATLAB Test Execution

`helper.m` invokes the `automatedTestingScript.m` file, which:

1. Loads the latest `.mldatx` test file
2. Runs all contained test cases
3. Exports results to:

   ```
   BASE_PATH/<subsystem>/results/<timestamp>_results.mldatx
   ```
4. Cleans up Simulink models & test manager
5. Returns:

```
0 → PASS
1 → FAIL
```

These values map directly to Python’s `ENUM_PASSED` and `ENUM_FAILED`.

---

## 5. MATLAB Output Capture (Python)

Python captures:

* `stdout` line-by-line → logged as INFO
* `stderr` line-by-line → logged as ERROR

Logs are written to:

```
<LOG_FILE>/run_<YYYY-MM-DD>.log
```

This prevents log bloat in the CSV.

---

## 6. Result Tracking (Python)

After MATLAB exits, Python builds an entry for `results_list`:

```python
{
  "test": "<subsystem_name>",
  "status": "PASSED" | "FAILED" | "ERROR",
  "returncode": <int>,
  "message": "...",
}
```

---

## 7. CSV Output

After completing all subsystems, Python writes a summary CSV to:

```
RESULTS_FILE
```

Columns include:

```
test, status, returncode, message
```

This CSV is intended to be lightweight — logs are *not* stored here by design.

---

## 8. CI Interpretation

CI checks the final return code:

* If **any subsystem failed**, CI marks the pipeline as failed
* If MATLAB crashed (`-99`), CI marks a hard failure
* If all applicable subsystems passed, CI succeeds

Subsystems marked `UNCHANGED (-1)` do not affect CI status.

---

## 9. Failure + Crash Behavior

### Test failures

* MATLAB test completed normally
* Results exported
* CI shows “FAILED tests”, not a broken pipeline

### MATLAB crashes

* No results exported
* Error logged
* System returns `-99`
* CI flags the pipeline as a system failure

---

## 10. Troubleshooting Quick Reference

If there are more complex issues, refer to the true debugging docs.

### MATLAB hangs

* Usually stuck in the compile stage (might take time for the first run)
* Kill MATLAB, check for pop-up windows
* Check `logs`

### **Results not exporting**

* Check folder permissions
* Check existence of `<subsystem>/results`

### CI skipped subsystem

* Ensure directory name is in `whitelist.project_folders`

### JSON not loading

* Filename **must be `config.json`**
* Glob ignores anything else
