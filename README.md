# Automated Simulink Test Harness Runner (Python + MATLAB CI/CD Integration)

> [!IMPORTANT]
This repository is currently being populated and will contain non-sensitive dummy data for the purpose of the goal. Additionally, this repository will include heavy documentation.

This repository provides a Python-MATLAB integration framework for automating Simulink Test Harness execution in a continuous integration (CI/CD) pipeline. It enables automatic subsystem validation, test execution, and report generation, triggered by a scheduled CRON job or updates to the Git repository.

## Overview

This project orchestrates MATLAB Simulink Test Manager runs via Python automation scripts. It enables a headless testing workflow, where changes to subsystem folders automatically trigger MATLAB test harness execution, and results are logged for analysis.

## Architecture

| Layer | Language | Role |
| --- | --- | --- |
| `CRON/Python` | Python | Schedules runs, manages configuration, launches MATLAB processes |
| `Gitea Repository` | n/a | Provides latest subsystem models |
| `MATLAB Scripts` | MATLAB | Executes Simulink tests and exports results |

## Sequence Diagram

![Sequence Diagram Image](Diagrams/SequenceDiagram.png)

