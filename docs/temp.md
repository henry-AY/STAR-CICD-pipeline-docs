## Architecture

| Layer | Language | Role |
| --- | --- | --- |
| `CRON/Python` | Python | Schedules runs, manages configuration, launches MATLAB processes |
| `Gitea Repository` | n/a | Provides latest subsystem models |
| `MATLAB Scripts` | MATLAB | Executes Simulink tests and exports results |

## Sequence Diagram

![Sequence Diagram Image](Diagrams/SequenceDiagram.png)
