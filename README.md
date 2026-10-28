# Traffic Analysis

A local vehicle detection and traffic-statistics portfolio project by **NoneTP**
and **Puntawit**. The project is currently in planning; implementation has not started.

Start with the **[central team plan](docs/superpowers/plans/2026-10-28-traffic-analysis.md)**
for shared milestones, task ownership, dependencies, and integration checkpoints.

| Developer | Focus | Detailed checklist |
| --- | --- | --- |
| NoneTP | Video/data pipeline, PostgreSQL, FastAPI, React, reliability, local setup | [Pipeline workstream](docs/superpowers/plans/2026-10-28-pipeline-workstream.md) |
| Puntawit | Dataset preparation, pretrained baseline, fine-tuning, tracking, ML evaluation | [ML workstream](docs/superpowers/plans/2026-10-28-ml-workstream.md) |
| Both | Shared contracts, early integration, count validation, final demo | Joint checkpoints in the central plan |

The first version processes preselected recordings, distinguishes cars, trucks,
motorcycles, and pickup trucks, stores crossing events, and updates a local
statistics dashboard every 3-5 seconds. Live CCTV input and fuel-type
classification are future goals.

See the [phase-one design](docs/superpowers/specs/2026-10-28-traffic-analysis-design.md)
for the agreed product behavior and scope.
