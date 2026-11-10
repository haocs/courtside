# Cut review implementation status

Released 2026-11-10 as website v0.19.0, processing-server v0.2.0, and video-worker source revision `5ea93b52c923db53701dbb4d6dc308987cb7bd26`. This public document describes released contracts; installation-specific deployment evidence stays private.

| Component | Implementation | Validation |
| --- | --- | --- |
| Website | [tennis-web PR 86](https://github.com/haocs/tennis-web/pull/86): original-time kept/gap review, worker hints, selected-revision command preview, metadata-only rerun bridge, historical/candidate playback and A/B comparison | 133 backend tests, 50 helper tests, 120 Chromium/WebKit browser tests; changed integration flows rechecked after catalog pagination |
| Video worker | [tennis-processing PR 8](https://github.com/haocs/tennis-processing/pull/8): bounded diagnostics from existing observations, complete proxy/model/config provenance, verified snapshots, final cut-command enforcement and profile refresh | 92 tests; real synthetic rendering in the cross-component check |
| Processing server | [tennis-processing-server PR 8](https://github.com/haocs/tennis-processing-server/pull/8): snapshots, capability/storage gating, explicit staging, version catalogs, reference pins, fenced retention/recovery and legacy migration | 25 tests, including worker integration, concurrent migration, TTL/grace, symlink/identity guards and pin isolation |

The independent synthetic round trip covers website review → immutable server snapshot → worker render → candidate staging → website import. It also checks idempotent retry, an all-exclude metadata-only candidate, rejection of empty serving activation, and constrained dryrun without NAS media/result publication. Container checks run in the component release workflows; do not infer a deployment from CI success.

- [Website v0.19.0 release](https://github.com/haocs/tennis-web/releases/tag/v0.19.0), source `481a56d7277ef6e9fbef4cb4901d18946e284f04`.
- [Processing-server v0.2.0 source](https://github.com/haocs/tennis-processing-server/tree/v0.2.0), revision `1e2bf5f89d0f81d18dceb066f86d99addb19acd5`.
- [Video-worker released source](https://github.com/haocs/tennis-processing/commit/5ea93b52c923db53701dbb4d6dc308987cb7bd26).

## Contracts to use

- The [approved design](REVIEW-CUT-DESIGN.md) records intent and historical baselines. Component source and the contracts below define the exact implemented schema; proposal markers in the design are not evidence of a running endpoint.
- Website: [cut-review contract](https://github.com/haocs/tennis-web/blob/main/docs/CUT-REVIEW.md).
- Server: [API contract](https://github.com/haocs/tennis-processing-server/blob/main/docs/API.md).
- Worker: [session/output contract](https://github.com/haocs/tennis-processing/blob/main/SESSION-CONTRACT.md).

The video-version identity remains the existing source SHA plus frozen publication-token hash. Staging, activation and file relocation do not mint another identity. Server source IDs are separate registered-file identifiers. Candidate import never changes the website serving pointer.

Snapshots embed full selected review revisions and a server-verified base manifest. The inbound payload is limited to 128 KiB; the enriched immutable NAS artifact is limited to 256 KiB and referenced by storage ID, path, bytes and SHA-256. The worker validates it again and uses a verified local copy for retries. Version catalogs are paginated and size-bounded; exact-version lookups avoid downloading unrelated run histories.

Evidence does not execute itself. Good-cut, wrong-player, too-short and direction-only feedback remain advisory. The operator confirms exact keep/exclude/boundary commands separately. Hard excludes run after detector padding/merging. Overrides use the base clip's original bounds rather than trying to identify a newly detected clip by overlap.

## Configuration and rollout boundary

The website's optional `TENNIS_PROCESSING_API_URL` is a private, credential-free processing-server base URL. The separate `TENNIS_PROCESSING_ROOM_URL` controls its portal link. With no API URL, ordinary serving playback and cut review remain usable; rerun/pin integration is disabled. The website continues to mount only the intentionally published library read-only and its own separate mutable state.

After source review and a separately requested release, upgrade the server and independently installed worker before enabling the website bridge. Old `process-v2` workers cannot claim jobs requiring `cut-constraints-v1`. New workers refresh their authenticated capability profile without discarding their existing worker identity or local journal. Never install a production worker into this hub or an editor workspace.

Back up stopped server state before the schema migration. A rollback to older positional-insert SQL needs its matching stopped-state backup, not just an old image. Follow the [deployment runbook](deployment.md) for the coordinated rollout and verification.

Automatic cleanup is disabled by default. If explicitly enabled later, unpinned candidates have a default 14-day TTL followed by a full grace period starting at eligibility transition. Serving and superseded serving remain permanent by default. Per-tab and saved-comparison references own independent pins. Quarantine fences new pins/activation; recovery validates exact lexical paths and attempt identity and stops on conflicting paths or symlinks. Small manifests and review/comparison evidence remain after eligible full media is removed.

## Deliberately deferred

Removed-original gap proxies, automatic training labels, a broad research/dataset registry and model training executors remain separate work. The worker records local observations and cache provenance; the current processing path does not claim integrated cross-rerun observation-cache reuse. Worker-local media cleanup remains a follow-up. Preferences, heuristics and human feedback are not accepted benchmark truth automatically.
