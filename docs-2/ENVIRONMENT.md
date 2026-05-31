# Environment & Dependencies

The background daemon operates in an isolated environment that does not inherit user-level shell configurations (such as paths exported inside `~/.bashrc` or `~/.profile`). Consequently, system-wide environmental configurations must be explicitly mapped out.

### The Headless MATLAB Trap
When running tests manually, your terminal knows where the `matlab` executable resides. When systemd runs the runner, it initializes a highly restricted `$PATH` (typically limited to `/usr/bin:/bin`). This will cause the Python orchestrator to fail immediately with a `FileNotFoundError (Code 101)`.

To resolve this system-wide, a symbolic link must be exposed in the default system binary paths.

1. Identify the exact path to the active MATLAB binary via a regular user shell:

```bash
which matlab
# Example Output: /usr/local/MATLAB/R2023b/bin/matlab
```

2. Create a system-wide symlink targeting `/usr/local/bin/` (which is scanned by the background daemon):

```bash
sudo ln -s /usr/local/MATLAB/R2023b/bin/matlab /usr/local/bin/matlab
```

| Dependency | Required Version | Purpose |
|---------|:----:|---------:|
| Ubuntu Desktop OS | 22.02 LTS / 24.04 LTS | Core operating environment |
| Python 3 | $\ge$ 3.10.x | Interprets the `pipeline.py` and manages process sync |
| MATLAB | R2023B | Core Model-in-the-Loop (MIL) engine |
| Git CLI | $\ge$ 2.34.1 | Required by `actions/checkout@v3/4` to fetch repositories |
