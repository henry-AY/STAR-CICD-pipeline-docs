# Configuration File

Place a JSON file named `config.json` (**STRICT**) in your CICD scripts folder (e.g., `cicd-scripts/config.json`). The automation expects the .json filename to be `config.json` (unless otherwise passed in via CLI).

### Example of `config.json`:

```json
{
  "whitelist": {
    "project_subfolders": {
      "tests": "<folder name>",
      "results": "<folder name>"
    }
  }
}
```

* ##### `whitelist.project_subfolders.tests` 

  * Name of the subfolder inside each subsystem that contains `.mldatx` test suites (for example, `tests`).

* ##### `whitelist.project_subfolders.results`

  * Name of the subfolder where test MATLAB results will be exported (for example, `results`).
