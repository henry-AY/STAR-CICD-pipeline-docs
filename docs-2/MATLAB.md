# MATLAB Test Runner Guide

The G-STAR Python execution layer acts as an isolated abstraction wrapper. It sits between the runner platform and MATLAB, managing process safety, real-time logging capture, and standard exit codes.

![Matlab Context](/Diagrams/MatlabContext.png)

### Context Injection via Environment Variables

The script reads inputs via command-line arguments and serializes them into a child environment dictionary (`os.environ.copy()`). This allows MATLAB to ingest the configuration fluidly without changing any hardcoded paths:
* `TEST_SUBSYSTEM`: Injects the name of the active subsystem under test (e.g., `ADAS`, `BCM`).
* `BASE_PATH`: Resolves the absolute path to the workspace root directory where codebases are actively checked out.
* `CONFIG_FILE`: Points to the shared JSON configuration schema containing simulation settings.

### Execution Engine & Batch Mode invocation

MATLAB is called using a non-interactive, headless execution sequence via `subprocess.Popen`:

```Python
cmd = ["matlab", "-batch", f"run('{helper_path}')"]
```

By utilizing `-batch`, MATLAB runs synchronously, routes all standard output directly to the stream buffer, disables desktop graphics rendering, and terminates automatically with a distinct exit code upon script completion.

### Project Isolation and Stream Redirection

The Python layer overrides default text buffering (`bufsize=1`) and captures combined stdout/stderr streams in real-time. This ensures that long-running MATLAB tests don't stall the runner console:

```Python
for line in proc.stdout:
    clean_line = line.strip()
    if clean_line:
        logging.info(f"\t\t[MATLAB / {SYSTEM}] {clean_line}")
```

### Watchdog Daemon & Process Group Kill

MATLAB spawning routines often spin up subsidiary background worker threads or toolboxes. Simply calling `proc.kill()` on the primary process ID can leave orphan MATLAB processes lingering in memory, quickly exhausting system RAM over subsequent runs.

G-STAR solves this by leveraging a tracking strategy:
1. `start_new_session=True` is declared inside `Popen`. This assigns a uniform Process Group ID (`PGID`) to MATLAB and any children it spawns.
2. A separate background thread (`threading.Timer`) handles the timeout monitoring.
3. If the timeout budget expires, `os.killpg()` issues a `signal.SIGKILL` across the entire group, wiping out lingering workers cleanly:

```Python
def kill_process(proc, timeout, timed_out_event):
    logging.error("[WATCHDOG TIMEOUT]: MATLAB test exceeded %s seconds", timeout)
    timed_out_event.set()
    try:
        # Destroys the process group entirely using the group ID
        os.killpg(os.getpgid(proc.pid), signal.SIGKILL)
    except Exception as e:
        logging.error("[WATCHDOG ERROR] Watchdog Failed to kill MATLAB subprocesses %s", e)
```
