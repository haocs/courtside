# Deployment and release runbook

This is a generic operator/agent procedure. It contains no installation inventory or access settings. Exact Compose files and CI workflows remain in the component repositories. A documentation-only change to Courtside does not deploy any component.

## What runs where

| Component | Delivery |
| --- | --- |
| Website | Published `ghcr.io/haocs/tennis-web` container image on NAS |
| Processing server | Published `ghcr.io/haocs/tennis-processing-server` container image on NAS |
| Video worker | Independent installed source/runtime and OS service on compute machine |
| Coaching render/AI roles | Separately installed worker package from `tennis-web/worker`; independent role services |
| Algorithm lab / hub | Documentation; no running service/image to deploy |

Use the installation's existing private project settings. There is no universal NAS hostname, filesystem root, numeric user/group, port or deployment folder to copy from this repository.

## Authorization and preparation

Work within the user's actual request and existing authorization. A request to implement does not authorize a production release. A request to deploy the verified component authorizes its routine image update within the existing access/mount boundaries; do not add a redundant confirmation gate merely because a runbook exists. Expanding access, deleting persistent data or changing unrelated services is a separate action.

Identify the exact component/commit, active project, current image digest and persistent state mounts before mutation. Keep this installation record privately. Confirm no active work will be interrupted. For website updates, inspect coaching jobs; for server/worker changes, inspect assigned/leased processing tasks. Wait for work to finish or follow an explicitly authorized cooperative cancellation.

Read these owner documents first:

- [Website release workflow](https://github.com/haocs/tennis-web/blob/main/docs/RELEASE.md) and [Synology deployment](https://github.com/haocs/tennis-web/blob/main/docs/SYNOLOGY.md).
- [Server deployment](https://github.com/haocs/tennis-processing-server/blob/main/README.md) and [workflow](https://github.com/haocs/tennis-processing-server/blob/main/.github/workflows/release.yml).
- [Standalone video worker](https://github.com/haocs/tennis-processing/blob/main/README.md) and [coaching worker](https://github.com/haocs/tennis-web/blob/main/worker/README.md).

## Verify and publish the source

1. Confirm local changes match the intended PR, with private config/media excluded. Run relevant component tests and wait for CI. Review mobile and desktop for player/UI changes.
2. Merge the verified source within scope; verify the merged tree. Select a fresh semantic version without moving an existing release tag.
3. Use the actual component workflow trigger. In the reviewed website workflow, a **published GitHub release** triggers image publication; pushing a tag alone is not enough. The reviewed server workflow supports version-tag pushes as well as published releases. Re-read the workflows before relying on this difference.
4. Prefer GitHub API/connector tooling. If a required action is unavailable, use an already-authorized API client or the normal product UI for that narrow step. Do not mine browser sessions or move access material between tools. A locked desktop needs a normal manual unlock, not a workaround.
5. Wait for the release's verification and image-publishing jobs to finish. A merged PR, release page or version tag alone is not a deployable image.

## Verify the image

Check the immutable digest from the successful workflow/registry and the image's source-revision label against the intended commit. Verify the NAS platform is present; published images currently target Linux amd64 and arm64. Confirm the selected package can be pulled through the installation's intended access method. Public repository visibility alone does not establish public package visibility.

Record the exact new digest and the previous digest privately. Deploy the digest, not an unverified mutable `latest` tag. A source-commit label is evidence about the image; still require the matching successful checks and registry manifest/config verification.

## Back up complete state

Do this before changing an image that can migrate state. Keep originals and served files outside images/replaceable containers.

- For a consistent stopped-state website backup, stop the website container and copy its **entire private state directory/volume**, including all databases and any WAL/SHM companions. Check copied database integrity locally or on native storage before proceeding. Preserve pinned coaching evidence separately alongside the database bundle.
- Website state includes `website.sqlite3`, `annotations.sqlite3` and `coaching.sqlite3`. Coordinated reactions live in the metadata database. A `website.sqlite3`-only backup is **not** a complete favorite/Moment/coaching backup.
- Do not open a live SQLite database over an SMB client mount. Native SQLite backup APIs are appropriate for their documented single-database scope; a sequence of unrelated live file copies is not a consistent multi-database snapshot.
- The processing server has its own native state and documented backup tooling. Preserve its task/publication history separately from website state.

Keep backup paths and contents out of this public repository. Do not mutate the backup while inspecting the running installation.

## NAS image switch

Check the component's release notes for companion coaching/media-worker upgrades before switching the website. Website Clip-study HDR sharing and player learning plans require coaching worker 0.4.0 or newer: render advertises `share-export-v2` (plus legacy v1), and AI advertises `learning-plan-v1` alongside structured coaching support. Upgrade both installed role services from the same reviewed package; no new provider account or video-worker change is required. A new export profile can require a newer renderer even when the website state schema is unchanged. Preserve the worker's private configuration, local journals, artifact references and existing role services; use a reviewed source revision and the component's installation procedure. Keep the video-processing worker and its queue untouched. Capability-gated jobs should remain unclaimed by incompatible workers, not be forced through the old renderer. Verify the applicable role/capability after upgrade without starting a real provider call merely to test deployment.

Preserve project/service names, data volume/bind identity, read-only library/artifact mounts, environment settings, user/group settings and reverse-proxy configuration. Change only what the release requires. NAS runtime directories contain deployment settings/state/media, not a development checkout.

With an already configured private Compose project, the normal image update is:

```sh
docker compose config --quiet
docker compose pull
docker compose up -d --no-build
docker compose ps
```

Set the reviewed digest through that project's existing private image setting first. Do not print a resolved configuration into public logs or replace the installation with a generic example. A restart alone does not select a new image.

In Synology Container Manager, update the image-only project settings and use its normal deploy/update operation. Some DSM versions label this **Build** even for an `image:`-only project. Confirm the selected project and read its operation result. Never substitute **Clean**, Delete, volume removal, or a broad prune for a normal update. Enabling SSH or changing permissions is not a prerequisite when the existing UI/deployment route is sufficient.

Use separate addresses for browser navigation and service-to-service API traffic. The website's `TENNIS_PROCESSING_ROOM_URL` is the browser-facing Worker Portal address; `TENNIS_PROCESSING_API_URL` can use the processing service's Docker DNS name and internal port over an explicitly shared private network. Keep the network limited to the intended services, preserve existing published ports and filesystem mounts, and do not loosen a LAN reverse-proxy access policy to make container requests pass. When the services use separate Compose projects, declare the existing server network as external in the website project rather than creating a second network with a similar name.

After a routing update, verify `/api/v1/external-tasks` contains the existing coaching, learning-plan and export jobs and `/api/v1/workers` reports their heartbeats. An HTTP 403 in `portal_mirror` logs with an empty external task list is a failed status feed, not evidence that no coaching jobs exist. Keep the execution queues separate; repair their metadata mirror without rerunning provider calls.

Keep the entry point LAN-only. A reverse proxy does not automatically provide DSM visitor identity or per-user file ACL enforcement. Do not expose public ports or add a new identity system as part of an image update.

## Verify the running result

Check health/readiness, selected image/revision, logs and actual application behavior. For a website update, compare a pre-update snapshot with the live API: recording/Clip discovery, original metadata, courts/players, preferences, entities, Moments, coaching results/revisions, learning view and byte-range playback. Check relevant worker heartbeats and a real mobile/desktop view.

Use read-only live checks. Validate reaction/edit write behavior against isolated synthetic state before release; do not create test annotations or run provider calls on real recordings just for a deployment screenshot. Refresh an old browser page before diagnosing stale static assets.

For server changes, verify storage readiness, retained tasks/attempts, leases, watcher state and publication records without activating experimental results. Updating the website alone does not require a processing-worker restart.

Record source, release, workflow, digest, backup, verification and actual deployment result privately. The public hub may link a public release; it must not become a host inventory or receive raw operational evidence.

## Failure and rollback

If pull/startup fails, inspect the selected project's result and retain the old digest/state. Do not delete volumes or broaden permissions to force startup. A rollback may need both the prior image and a compatible full state snapshot; rolling back an image is not a database migration strategy.

Preserve writes made since the backup. Replacing live state can discard new feedback/jobs and needs a specific recovery decision, not an automatic overwrite. Distinguish published, image-ready, deployed and verified in the final report. If a tool/access limitation blocks the final step, report exactly what remains; do not claim deployment completed.
