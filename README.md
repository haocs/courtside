# Courtside

The documentation home for a private-network tennis recording, review and improvement system.

Record on a phone or camera, retain the original, process on an independent compute worker, publish selected play to shared storage, and review it from mobile or desktop. The NAS hosts the website and processing orchestration; computation stays on worker machines.

This repository contains **documentation and agent instructions only**. Application code remains in the component repositories. It contains no deployment instance settings, recordings, personal metadata or access material.

## Start here

| Reader / task | Entry point |
| --- | --- |
| New human contributor | [Development setup](docs/development.md) |
| New coding agent | [AGENTS.md](AGENTS.md), then [agent onboarding](docs/agent-guide.md) |
| Coordinate implementation agents | [Agent collaboration](docs/agent-collaboration.md) |
| Understand the system | [Architecture](docs/architecture.md) and [domain model](docs/domain-model.md) |
| Choose where a change belongs | [Repository map](docs/repositories.md) |
| Release or deploy | [Deployment runbook](docs/deployment.md) |
| Diagnose a running installation | [Operations](docs/operations.md) |
| Understand current versus proposed behavior | [Decisions](docs/decisions.md) and [roadmap](docs/roadmap.md) |
| Publish docs or artifacts | [Privacy rules](docs/privacy.md) and [contribution workflow](CONTRIBUTING.md) |

## Component repositories

| Repository | Owns |
| --- | --- |
| [tennis-web](https://github.com/haocs/tennis-web) | Library, review/watch pages, serving metadata, Moments, coaching interface and separate render/AI workers |
| [tennis-processing](https://github.com/haocs/tennis-processing) | Installed video-processing worker, detector, rendering, stage checkpoints and local runtime |
| [tennis-processing-server](https://github.com/haocs/tennis-processing-server) | Scheduling, leases, tasks/attempts, input discovery, publication and processing control portal |
| [tennis-algorithm-lab](https://github.com/haocs/tennis-algorithm-lab) | Algorithm specifications, benchmark formulas, evaluation design and research milestones |

## Documentation ownership

Courtside owns cross-repository architecture, operating boundaries and onboarding. Exact APIs, dependency pins, database schemas and executable release workflows live with their components. Follow the repository map to those sources before implementing a change. This hub does not turn a proposed lab design into an implemented service.

Component releases remain independent. This repository has no application build, container image, worker service or automatic deployment.

Private installation settings stay outside all public checkouts. A new agent should discover the operator's chosen private configuration through the operator, not through committed machine-specific defaults.
