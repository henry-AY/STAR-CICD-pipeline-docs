# Workflow Setup (`yaml`)

The G-STAR workflow file employs a **Centralized Controller Pattern**. Rather than copying workflow configurations across every repository, a single controller repository (CICD) houses the execution logic. It pulls the targeted codebase dynamically based on instructions defined inside a centralized matrix.

### Core Architecture Guardrails
* Artifact Action Constraints: Gitea Actions platforms utilize an execution engine mimicking GitHub Enterprise Server (GHES). The modern `upload-artifact@v4` action utilizes an optimized API infrastructure incompatible with standard Gitea releases. You must strictly use `upload-artifact@v3` to prevent total workflow termination during artifact collection.
* Empty Matrix Panics: If the matrix configuration block contains empty parameters (`system: []`), the YAML parser fails immediately with an unrecoverable syntax error. The list must contain at least one valid entry, or the schedule trigger must be removed entirely to preserve stability.
* Timezone Alignments: The Gitea scheduler computes execution blocks solely using Coordinated Universal Time (UTC). Adjustments for local execution require offset calculations. For instance, configuring an execution targeting 3:00 AM PST requires mapping out 11:00 AM UTC (`0 11 * * *`).

### Production YAML Implementation
Below is the validated, stable workflow configuration file incorporating the time-filtered staging directory fix to avoid bloated uploads:

```YAML
name: CICD MIL System Tests
run-name: 3 AM Batch - ${{ github.actor }}

on:
  schedule:
    # 11:00 AM UTC maps out to exactly 3:00 AM PST (Pacific Standard Time)
    - cron: '0 11 * * *' 
  workflow_dispatch:

jobs:
  test-matrix:
    runs-on: self-hosted # Must be self-hosted
    strategy:
      fail-fast: false  # Ensure independent subsystems continue testing if an adjacent system fails
      matrix:
        # To scale testing, simply add additional verified repository names to this array
        system: [ARG, PCD, IOM]

    steps:
      # Step 1: Checkout the Centralized CI/CD Controller Infrastructure
      - name: Checkout Controller Tools
        uses: actions/checkout@v3
        with:
          repository: <Repo Owner Name>/CICD
          path: cicd_tools
          token: ${{ secrets.GITEA_TOKEN }}
          ref: main

      # Step 2: Dynamically Checkout the Target Subsystem from the Matrix
      - name: Checkout System (${{ matrix.system }})
        uses: actions/checkout@v3
        with:
          repository: <Repo Owner Name>/${{ matrix.system }}
          path: ${{ matrix.system }}
          token: ${{ secrets.GITEA_TOKEN }}
          ref: main

      # Step 3: Run the Python Orchestration Layer wrapping MATLAB
      - name: Execute MATLAB Test
        working-directory: cicd_tools
        run: |
          python3 script/pipeline.py \
            --system "${{ matrix.system }}" \
            --base-path "${{ github.workspace }}" \
            --script-path "${{ github.workspace }}/cicd_tools/script/helper.m" \
            --config "${{ github.workspace }}/cicd_tools/script/config.json" \
            --timeout 1200

      # Step 4: Isolate Freshly Modified Test Files (Staging Pattern)
      - name: Gather Recent Test Results
        if: always()
        run: |
          # Construct an isolated staging directory for the active run
          mkdir -p ${{ github.workspace }}/recent_results
          
          # Recursively identify .mldatx files altered within the last 15 minutes and stage them
          find ${{ github.workspace }}/${{ matrix.system }} -type f -name "*.mldatx" -mmin -15 -exec cp {} ${{ github.workspace }}/recent_results/ \;

      # Step 5: Securely Archive Test Metrics Using Gitea-Compatible API Layer
      - name: Upload MATLAB Test Results
        if: always()
        uses: actions/upload-artifact@v3  # Enforced v3 for structural compatibility with Gitea
        with:
          name: Test-Results-${{ matrix.system }}
          path: ${{ github.workspace }}/recent_results/
          retention-days: 7

      # Step 6: Clean Workspace Environment to Ensure Cleanliness
      - name: Cleanup Workspace
        if: always()
        run: |
          rm -rf ${{ github.workspace }}/*
          rm -rf ${{ github.workspace }}/.* || true
```
