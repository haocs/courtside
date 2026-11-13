# Cross-project roadmap

Current implementations and the lab's proposed milestones must be tracked separately. The [lab milestones](https://github.com/haocs/tennis-algorithm-lab/blob/main/docs/MILESTONES.md) remain the detailed research plan; unchecked historical items there are not proof that no related serving UI exists today.

| Area | Working now | Next work / owner |
| --- | --- | --- |
| Review and serving metadata | Sessions, Videos/versions, Clips, Moments, coaching reviews and output-bound usefulness | Improve feedback/inspection ergonomics in tennis-web as requested; preserve current identities and history. |
| Processing execution | Lease-based process/dryrun/experiment tasks, checkpoints, timings, separate activation, and Worker Portal visibility across video, AI coach, learning plan, and clip-interval export tasks | **TODO**: standardize structured, secret- and path-redacted failure error logging/reporting from video, render, and AI workers into the Worker Portal; per-clip resume, verified Windows service/HDR support and measured bounded parallelism in worker/server. |
| Feedback for detector improvement | User-facing processing/cut feedback and serving annotations exist | Freeze a research export/acceptance contract across web, server and lab. UI feedback is not benchmark truth. |
| Benchmark | Scoring, labeling and split design documented | Build a reviewed private benchmark release, reproduce the exact baseline and implement scoring in the lab. |
| Feature reuse | Current detector/executor provides a baseline | Implement versioned observations/features and prediction-only experiments before rendering every candidate. |
| Candidate comparison | Candidate B0/B1/B2 ideas specified | Evaluate rule improvements on frozen development/test practice splits; comparison UI and experiment executor need implementation. |
| Retention | Originals/history/evidence are retained; no blanket automatic deletion | Design reference registry, pins, budgets and recovery-safe TTL enforcement; local dryrun cleanup is P2. |
| Training | Deferred | Revisit only when measurements justify cost and dataset quality. Heavy work belongs on compute workers, not the NAS. |

## Experiment principles

The lab specification defines duration precision/recall/F2, complete-action retention, boundary error and fragmentation gates. Implement the exact formulas, empty/unknown cases and scoring version from that specification rather than inventing a single scalar score here.

Keep adjacent recordings from one practice in the same benchmark split. Separate activity judgments from keep/remove editing preferences and preserve target-player/visibility context. Use held-out practices for comparison; tuning on them changes their role.

AI proposals and processor verdicts need provenance and human acceptance before entering ground truth. Retain small labels, recipes, reports and run history even when future retention removes regenerable media. Serving and protected rollback media do not receive an experimental TTL by default.

## Tracking work

Create system-level epics/decisions in Courtside with links to focused component issues. Put implementation details and tests in the owner repo. An issue should identify current versus planned behavior, source/time identities, acceptance evidence and rollout/retention consequences. Do not attach private benchmark media, profiles, locations or runtime configs to public issues.
