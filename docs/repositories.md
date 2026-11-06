# Repository map and sources

## Owners and entry points

| Repository | Change here for | Maintained entry points |
| --- | --- | --- |
| [tennis-web](https://github.com/haocs/tennis-web) | Library/session cards, Video/Clip review, player controls, Moments, reactions, serving metadata, coaching UI and render/AI adapters | [README](https://github.com/haocs/tennis-web/blob/main/README.md), [agent copy guidance](https://github.com/haocs/tennis-web/blob/main/AGENTS.md), [entity model](https://github.com/haocs/tennis-web/blob/main/docs/ENTITY-MODEL.md), [release](https://github.com/haocs/tennis-web/blob/main/docs/RELEASE.md), [Synology](https://github.com/haocs/tennis-web/blob/main/docs/SYNOLOGY.md), [coaching workers](https://github.com/haocs/tennis-web/blob/main/worker/README.md) |
| [tennis-processing](https://github.com/haocs/tennis-processing) | Video executor, detector/signals, encoding/metadata preservation, checkpoints, worker packaging and local service | [README](https://github.com/haocs/tennis-processing/blob/main/README.md), [publication contract](https://github.com/haocs/tennis-processing/blob/main/SESSION-CONTRACT.md), [dependency notices](https://github.com/haocs/tennis-processing/blob/main/DEPENDENCIES.txt), [follow-ups](https://github.com/haocs/tennis-processing/blob/main/PIPELINE-V2-TODO.txt) |
| [tennis-processing-server](https://github.com/haocs/tennis-processing-server) | Source discovery, scheduling, leases, tasks/attempts/events, worker status, publication/activation and processing portal | [README](https://github.com/haocs/tennis-processing-server/blob/main/README.md), [API](https://github.com/haocs/tennis-processing-server/blob/main/docs/API.md), [architecture](https://github.com/haocs/tennis-processing-server/blob/main/docs/ARCHITECTURE.txt), [workflow](https://github.com/haocs/tennis-processing-server/blob/main/.github/workflows/release.yml) |
| [tennis-algorithm-lab](https://github.com/haocs/tennis-algorithm-lab) | Evaluation formulas, annotation/signal/model contracts, benchmark design and experiment milestones | [README](https://github.com/haocs/tennis-algorithm-lab/blob/main/README.md), [system proposal](https://github.com/haocs/tennis-algorithm-lab/blob/main/docs/SYSTEM-DESIGN.md), [algorithm spec](https://github.com/haocs/tennis-algorithm-lab/blob/main/docs/ALGORITHM-SPEC.md), [milestones](https://github.com/haocs/tennis-algorithm-lab/blob/main/docs/MILESTONES.md) |

Related web contracts: [published sessions](https://github.com/haocs/tennis-web/blob/main/docs/SESSION-CONTRACT.md), [coaching evidence](https://github.com/haocs/tennis-web/blob/main/docs/COACHING-ANNOTATIONS.md), [moment panel](https://github.com/haocs/tennis-web/blob/main/docs/MOMENT-PANEL.md). Compatibility names containing `session` sometimes mean one recording/Video; use the domain model, not an old URL name, to choose the entity being edited.

## Reviewed source snapshot

Documentation review: **2026-11-05**. These are public source baselines, not a deployment inventory or an instruction to pin all future work to old code.

| Component | Reviewed main revision |
| --- | --- |
| tennis-web | [b3e2498427b2634f77f5e1ffa46041f77694d663](https://github.com/haocs/tennis-web/commit/b3e2498427b2634f77f5e1ffa46041f77694d663) |
| tennis-processing | [810690b29df1826f5f144fd8c0882f7938155281](https://github.com/haocs/tennis-processing/commit/810690b29df1826f5f144fd8c0882f7938155281) |
| tennis-processing-server | [3db34a89676a174d72cf68d8b455203785d6fa44](https://github.com/haocs/tennis-processing-server/commit/3db34a89676a174d72cf68d8b455203785d6fa44) |
| tennis-algorithm-lab | [7ee67d82d4a21d3ce14965278e92414f0c777d22](https://github.com/haocs/tennis-algorithm-lab/commit/7ee67d82d4a21d3ce14965278e92414f0c777d22) |

Re-read the current default branch, dependency pins and workflow before a new task. Resolve discrepancies against actual implementation and explicit accepted decisions. Preserve the status of proposals: the lab's research-registry ownership is not the current serving-Moment storage arrangement.
