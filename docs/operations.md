# Operations and troubleshooting

Use the installation's private service controls and configuration. Paths below are generic program/configuration references, never an inventory of an actual machine. Do not paste raw config, logs, profile data or screenshots into public issues.

## Video worker

The installed API worker has `workerctl` commands:

```text
workerctl status
workerctl logs
workerctl start
workerctl stop
workerctl restart
```

Use the wrapper from the actual standalone installation, not whichever similarly named checkout is in the shell. The Mac installer documents its installation location and update procedure. Restart normally refuses active work; do not add `--force` as a routine troubleshooting step.

`tennisctl` controls the older folder-watching pipeline. Running it alongside the API worker can duplicate scheduling. Establish which system owns the installation before starting anything.

When an upload does not process, check server watcher/scans, stable input discovery, queued task, worker accessibility/capabilities, mount identity, lease/heartbeat and stage progress in that order. A heartbeat proves contact, not useful processing progress. Reconnect storage through the normal OS mechanism; never embed access data in a path.

## Server and publication

Use the processing portal for task/attempt/stage inspection, watcher scan/enable state and worker pause/resume. Exact routes, desired-version semantics and retry behavior are in the [API](https://github.com/haocs/tennis-processing-server/blob/main/docs/API.md).

Pause affects future assignment; cancellation is cooperative. Retry/rerun is not the same as activating a result. Manual reruns should remain unpublished until reviewed. Never repair a stuck task by editing queue/database rows directly or treating a stale local lease as current ownership.

Record attempt/stage wall-clock timings and known source duration separately from upload-settling/queue delay. Do not fabricate historical timing for older results or add repeated totals across per-stage CSV rows.

## Website

For container checks, use the existing private project's equivalent of `docker compose ps` and service logs. Inspect the logs locally and sanitize any shared excerpt. Keep persistent state and the current image digest when restarting.

If a video is absent, verify a committed manifest exists and points to declared, readable published files. The website reads mounted filesystem paths; an SMB URI alone is not an HTTP media source. It must not mount private originals as a shortcut.

If playback contrast looks wrong, inspect original and rendered stream/color properties and the worker's export validation. Do not treat a browser screenshot alone as a color-analysis result. Preserve the original and use the processor's metadata evidence.

For failed edits, distinguish a stale version conflict from permissions, origin/host mismatch or storage readiness. The reverse proxy must preserve the expected Host (including the external-facing local port). Do not disable same-origin checks or replace a conflicting user edit blindly.

## Coaching roles

Render/AI workers are independent of the video worker. With an existing private config, their package offers role-specific `--status`, `--once` and foreground execution; see the [worker guide](https://github.com/haocs/tennis-web/blob/main/worker/README.md). Do not start a real AI role merely to diagnose UI layout.

An ambiguous provider outcome can be `needs_review`; inspect the retained local journal/result before an explicit retry. Exactly-once external provider execution is not guaranteed after a lost connection. Preserve raw responses and pinned evidence, not just temporary rendered excerpts.

Normal Mac Local Network permissions and the named native launcher can matter for background workers. Use the documented installer/toolchain route; do not reset privacy settings, bypass a denial or depend on an IDE's interpreter/session to keep a service alive.

## Storage and retention

Check the configured mount identity before accepting a newly created local directory as the NAS. The video worker should upload results once into staging; server activation renames on the same filesystem. Avoid unnecessary round trips or recopying whole recordings for metadata-only work.

Local dryrun scratch cleanup is planned, not an automatically safe command. Candidate TTL, independent reference pins and fenced garbage collection are implemented; automatic cleanup remains disabled by default. See the [released cut-review contracts](CUT-REVIEW-IMPLEMENTATION.md) before enabling retention cleanup. Preserve originals, current serving/rollback results, active/recovery task files, immutable labels/history and pinned coaching evidence. Any deletion needs a real ownership/reference/retention decision; age alone is insufficient.

See [deployment](deployment.md) for complete state backup/restore and [privacy](privacy.md) before sharing diagnostics.
