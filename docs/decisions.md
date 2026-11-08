# Accepted decisions and implementation status

Reviewed against the [source snapshot](repositories.md#reviewed-source-snapshot), 2026-11-05. This is a system-level decision ledger, not a per-installation deployment report.

| Decision | Current status / implication |
| --- | --- |
| Keep originals and intentionally shared publications separate | Implemented storage boundary. Website does not need original access. |
| NAS hosts orchestration/viewing; compute runs on independently installed workers | Implemented Mac-first architecture. Windows service/HDR parity remains unverified work. |
| APIs carry state/metadata and artifact references, not media blobs | Implemented processing contract. Worker stages/uploads; server validates/activates. |
| Server-owned leases and durable task/attempt history | Implemented. Cached worker state is resumable context, not scheduling authority. |
| Separate coaching render/AI queues from video-processing queue | Implemented website/worker boundary. Do not merge these to reuse a worker name. |
| Separate coaching preparation from sharing-download profiles | The website's render role creates purpose-specific artifacts with explicit worker capabilities. Sharing downloads do not start AI, become coaching evidence or create serving Video versions. |
| Session / Video / version / Clip / Moment have distinct meanings | Implemented serving model. Compatibility API names can predate it. |
| Moment anchors survive reprocessing; cut/usefulness judgments are output-bound | Implemented model; keep any future remapping explicit and evidence-based. |
| Up/down are exclusive player reactions, outside the moment panel | Implemented website behavior. Down stays entity metadata; revisions are retained. |
| Complete website state is multi-database plus retained evidence | Implemented. A single website.sqlite3 copy is an incomplete application backup. |
| Shared LAN website state; no extra website accounts | Current behavior. A NAS login/reverse proxy does not automatically imply SSO or per-visitor filesystem filtering. |
| Preserve metadata and HDR semantics through processing | Implemented capture/property snapshot and validation, with documented dynamic-metadata limitations. Keep originals authoritative. |
| Improve detection through signals, temporal rules and measurement before training | Accepted direction. The lab's candidate/benchmark design is still largely proposed. |
| Cut review, diagnostic hints, confirmed reruns and version comparison | [Approved for staged implementation](REVIEW-CUT-DESIGN.md). Reviews are evidence; only separately confirmed commands affect candidate recuts. Component endpoints remain proposals until implemented. |
| Automatic experimental TTL/pins/cleanup, research registry and training executors | Retention/reference protection is included in the approved cut-review design, with automatic cleanup disabled by default. The broad research registry and training executors remain proposals. Do not delete artifacts based on design documents. |

## Resolving the lab storage proposal

The lab's earlier system proposal describes a future processing-server research registry for annotations, datasets, model manifests and artifact retention. It does not describe today's serving storage. Current Moments, reactions, entity notes and coaching history are owned by the website and its existing databases/adapters.

Keep these statements separate:

1. The processing server already owns processing tasks/attempts and activation.
2. The website currently owns serving metadata/Moments/coaching records; it exposes processing-feedback records through its current boundary.
3. A shared research registry, accepted dataset release flow and evaluation executor need explicit implementation and compatible migration contracts. They are not created by renaming a record or copying an old architecture diagram.

When a cross-project decision changes, update this page, the affected component contract and the proposal/status reference together. Avoid retaining contradictory instructions that would send an agent to a nonexistent endpoint or require a speculative migration.
